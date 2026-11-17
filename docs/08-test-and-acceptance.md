# 08 テスト計画・受入条件

## 1. 必須テスト
| ID | Given / When | Then |
|---|---|---|
| AT-01 | 同じsitewide週次snapshotを2回取込 | rawのバージョン管理とmartに論理重複がない |
| AT-02 | 二つの隣接する確定週で曲Aが10→20listen | `delta=10`, `growth_rate=1.0`、スコアは仕様式で再現 |
| AT-03 | 今週上位N内・先週上位N外の曲B | 前週件数はNULL、急上昇率NULL、`new_entries`タブへ |
| AT-04 | 期間の開始/終了が重複する2取得 | 比較は無効、`period_mismatch`、growth_rate=NULL |
| AT-05 | `this_week`など未確定の統計 | `provisional`、前週比較スコアなし |
| AT-06 | MusicBrainz MBID欠落 / 同名別曲 | 自動的な確定マージをしない、`unresolved`表示 |
| AT-07 | サイト全体アーティスト件数と曲上位N合計が違う | アーティストAPIの件数を優先 |
| AT-08 | Geminiに個人listen履歴を含むpayload | 送信前に拒否、API呼出0回 |
| AT-09 | Gemini出力が不正な数字を追加 | 出力破棄、通常ランキングは表示継続 |
| AT-10 | API 429 / 503 | 上限付き再試行、失敗時には既存マート維持 |
| AT-11 | 同じ期間/`last_updated`/payload hashを再取得 | 同じ版としてno-op、raw/martに論理重複なし |
| AT-12 | APIレスポンスに`market`/`chart_rank` | 契約違反として検出（新仕様から廃止） |
| AT-13 | fixture E2EでUIの人気一覧→曲詳細 | 曲・期間・listen回数・出典が一致 |
| AT-14 | UTC月曜境界の`range=week`と`this_week` | 返却`from_ts/to_ts`を正規化し、隣接7日でない限り比較しない |
| AT-15 | 同週2回取得が24時間以上離れて`last_updated`またはhash不一致 | 確定しない、再取得/古い確定版維持 |
| AT-16 | `range=week`の応答期間が直近完了UTC週より古い | staleとして表示、最新ランキング/成長率/確定に使用しない |
| AT-17 | `last_updated < to_ts` | provisionalとして確定・成長率に使わない |
| AT-18 | `count`/`offset`で上位ページを取得 | count既定25/最大1000の契約を守り、ページの連続性を検証。APIの計算上限外は未掲載を0扱いしない |
| AT-19 | ページの途中で期間/`last_updated`が異なる | 全取得を破棄し、部分ページの混合snapshotを公開しない |
| AT-20 | current top Nに掲載、前週top Nに不在 | `new_entries`はtop N新規掲載という意味に限り、前週count/growthはNULL |
| AT-21 | 同じ週の`last_updated`またはpayload hashが変わる | immutable revision作成、martを原子的に置換、旧版を二重計上しない |
| AT-22 | ListenBrainz APIが429/レートヘッダーを返す | reset情報/`Retry-After`まで待機し、固定RPSを前提としない |

## 2. レベル
`unit`（期間検証/正規化/成長率/revision）、`contract`（OpenAPI/ListenBrainzレスポンスshape/ページング/必須メタデータ）、`integration`（fixture→BigQuery→API）、`e2e`（Next.js）、`security`（認証/LLM入力/外部API負荷）、`data quality`（null/負数/ID衝突/重複/鮮度）。

## 3. Definition of Done
対象仕様ID、失敗テスト→実装→成功、OpenAPI合格、外部API規約遵守、SQLの再現性確認、ログから個人ID/Secretsが漏れないこと、実行方法/変更箇所をREADMEへ記載、PRにロールバック手順。
