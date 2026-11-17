# Music Trend Radar — ListenBrainz版・仕様駆動開発パッケージ

**Version:** 0.2.0  / **更新日:** 2026-11-17 / **ステータス:** 実装前仕様（MVP）

## 目的
ListenBrainzに公開された聴取データを主要な観測ソースに、MusicBrainzで楽曲識別を補完し、現在人気の曲・アーティストと、上昇傾向の強い候補を発見するWebアプリ。Google Cloud（Cloud Run, Cloud Run Jobs, Cloud Storage, BigQuery, Vertex AI）上で動作させる。

**重要:** ListenBrainzは全世界の音楽市場を代表するサンプルではない。「流行」の意味は **ListenBrainz利用者内での観測傾向**。MVPのスコアは予測確率でなく、説明可能な「勢い」指標。

## 変更点 v0.1 → v0.2 / 2026-11-17 API仕様更新
- 主データソースをSpotify/一般チャートから **ListenBrainz** に変更。Spotify API連携・Spotifyリンク表示要件を削除。
- **MusicBrainz** を楽曲/アーティストID補完に追加（人気データソースではない）。
- `chart_date/rank/chart_value/market=JP` 中心のモデルを廃止し、聴取回数 `listen_count`・期間 `[period_start, period_end)`・集計確定時刻で管理。
- API **sitewide統計取得** と **公開ダンプ取込** を明確に分ける。MVPはsitewide週間スナップショット、個別ユーザー・ユニークリスナー分析は将来のダンプ処理フェーズ。
- 指標、UI、OpenAPI、テスト、実装タスクを聴取回数ベースへ変更。
- ListenBrainzでユーザー許諾された公開listen dumpのCC0と、API運用規則・別メタデータの権利・個人情報の配慮を区別。
- 公式API資料と2026-11-17の公開API観測に基づき、週境界・ページング・確定/再取得・鮮度判定を明文化。APIが返す期間は最新とは限らず、最終性も保証されないため、遅延週は最新ランキングや比較対象として扱わない。

## 読む順番
1. `docs/01-product-requirements.md`（PRD）
2. `docs/02-data-rights-and-sources.md`（ソースと利用条件）
3. `docs/03-system-architecture.md`（GCP構成）
4. `docs/04-data-contract-and-model.md`（BigQuery）
5. `docs/05-analytics-and-evaluation.md`（算出ロジック）
6. `docs/06-api-and-ux.md`（UI/API）
7. `docs/07-security-operations.md`（運用）
8. `docs/08-test-and-acceptance.md`（受入テスト）
9. `docs/09-implementation-plan.md`（チケット）
10. `docs/10-decisions-and-risks.md`（ADR）
11. `contracts/openapi.yaml`（API契約）
12. `prompts/agent-start.md`（実装エージェントへの指示）

## 推奨実装順
fixtureでエンドツーエンド → ListenBrainz sitewide統計の週次取得 → MusicBrainz IDの補完 → 集計済みマートからのランキング/推移表示 → Vertex AIで数字を根拠にした説明文 → 十分な履歴が集まり、コスト/プライバシー検証後に公開ダンプからリスナー系特徴量。

**公開参照**
- https://listenbrainz.org/data/
- https://listenbrainz.readthedocs.io/en/latest/users/api/statistics.html
- https://listenbrainz.readthedocs.io/en/latest/users/listenbrainz-dumps.html
- https://listenbrainz.readthedocs.io/en/latest/users/json.html
- https://musicbrainz.org/doc/MusicBrainz_API
- https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting

ListenBrainz/MusicBrainzのAPI契約・確認日・保留事項は `docs/02-data-rights-and-sources.md`、取得と期間の正規化は `docs/04-data-contract-and-model.md` を参照。
