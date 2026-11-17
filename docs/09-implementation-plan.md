# 09 実装計画・バックログ

## Phase 0 — 契約とfixture
- P0-01 完了（仕様確定）: 公式資料と2026-11-17の実測でshape・ページング・期間メタデータを記録。確定時刻/最終性は公式保証がなく、保守的なアプリ内確認規則を採用。
- P0-02 完了（仕様確定）: ListenBrainzのrate-limit headers/User-Agent、MusicBrainz非商用条件・1rps・ライセンス区分、MBID欠損時の安全策を記録。
- P0-03 架空の連続2週のサイト全体統計fixtureとエラーケースを作成する。
- P0-04 取得は`count`/`offset`のページング、同一取得内の期間/更新値検証、同じhashのno-op、期間改訂時の不変revision+原子的置換を満たす。

## Phase 1 — 垂直スライス
- P1-01 `apps/web` Next.js + `services/api` FastAPI の基本構築。
- P1-02 `ListenBrainzSitewideProvider`と`FixtureSitewideProvider`を同じインタフェースで実装。`week`が期待する直近完了UTC週と異なる場合はstaleにする。
- P1-03 JSONスナップショットをGCS保存し、BigQuery raw/stageへ冪等ロード。
- P1-04 BigQueryの`mart.track_weekly`, `mart.artist_weekly` と人気ランキングSQL。
- P1-05 `/tracks`・`/artists`画面、出典と期間を表示。
- P1-06 Cloud Run API/Job、Cloud Scheduler、CI環境を構築。

## Phase 2 — 急上昇・詳細
- P2-01 隣接した確定2週の比較可能性チェック。
- P2-02 growth_rate、trend_score、新登場カテゴリ、coverage_status。
- P2-03 曲詳細・アーティスト詳細、MusicBrainz MBID補完（レート制限/キャッシュ）。
- P2-04 更新遅延、上位N打ち切り、未同定等の表示。

## Phase 3 — AIによる説明
- P3-01 Vertex AI Geminiに送れる集計値のみを抽出するadapter。
- P3-02 evidence付きJSONと厳格な値照合、LLM停止時フォールバック。

## Phase 4 — 発展（別仕様/ADR承認必須）
- P4-01 フル/増分公開ダンプのボリューム・利用権・プライバシー・費用検証。
- P4-02 リスナー数・新規リスナー数等を導入する場合は処理計画と削除同期を実装。
- P4-03 過去データによる未来の上位入り予測・バックテスト。
- P4-04 Google Trendsなど外部ソースはAPI利用承認後に別Provider追加。

## 推奨リポジトリ
```text
music-trend-radar/
  apps/web/
  services/api/
  services/ingest/providers/{base,fixture,listenbrainz,musicbrainz}.py
  analytics/sql/
  fixtures/
  contracts/openapi.yaml
  infra/terraform/
  tests/{unit,contract,integration,e2e}/
  docs/
  .github/workflows/
```

**実装方針:** 1PR 1ストーリー。`spec_id`, `acceptance_tests`, `rights_privacy`, `rollback`をPR本文に記録。資格情報なしでfixtureのE2Eを先に通す。
