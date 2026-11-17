# 02 データソース・利用権・プライバシー

## 1. 利用データ
### ListenBrainz（主要）
- **MVP** `GET https://api.listenbrainz.org/1/stats/sitewide/recordings?range=week` と `/artists?range=week`。`week` は公式UIで「Last Week」、`this_week` は「This Week」。`this_week` は未確定候補として扱い、確定週との比較に使わない。
- APIの対象期間はラベルから推定せず、応答のUTC Unix秒 `from_ts` / `to_ts` を保存する。内部契約は `[from_ts, to_ts)` に正規化し、元の秒値も保持する。週次比較は、両方が7日間で境界が隣接する場合だけ許可する。公式資料は統計の確定時刻、更新SLA、将来変更されないことを保証していない。
- API実装資料では `count` は既定25・最大1000、`offset` で順位をページングする。アーティストは各期間上位1000まで、録音は少なくともall-time上位1000までと明記される一方、録音の期間別上限は文書上明確でない。したがってMVPは取得した上位範囲のランキングとしてのみ表示し、全分布・未掲載の聴取ゼロ・ユニークユーザー数を意味させない。`total_*_count` も全listen母集団の網羅性を保証する値とは解釈しない。
- **確定判定（アプリ独自ポリシー、ListenBrainzの保証ではない）:** `range=week`の応答が直近の完了UTC週（UTC月曜00:00境界）と一致し、期間が正確に7日、`last_updated >= to_ts`、必須メタデータが揃うものを候補とする。少なくとも24時間離れた2回の成功取得で期間、`last_updated`、正規化payload hashがすべて一致して初めて確定とする。1回目は暫定、`this_week`は常に暫定。直近週以外は`stale`、`last_updated < to_ts`/不一致は`provisional`、ページ欠落は`partial`とし、新しい確定週・成長率に使わない。24時間は保守的なMVP再確認間隔であり、外部SLAではない。
- **2026-11-17の実測（単発観測であり保証ではない）:** `range=week` は `from_ts=1791763200`〜`to_ts=1792368000`（2026-10-12〜10-19 UTC、`last_updated=1792818907` / 2026-10-24 05:15:07 UTC）、`this_week` は `1792368000`〜`1792972800`（10-19〜10-26 UTC、`last_updated=1792810776` / 2026-10-24 02:59:36 UTC）を返した。11-17時点の直近完了週ではなく、3週間以上遅れた期間だった。よって範囲名やHTTP 200だけで新鮮・確定と判定できない。呼出しで既定25件、`count=100`、`offset=25`、`range=week/this_week`が応答し、録音・アーティストとも`from_ts`,`to_ts`,`last_updated`,`range`,`count`,`offset`と集計配列を確認した。
- レスポンス契約: 録音は `payload.recordings[]` の `track_name`, `artist_name`, `listen_count` と任意の `recording_mbid`, `artist_mbids`, `release_mbid`, `release_name` 等。アーティストは `payload.artists[]` の `artist_name`, `listen_count`, nullable/任意の `artist_mbid`。双方で`count`, `offset`, `range`, `from_ts`, `to_ts`, `last_updated`を保存し、ページを集約した取得件数・ページ数・hash・取得時刻も内部記録する。未知フィールドは捨てずraw保存する。
- APIの応答レート制限は固定値を仮定しない。`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset-In`（優先）, `X-RateLimit-Reset`を毎回確認し、`429`ではreset情報/`Retry-After`を尊重する。全APIリクエストに連絡可能な`User-Agent`とバージョンを設定する。API実装の公開キャッシュは300秒だが、これは統計の再計算頻度や確定時刻ではない。
- **後続フェーズ** 公開フルダンプ + 増分ダンプ。フルは概ね月2回、増分は日次公開（配信・更新遅延は実際に確認）。増分は提出されたlisten基準であり再生日時とは異なる。削除反映はフルダンプ再同期が必要になる場合がある。
- 公式同意文面では、許諾されたlistening historyを公開ダンプでCC0公開する。これは提供APIの運用ルール、artist/release metadata等の別権利、プライバシー、安全な公開方法を一括で許諾するものではない。

### MusicBrainz（補助）
- recording MBID/artist MBIDと名称・リリース関連メタデータの補助。聴取の人気指標として使用しない。
- APIは非商用利用無料、商用利用はプラン/連絡を確認する。連絡先を含む正しいUser-Agentを設定し、送信元IP当たり平均1リクエスト/秒を超えず、503は間隔を空けて再試行する。MBID lookupを優先し、名称検索結果は候補であって確定同定ではない。
- `recording_mbid`や`artist_mbids`が欠落する場合、タイトル文字列だけで自動統合しない。複数artist MBID、MBID欠落、MBIDのない重複名称は別状態として保持し、MusicBrainzへの照会だけでListenBrainz側の同一性を確定しない。
- MusicBrainzデータベースのcore dataはCC0、supplementary dataはCC BY-NC-SA 3.0。API利用料の説明とデータベースの再利用ライセンスは別の条件である。MVPは非公開利用を前提とし、外部公開・商用化・追加フィールド利用の前に対象フィールドのライセンスと表示要件を再確認する。

### Fixture
- すべて架空のアーティスト、曲、識別子。外部サービスに接続しない。CIはfixtureで完走。

## 2. 利用ゲート
データソースごとに `source_id`, `license_reference`, `data_class`, `allow_storage`, `allow_derived_metrics`, `allow_public_display`, `allow_llm_input`, `allow_model_training`, `allow_export`, `retention_policy`, `attribution`, `reviewed_at`, `expires_at`, `privacy_reviewed` を保持。不明は拒否。**CC0ライセンスだから個人情報を無制限公開してよいわけではない。**

## 3. 個人リスナー保護
MVPはサイト全体集計APIのみを利用し、ユーザー名・個人listen履歴は収集しない。将来のダンプ版では識別子を集計処理の短期領域に限定、必要なら鍵付きHMACで内部仮名化、公開マートへユーザーIDを持ち込まない。小集団・単独個人へ遡れる出力を避ける。公開範囲・保存期間・削除再同期・地域法規を別途審査する。

## 4. Gemini向けデータ
許可された集計数値、曲名/アーティスト名、期間、出典のみを送信する。個人のリスニングイベント、ユーザー名、認証情報、音源を送らない。プロンプトと出力ログも保存範囲を最小化。

## 5. 公式一次資料（確認日: 2026-11-17）

| 対象 | 公式資料 |
|---|---|
| ListenBrainz sitewide統計、フィールド、ページング、トップ件数 | [Statistics API](https://listenbrainz.readthedocs.io/en/latest/users/api/statistics.html) |
| ListenBrainz rate-limit headers、429、User-Agent | [API documentation: rate limiting](https://listenbrainz.readthedocs.io/en/latest/users/api/index.html#rate-limiting), [User-agent](https://listenbrainz.readthedocs.io/en/latest/users/api/index.html#user-agent) |
| ListenBrainz timestamps are UTC Unix seconds | [Core API documentation: timestamps](https://listenbrainz.readthedocs.io/en/latest/users/api/core.html#timestamps) |
| 実装のキャッシュ、top-1000注記、ページ上限/既定値 | [sitewide stats endpoints](https://github.com/MetaBrainz/listenbrainz-server/blob/6d5d061f4dc5c93ca097a17d9698e51898ea0e34/listenbrainz/webserver/views/stats_api.py#L1130-L1154), [recordings endpoint](https://github.com/MetaBrainz/listenbrainz-server/blob/6d5d061f4dc5c93ca097a17d9698e51898ea0e34/listenbrainz/webserver/views/stats_api.py#L1322-L1355), [API page limits](https://github.com/MetaBrainz/listenbrainz-server/blob/6d5d061f4dc5c93ca097a17d9698e51898ea0e34/listenbrainz/webserver/views/api_tools.py#L36-L61) |
| `week`/`this_week`のUI上の名称 | [ListenBrainz stats range labels](https://github.com/MetaBrainz/listenbrainz-server/blob/6d5d061f4dc5c93ca097a17d9698e51898ea0e34/frontend/js/src/user/stats/utils.tsx#L160-L174) |
| Listenデータダンプと利用許諾 | [ListenBrainz data](https://listenbrainz.org/data/), [ListenBrainz dumps documentation](https://listenbrainz.readthedocs.io/en/latest/users/listenbrainz-dumps.html), [ListenBrainz consent text](https://github.com/MetaBrainz/listenbrainz-server/blob/6d5d061f4dc5c93ca097a17d9698e51898ea0e34/frontend/js/src/gdpr/GDPR.tsx#L43-L64) |
| MusicBrainz API利用、商用条件、MBID lookup/search | [MusicBrainz API](https://musicbrainz.org/doc/MusicBrainz_API) |
| MusicBrainz 1 rps、User-Agent、503 | [MusicBrainz API rate limiting](https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting) |
| MusicBrainz core/supplementary data license | [MusicBrainz data license](https://musicbrainz.org/doc/About/Data_License) |

ListenBrainzのCC0はユーザーが許諾したlistenデータに関するものとして扱い、API負荷、個人情報、artist/release metadataやcover art等の別権利を一括で許諾するものとはみなさない。sitewide集計から個人単位の情報を再構成しない。

## 6. 未確定事項と安全な扱い
- ListenBrainzがsitewide統計をいつ計算し終えるか、`last_updated`の更新SLA、最終性、期間別recordings全順位の上限・`total_recording_count`の厳密な意味は公式資料だけでは確定できない。最新の`from_ts/to_ts`と`last_updated`を監視し、安定確認前は暫定、期待期間とずれたらstaleとして扱う。
- ページ取得中の順位変動を原子的に固定するAPI機能は確認できない。全ページを同一`range`/`from_ts`/`to_ts`/`last_updated`で取得し、いずれかがずれたらその回を破棄して再取得する。上限より多くある場合も未取得を0に変換しない。
- 公開APIの観測は2026-11-17の単発取得。応答遅延・仕様変更を継続監視し、再現しない挙動を外部保証として記載しない。

## 7. 参照・検査
https://listenbrainz.org/data/
https://listenbrainz.readthedocs.io/en/latest/users/api/statistics.html
https://listenbrainz.readthedocs.io/en/latest/users/listenbrainz-dumps.html
https://listenbrainz.readthedocs.io/en/latest/users/json.html
https://musicbrainz.org/doc/MusicBrainz_API
https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting

**リリースゲート:** ソースの更新規則・利用条件・個人情報への影響を再確認したら初めて本番スケジュールを有効化する。
