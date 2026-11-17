# 05 トレンド分析・評価仕様

## 1. 指標の意味
**Popular** = ListenBrainz全体での直近の確定週の`listen_count`の多い順（MVPはsitewide上位N内）。
**Rising** = 連続した比較可能な確定2週のうち、聴取件数の増加が大きい曲。**Prediction** = 将来の順位入りをラベルとして学習し検証したモデル（MVPには含めない）。

## 2. 数式と欠損規則
今週`C_t`、前週`C_(t-1)`、先行指標は同じ同一性キーの比較。
- `delta = C_t - C_(t-1)`。
- `growth_rate = (C_t - C_(t-1)) / max(C_(t-1), 1)`。ゼロ基準や新登場は別扱い。前週未掲載を0としない。
- `log_growth = ln(1+C_t) - ln(1+C_(t-1))`。
- `trend_score = 100 * clamp(0,1, 0.65 * normalized_log_growth + 0.35 * normalized_log_current)`。2指標は当該週の比較対象内でwinsorize(5%,95%)→min-max正規化、分母0なら0とする。**学習確率ではないヒューリスティック**。正規化母集団とscore_versionを記録。
- `rank_within_snapshot`はlisten_count降順、同点はrecording_key昇順。

**計算許可条件:** 確定した同じ長さの連続した非重複の2期間がある、当該曲が**両期間の取得対象に含まれている**、IDが比較可能。満たさない場合は`growth_rate=NULL`,`trend_score=NULL`で`coverage_status`を`new_or_outside_top_n`, `incomplete`, `period_mismatch`, `identity_ambiguous`等で表示。現在上位入りしたばかりの曲は「新登場」として別セクションに表示（急上昇率を捏造しない）。

**確定と鮮度:** `range=week`はAPIへの要求値であって最終性の保証ではない。`from_ts`/`to_ts`が直近のUTC月曜境界の完了週と一致し、`last_updated >= to_ts`、同じ境界・`last_updated`・全ページhashが24時間以上離れた2回の成功取得で一致した場合だけアプリ内で確定とする（外部SLAではない）。`this_week`、更新時刻が期間終端より前の週、期待期間とずれた週、ページ不完全な取得は暫定またはstaleとして比較・「最新」扱いしない。APIが2026-11-17に返した`week`は2026-10-12〜19 UTCであり、遅延応答が現実に観測された。詳細は[データソース仕様](02-data-rights-and-sources.md)。

**Coverage:** `new_entries`は「現在取得した上位Nに存在し、直前の隣接確定週に取得した上位Nには存在しない」の意味に限定する。「新曲」「新規リスナー」「前週聴取ゼロ」を意味しない。`total_*_count`やページング完了も全ListenBrainz listen母集団の完全性を示さない。

## 3. サイト全体の成長補正（後日）
サイト全体のlisten総件数も増減する。必要なら`sitewide/listening-activity`などで、取得期間が厳密に一致する総listen数を確認したうえで、割合 `song_listens / all_site_listens` を比較する。現時点では正確に一致する分母が保証されないため**MVP必須スコアには入れない**。

## 4. アーティスト指標
sitewide artist APIの`listen_count`を採用。曲ランキング由来の部分合計で人気を代用しない。アーティストMBIDはサイト全体統計でnullの場合があるため名前キーは`unresolved`扱い。曖昧一致は手動検証。

## 5. 予測モデル導入の条件
履歴を十分に蓄積（例: 12週以上は単なる計画目安）、将来の正解ラベル例「次の4週でListenBrainz上位1000初登場」。正解が打ち切られているため、対象母集団は定義可能な曲だけに限定。評価は時間順ロールフォワード、Precision@K、Recall@K、ランダム/当週人気順/現行スコアと比較。未来のMusicBrainz更新情報、未来のスナップショット、集計後からしか入手できないデータを特徴量に使わない。

## 6. Gemini役割
スコア・順位はSQL/Pythonで再現可能に算出。Geminiは入力値に基づき、`headline`, `summary`, `evidence[]`, `caveats`のJSONを生成。個人listen履歴は入力しない。入力数値との機械的一致検証に失敗した説明は破棄。表現は「ListenBrainz内で伸びている可能性」までに留める。
