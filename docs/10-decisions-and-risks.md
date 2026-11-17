# 10 設計判断（ADR）・リスク台帳

## ADR
- **ADR-001 Accepted:** 主要データソースはListenBrainzのサイト全体集計。Spotify依存を削除。市場全体の代表性は主張しない。
- **ADR-002 Accepted:** MusicBrainzは楽曲メタデータとID補完。人気指標の取得先ではない。
- **ADR-003 Accepted:** MVPは**週次のサイト全体API統計**、公開ダンプは後続。sitewide上位Nにない楽曲は0と判定しない。
- **ADR-004 Accepted:** BigQueryが決定的スコアを計算し、Geminiは根拠付き文章生成のみ。
- **ADR-005 Accepted:** `JP市場`フィルタはMVPで削除。ListenBrainz全体が母集団であるため。
- **ADR-006 Accepted:** 未確定の今週値と確定した前週値の成長率比較は禁止。期間境界が一致しない限りトレンド算出を保留。
- **ADR-007 Accepted:** `from_ts/to_ts`を期間識別の正本とし、UTC月曜境界の直近完了週との一致、7日長、`last_updated >= to_ts`、24時間以上離れた2回の同一`last_updated`/payload hashをアプリ独自の確定条件とする。外部データの最終性保証ではない。
- **ADR-008 Accepted:** APIのページングは最大1000件/回・offsetで行うが、ランキングの取得可能上限より先は不明。取得上位範囲と明示し、未掲載を0扱いしない。`new_entries`は前週上位N外からの掲載を意味する。
- **ADR-009 Accepted:** ListenBrainzのrate-limit headers/User-Agentに従い、固定RPSを仮定しない。MusicBrainz補完はIP当たり平均1req/s以下・MBID照会優先。

## リスク
| ID | 内容 | 対応 |
|---|---|---|
| R-01 | ListenBrainzの利用者は音楽市場を代表しない | 母集団注記と相対的傾向の表現 |
| R-02 | sitewide統計はトップN打ち切り | 未掲載を0と扱わない、ダンプを別フェーズ |
| R-03 | APIの期間/データが遅延・改訂・不安定 | `from_ts/to_ts/last_updated/hash`検査、直近完了週とずれたらstale、安定確認後のみ確定 |
| R-04 | MBIDが無い/誤マッピング | 暫定IDと同定確度、曖昧自動マージ禁止 |
| R-05 | 週次データ量不足で予測できない | ルールベースに限定、十分な履歴でバックテスト |
| R-06 | 外部APIレート制限 | バックオフ、キャッシュ、MusicBrainz 1rps |
| R-07 | 公開ダンプの規模と個人履歴 | 別設計・コスト試算・個人識別子隔離 |
| R-08 | Geminiが根拠のない説明を出力 | 構造化出力、evidence照合 |
| R-09 | ライセンスやサービス運用方針変更 | ソースレジストリ、定期見直し |

## 公式資料では確定できず、実装/運用で監視する事項
1. sitewide統計の計算完了・改訂のSLA、`last_updated`の保証、`to_ts`境界の将来一貫性。
2. recordings期間別の計算上限と`total_recording_count`の厳密な意味、全ページ取得中に順位が固定されるか。
3. 2026-11-17実測ではAPIが約1か月前の週を返した。遅延の継続性・原因・復旧時期は不明。期待期間外ならstaleで保持し、未検証の古い週を最新扱いしない。
4. MusicBrainzを使う機能を非公開個人利用の範囲から公開/商用へ拡大する場合、最新のAPIプラン・フィールド別データライセンス・表示要件を再確認する。
