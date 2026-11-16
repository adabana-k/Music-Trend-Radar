# 04 データ契約・BigQueryテーブル設計

## 1. ソース契約
MVP入力はListenBrainzのsitewide統計。楽曲: `payload.recordings[]`（`track_name`, `artist_name`, `listen_count`, 任意の`recording_mbid`, `artist_mbids`, `release_mbid`等）。アーティスト: `payload.artists[]`（名称、件数、任意のMBID）。**レスポンス実体をfixtureで記録してプロパティ名をAPI仕様に合わせて検証する**。共通メタデータ：`from_ts`, `to_ts`, `last_updated`, `range`, `fetched_at`。

ListenBrainz統計の`listen_count`はサイト内観測のlisten件数。Spotifyのストリーム数でも世界全体の聴取回数でもない。`listen_count`と`unique_listeners`は別指標、sitewideランキングから後者を推定しない。

## 2. SQLモデル（提案）
| テーブル | 主な列/粒度 | キー |
|---|---|---|
| `raw.lb_sitewide_snapshots` | `snapshot_id, entity_type, source_range, period_start, period_end, source_updated_at, fetched_at, raw_gcs_uri, checksum` | `snapshot_id` |
| `stage.lb_recording_stats` | `snapshot_id, recording_source_key, recording_mbid NULL, artist_name, track_name, listen_count, ordinal, identity_status` | `(snapshot_id, recording_source_key)` |
| `stage.lb_artist_stats` | `snapshot_id, artist_source_key, artist_mbid NULL, artist_name, listen_count, ordinal` | `(snapshot_id, artist_source_key)` |
| `stage.mb_recordings` | `recording_mbid, canonical_title, release_date NULL, enriched_at` | `recording_mbid` |
| `stage.mb_artists` | `artist_mbid, canonical_name, enriched_at` | `artist_mbid` |
| `mart.track_weekly` | `period_start, period_end, recording_key, listen_count, rank_within_snapshot, last_updated, is_complete, source_id` | `(period_start, period_end, recording_key)` |
| `mart.artist_weekly` | 同上、`artist_key`で集計（sitewideソース値） | `(period_start, period_end, artist_key)` |
| `mart.track_trends` | `as_of_period_end, recording_key, this_count, prev_count NULL, abs_change NULL, growth_rate NULL, trend_score NULL, coverage_status, score_version` | `(as_of_period_end, recording_key)` |
| `mart.artist_trends` | 同様 | `(as_of_period_end, artist_key)` |
| `ops.ingestion_runs` | `run_id, status, count, payload_hash, error, started_at, completed_at` | `run_id` |
| `ops.source_rights` | ソース単位の許諾・privacyフラグ | `source_id` |

日時はUTC TIMESTAMP。比較は `[start,end)` 半開区間を正規化して行う。`to_ts`の包含/排他は実レスポンス仕様に沿って正規化し、境界テストで検証。週ラベルはUTCに固定、画面表示だけAsia/Tokyoに変換。取得日のDATEと対象期間は区別する。

## 3. 同一性解決
- 第一優先: 有効なrecording MBID / artist MBID。
- MBIDなし: `lb:recording:<hash(normalized_title,normalized_artist)>` の**暫定**キー。collisionsを想定し`identity_status=unresolved`、別録音同士の統合を保証しない。原文と正規化バージョンを保持。
- MusicBrainz補完時は`identity_alias`ブリッジで連携し、**過去集計に遡及適用する前に衝突確認**。自動あいまい一致は行わない。
- サイト集計楽曲をアーティスト別に合算すると重複・上位N打ち切りが生じるため、アーティストランキングはsitewideアーティスト統計を採用。

## 4. データ品質
DQ-01 件数非負・必須文字列空でない・記録日未来不可。DQ-02 期間`start < end`、同一確定週の一意性。DQ-03 同じスナップショット再取込で重複0。DQ-04 前後期間比較は`prev.end == curr.start`かつ期間長同一・確定であること。DQ-05 上位N打ち切りで未掲載は0としない。DQ-06 更新遅延・期間重複・順位同数・ID衝突は監視。DQ-07 `last_updated`変化の同週再取得はリビジョン扱い。

## 5. fixture
`fixtures/sitewide_recordings_week_A.json`, `...week_B.json`, `sitewide_artists_week_A.json`, `...week_B.json` に架空曲を格納。各fixtureは`payload.from_ts`, `to_ts`, `range`, `last_updated`を含み、週AとBは隣接する確定UTC週とする。ランキング上位Nから外れた曲、MBID未付与、同名別アーティスト、再実行、確定前の週も用意。

## 6. パーティショニング
`raw.lb_sitewide_snapshots`は`DATE(fetched_at)`、`mart.*_weekly`は`DATE(period_start)`でパーティションし、ソースキーでクラスタリング。MERGEの更新範囲を対象週に絞る。原JSONはGCSにハッシュを含むキーで不変保存し、最新有効リビジョンをViewで公開。
