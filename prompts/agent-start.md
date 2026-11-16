# コーディングエージェント初回指示（ListenBrainz版）

あなたはMusic Trend Radarの実装担当です。`README.md`、`docs/01`〜`docs/10`、`contracts/openapi.yaml`を順に読み、**仕様駆動/TDD**で実装してください。

## 絶対ルール
1. 主データソースはListenBrainz sitewide recordings/artists API。Spotify APIやSpotify固有の`popularity`等を実装しない。
2. MVPは週次のListenBrainz全体統計。`market=JP`, `chart_rank`, `spotify_url`など旧仕様の残骸は使わない。
3. `from_ts,to_ts,last_updated`を保存し、**比較可能な確定した連続週のみ**前週比を計算する。トップN外の曲は0件ではなく観測欠損。
4. MusicBrainzは補助ID解決のみ。欠損MBIDを架空の確定IDに変えず、レート制限(1rps)とUser-Agentを守る。
5. 個人listen履歴、ユーザー名をMVPで収集しない。将来ダンプ利用には別のプライバシー審査を必要とする。
6. BigQuery SQLの指標は決定的に算出し、Geminiは集計済み値の説明だけに使用。LLM出力の数値は必ず照合する。
7. 実APIアクセスができない環境では外部ネットワーク接続を試みず、fixtureと契約テストで開発を進める。実データ取得の成功を偽装しない。
8. Secretをコード・Git・ログへ含めない。外部有料リソース作成、本番デプロイは明示的承認まで実行しない。

## 最初のPR
Phase 0のドキュメント・契約確認 + Phase 1のfixture Provider → BigQuery DDL → FastAPI `GET /api/v1/tracks/trending` → Next.jsランキング画面までを実装。初めにAT-01, AT-02, AT-03, AT-04, AT-12, AT-14の失敗テストを作成。READMEにローカル実行手順を書く。

## 完了報告
実装ファイル、実行したテスト結果、未確定のAPI仕様、コスト/プライバシーの注意点、次のPR計画を簡潔に報告する。
