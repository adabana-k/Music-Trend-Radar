# 04 データ契約・BigQueryテーブル設計

## 1. ソース契約
MVP入力はListenBrainzのsitewide統計。録音: `payload.recordings[]`（`track_name`, `artist_name`, `listen_count`, 任意の`recording_mbid`, `artist_mbids`, `release_mbid`, `release_name`等）。アーティスト: `payload.artists[]`（`artist_name`, `listen_count`, 任意/nullの`artist_mbid`）。共通ソースメタデータは`count`, `offset`, `range`, `from_ts`, `to_ts`, `last_updated`。実測shape・ページング・権威資料は[データソース仕様](02-data-rights-and-sources.md)を参照。

APIは`count`を省略すると25件、最大1000件を返し、`offset`でページングする。アーティストは期間ごとに上位1000件、録音は少なくともall-timeで上位1000件の計算が公式資料に記載される。録音の期間別網羅上限は不明。全ページ取得できたことと全listen対象を網羅したことは別である。MVPでは最大取得順位までの部分ランキングと明示し、`total_*_count`を全母集団の網羅保証に用いない。

正規化する週はUTC月曜00:00境界の7日間。`range=week`を直近完了週として要求し、`range=this_week`は暫定表示に限る。取得期間の正はAPIの`from_ts`/`to_ts`であり、元値を整数UTC Unix秒のまま保存したうえ、内部期間を半開区間`[from_ts,to_ts)`として扱う。`source_updated_at`は`payload.last_updated`をUTCに変換し、`fetched_at`は実際の取得時刻とする。仕様資料に確定時刻・最終性の保証はない。期待する直近完了週と境界が一致し、`last_updated >= to_ts`で、少なくとも24時間離れた2回の取得で同一期間・`last_updated`・全ページpayload hashが一致したときだけ、アプリ独自の`complete`状態にする。APIの`last_updated`はソース更新時刻であり、単独で確定判定しない。

各ページは固定した`range`, `count`, `offset`で要求し、全ページで`from_ts`, `to_ts`, `last_updated`が一致することを確認する。途中で期間/更新値が変わったら取得全体を破棄する。5分の公開キャッシュはAPI実装上のキャッシュであり、統計の更新周期ではない。

ListenBrainz統計の`listen_count`はサイト内観測のlisten件数。Spotifyのストリーム数でも世界全体の聴取回数でもない。`listen_count`と`unique_listeners`は別指標、sitewideランキングから後者を推定しない。

## 2. SQLモデル（提案）
| テーブル | 主な列/粒度 | キー |
|---|---|---|
| `raw.lb_sitewide_snapshots` | `snapshot_id, entity_type, source_range, period_start, period_end, source_from_ts, source_to_ts, source_updated_at, fetched_at, page_count, returned_count, coverage_limit, payload_hash, raw_gcs_uri` | `snapshot_id` |
| `stage.lb_recording_stats` | `snapshot_id, recording_source_key, recording_mbid NULL, artist_name, track_name, listen_count, ordinal, identity_status` | `(snapshot_id, recording_source_key)` |
| `stage.lb_artist_stats` | `snapshot_id, artist_source_key, artist_mbid NULL, artist_name, listen_count, ordinal` | `(snapshot_id, artist_source_key)` |
| `stage.mb_recordings` | `recording_mbid, canonical_title, release_date NULL, enriched_at` | `recording_mbid` |
| `stage.mb_artists` | `artist_mbid, canonical_name, enriched_at` | `artist_mbid` |
| `mart.track_weekly` | `period_start, period_end, recording_key, listen_count, rank_within_snapshot, snapshot_id, revision, is_latest_revision, last_updated, is_complete, coverage_status, source_id` | `(snapshot_id, recording_key)` |
| `mart.artist_weekly` | 同上、`artist_key`で集計（sitewideソース値） | `(snapshot_id, artist_key)` |
| `mart.track_trends` | `as_of_period_end, recording_key, this_count, prev_count NULL, abs_change NULL, growth_rate NULL, trend_score NULL, coverage_status, score_version` | `(as_of_period_end, recording_key)` |
| `mart.artist_trends` | 同様 | `(as_of_period_end, artist_key)` |
| `ops.ingestion_runs` | `run_id, status, count, payload_hash, error, started_at, completed_at` | `run_id` |
| `ops.source_rights` | ソース単位の許諾・privacyフラグ | `source_id` |

日時はUTC TIMESTAMP。比較は元の`from_ts/to_ts`を保持しつつ、内部で`[start,end)`半開区間に正規化して行う。API資料は一般的な`to_ts`包含/排他規則を明示していないため、UTC月曜境界の実測値と7日長を検査し、期待境界と異なる場合は公開しない。週ラベルはUTCに固定、画面表示だけAsia/Tokyoに変換。取得日のDATEと対象期間は区別する。

## 3. 同一性解決
- 第一優先: 有効なrecording MBID / artist MBID。
- MBIDなし: `lb:recording:<hash(normalized_title,normalized_artist)>` の**暫定**キー。collisionsを想定し`identity_status=unresolved`、別録音同士の統合を保証しない。原文と正規化バージョンを保持。
- MusicBrainz補完時は`identity_alias`ブリッジで連携し、**過去集計に遡及適用する前に衝突確認**。自動あいまい一致は行わない。
- サイト集計楽曲をアーティスト別に合算すると重複・上位N打ち切りが生じるため、アーティストランキングはsitewideアーティスト統計を採用。

## 4. データ品質
DQ-01 件数非負・必須文字列空でない・取得時刻より先の期間を不正扱い。DQ-02 期間`start < end`、週は7日・UTC月曜境界、同一確定週の有効版は1つ。DQ-03 同じ期間・`last_updated`・payload hashの再取得はno-op。DQ-04 同じ週で`last_updated`またはhashが変われば不変raw版を追加し、martの対象週を原子的に置換（加算しない）。DQ-05 比較は`prev.end == curr.start`、期間長同一、両方確定の場合のみ。DQ-06 上位N外は欠損/coverage不足であり0としない。DQ-07 全ページの期間・更新値不一致はスナップショットを公開しない。DQ-08 更新遅延・順位同数・ID衝突・取得上限到達は監視する。

## 5. fixture
`fixtures/sitewide_recordings_week_A.json`, `...week_B.json`, `sitewide_artists_week_A.json`, `...week_B.json` に架空曲を格納。各fixtureは`payload.from_ts`, `to_ts`, `range`, `last_updated`, `count`, `offset`を含み、週AとBはUTC月曜境界で隣接する確定UTC週とする。ランキング上位N外、MBID未付与、同名別アーティスト、同じ版の再実行、`last_updated`変化、hashだけの変化、確定前/古い週、ページ途中の更新値不一致を用意。

## 6. パーティショニング
`raw.lb_sitewide_snapshots`は`DATE(fetched_at)`、`mart.*_weekly`は`DATE(period_start)`でパーティションし、ソースキーでクラスタリング。MERGEの更新範囲を対象週に絞る。原JSONはGCSにハッシュを含むキーで不変保存し、最新有効リビジョンをViewで公開。
