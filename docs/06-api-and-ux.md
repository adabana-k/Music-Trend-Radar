# 06 API・UI仕様

API契約の正本は `contracts/openapi.yaml`。

## 1. 画面
- `/`: 最新**確定**週の人気・急上昇ランキング、期間とデータ鮮度、上位N打ち切り注記。
- `/tracks`: `popular | rising | new_entries`のタブ、検索、件数、ソート、観測不足理由。
- `/tracks/[id]`: 楽曲名・アーティスト・MBIDリンク（存在時のみ）、週間listen数の推移、前週比、ID確度、出典。
- `/artists`: アーティスト別人気/急上昇、曲の合計ではないことの説明。
- `/about-data`: ListenBrainz内の観測値であり世界市場全体の推計ではない、集計の更新頻度、ID解決、統計の上位N打ち切り、CC0表記。

## 2. REST
`GET /health`, `GET /api/v1/sources`, `GET /api/v1/tracks/trending?sort=popular|rising|new_entries`, `GET /api/v1/tracks/{id}`, `GET /api/v1/artists/trending`, `POST /api/v1/insights`。

**廃止:** `market=JP`、`window_days=7|14|28`、`chart_rank`など。MVPはsitewide確定週単位。後で追加するフィルタは取得可能な統計に基づいて個別設計。

## 3. 返却必須メタデータ
`source_id="listenbrainz"`, `period_start`, `period_end`, `source_updated_at`, `fetched_at`, `status`, `coverage_note`, `scope="listenbrainz_sitewide"`。各曲`listen_count`, `previous_listen_count|null`, `growth_rate|null`, `trend_score|null`, `identity_status`。

## 4. 画面状態
`loading`, `success`, `empty`, `partial`, `stale`, `error`を明確に区別。未確定・古い統計・前週比較不可の場合、人気ランキングは表示可能なら残し、伸び率ランキングのみ無効化。**未掲載曲を0回再生と表示しない。**

## 5. 文言
「再生回数」ではなく「ListenBrainzで記録された聴取回数」。新登場は「前週上位N外（聴取ゼロではありません）」。予測という文言は検証モデル実装後まで使わない。

## 6. AIインサイト
ユーザー操作でのみ実行。根拠値・対象週・出典・免責を併記。料金上限とAPI呼出回数制限。LLM出力に入力されていない指標があれば公開しない。
