# 08 テスト計画・受入条件

## 1. 必須テスト
| ID | Given / When | Then |
|---|---|---|
| AT-01 | 同じsitewide週次snapshotを2回取込 | rawのバージョン管理とmartに論理重複がない |
| AT-02 | 二つの隣接する確定週で曲Aが10→20listen | `delta=10`, `growth_rate=1.0`、スコアは仕様式で再現 |
| AT-03 | 今週上位N内・先週上位N外の曲B | 前週件数はNULL、急上昇率NULL、`new_entries`タブへ |
| AT-04 | 期間の開始/終了が重複する2取得 | 比較は無効、`period_mismatch`、growth_rate=NULL |
| AT-05 | `this_week`など未確定の統計 | `incomplete`、前週比較スコアなし |
| AT-06 | MusicBrainz MBID欠落 / 同名別曲 | 自動的な確定マージをしない、`unresolved`表示 |
| AT-07 | サイト全体アーティスト件数と曲上位N合計が違う | アーティストAPIの件数を優先 |
| AT-08 | Geminiに個人listen履歴を含むpayload | 送信前に拒否、API呼出0回 |
| AT-09 | Gemini出力が不正な数字を追加 | 出力破棄、通常ランキングは表示継続 |
| AT-10 | API 429 / 503 | 上限付き再試行、失敗時には既存マート維持 |
| AT-11 | 確定済み週の`last_updated`が更新され再取得 | revision作成、最新有効版に切り替え、旧版を二重計上しない |
| AT-12 | サイト全体集計がトップNのみ | 未掲載=0としない、coverage noteを返す |
| AT-13 | APIレスポンスに`market`/`chart_rank` | 契約違反として検出（新仕様から廃止） |
| AT-14 | fixture E2EでUIの人気一覧→曲詳細 | 曲・期間・listen回数・出典が一致 |

## 2. レベル
`unit`（期間検証/正規化/成長率）、`contract`（OpenAPI/ListenBrainzレスポンスshape）、`integration`（fixture→BigQuery→API）、`e2e`（Next.js）、`security`（認証/LLM入力）、`data quality`（null/負数/ID衝突/重複）。

## 3. Definition of Done
対象仕様ID、失敗テスト→実装→成功、OpenAPI合格、外部API規約遵守、SQLの再現性確認、ログから個人ID/Secretsが漏れないこと、実行方法/変更箇所をREADMEへ記載、PRにロールバック手順。
