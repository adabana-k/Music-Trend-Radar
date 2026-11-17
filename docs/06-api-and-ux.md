# 06 API・UI仕様

API契約の正本は `contracts/openapi.yaml`。

## 1. 画面
- `/`: 最新**確定**週の人気・急上昇ランキング、期間とデータ鮮度、上位N打ち切り注記。
- `/tracks`: `popular | rising | new_entries`のタブ、検索、件数、ソート、観測不足理由。
- `/tracks/[id]`: 楽曲名・アーティスト・MBIDリンク（存在時のみ）、週間listen数の推移、前週比、ID確度、出典。
- `/artists`: アーティスト別人気/急上昇、曲の合計ではないことの説明。
- `/about-data`: ListenBrainz利用者内の観測値で世界市場全体の推計ではないこと、対象週/取得時刻/更新遅延、上位N打ち切りと未掲載の意味、ID解決、各データソースの利用条件を表示。

## 2. REST
`GET /health`, `GET /api/v1/sources`, `GET /api/v1/tracks/trending?sort=popular|rising|new_entries`, `GET /api/v1/tracks/{id}`, `GET /api/v1/artists/trending`, `POST /api/v1/insights`。

**廃止:** `market=JP`、`window_days=7|14|28`、`chart_rank`など。MVPはsitewide確定週単位。後で追加するフィルタは取得可能な統計に基づいて個別設計。

## 3. 応答に含める必須メタデータ
`source_id="listenbrainz"`, `scope="listenbrainz_sitewide"`, `source_range`, 元の`source_from_ts`/`source_to_ts`, 正規化した`period_start`/`period_end`, `source_updated_at`, `fetched_at`, `status`, `snapshot_revision`, `returned_count`, `result_limit`, `pagination_complete`, `coverage_note`を返す。`status=complete`はアプリの期間/安定性検査を通過した意味で、ListenBrainzの最終性保証でも全listen母集団の完全性でもない。上位Nのcoverageは独立して示す。各曲は`listen_count`, `previous_listen_count|null`, `delta|null`, `growth_rate|null`, `trend_score|null`, `identity_status`, `coverage_status`を返す。未掲載分の件数は推定しない。

## 4. 画面状態
`loading`, `success`, `empty`, `partial`, `stale`, `error`を明確に区別。APIの週が直近完了UTC週とずれる、`last_updated`が古い、安定確認が済まない場合は`stale`/暫定表示とし、最新確定週・伸び率ランキングに使わない。古い人気順位を表示する場合は対象期間と取得時刻を明示する。**未掲載曲を0回再生と表示しない。**

## 5. 文言
「再生回数」ではなく「ListenBrainzで記録された聴取回数」。新登場は「今回の取得上位Nに掲載、前週の取得上位Nには未掲載（前週聴取ゼロを意味しません）」と説明する。ランキングはListenBrainz利用者内・取得上位範囲の観測であり、世界市場推計ではない。予測という文言は検証モデル実装後まで使わない。

## 6. AIインサイト
ユーザー操作でのみ実行。根拠値・対象週・出典・免責を併記。料金上限とAPI呼出回数制限。LLM出力に入力されていない指標があれば公開しない。
