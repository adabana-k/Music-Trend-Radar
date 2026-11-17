# 07 セキュリティ・運用

## IAM/Secret
Cloud Run API/取込/MusicBrainz補完を別サービスアカウント、最小IAM、Secret Manager。MVPは認証付き個人利用。APIキーや連携認証情報はWebクライアントへ送らない。MusicBrainzには連絡先を含む明示的User-Agentを付ける。

## 外部API利用
ListenBrainzは識別可能なUser-Agent（アプリ名/版/連絡先）を送り、毎回`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset-In`, `X-RateLimit-Reset`を確認する。`429`はresetヘッダー（`Reset-In`優先）または`Retry-After`まで待機し、5xx/接続失敗は上限付き指数backoff+jitter。固定の許容RPSを仮定しない。`range`ごとの成功payload hashが同じであれば保存/集計をno-opにする。API実装上の公開キャッシュは300秒だが、source update cadenceを示さないため、頻繁なpollingはしない。

MusicBrainzは**平均1リクエスト/秒以下（送信元IP単位）**、意味のあるUser-Agent、503時の遅延再試行、MBID単位のcacheを適用する。メタデータの変更を頻繁にpollしない。非商用API利用が無料であることは、supplementary dataのライセンス条件を無効にしない。

## 監視
sitewideの`from_ts,to_ts,last_updated`異常、スナップショット欠落、更新遅延、API429、DBロード重複、LLM根拠不整合、Cloud Run Jobs失敗、課金増加を監視。前回の成功値に切り戻す場合はUIに古さを表示。生の個人情報をログへ記録しない。

## コスト
Cloud Runの最大インスタンス数、Job最大実行時間、BigQuery課金バイト上限、GCSライフサイクル、Gemini入力/呼出上限。予算アラートは自動停止ではない。公開ダンプは大規模データなので計画・測定後に開始。

## プライバシー/削除
MVPでは個人のリスニング履歴を取得しない。将来ダンプを取り込む際は個人識別子を隔離し、公開集計を十分に粗くし、増分だけでは削除が反映されない問題に備え定期フル再同期を設計。外部提供・公開前に法的/倫理的レビュー。

## 本番変更
仕様・OpenAPI・SQL・テストを同じPRで更新。Cloud Build/CIの成功と手動承認後に本番デプロイ。環境変数の変更・アカウント権限はレビュー必須。
