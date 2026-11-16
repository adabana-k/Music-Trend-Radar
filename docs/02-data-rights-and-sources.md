# 02 データソース・利用権・プライバシー

## 1. 利用データ
### ListenBrainz（主要）
- **MVP** `GET https://api.listenbrainz.org/1/stats/sitewide/recordings?range=week` と `/artists?range=week`。APIドキュメントの有効なrange値を使う。レスポンスには件数、`from_ts`, `to_ts`, `last_updated`等がある。**実際の期間境界が一致する場合のみ比較対象**。
- `week` と `this_week` は意味が異なる。現在週は未確定としてラベル表示し、確定週との同列比較に使わない。
- sitewideランキングは上位だけ（上位N打ち切り）であり、**全曲分布・ユニークユーザー数を取得したことにはならない**。該当エンドポイントのページ数/最大件数/データ構造は実装時に確認。
- **後続フェーズ** 公開フルダンプ + 増分ダンプ。フルは概ね月2回、増分は日次公開（配信・更新遅延は実際に確認）。増分は提出されたlisten基準であり再生日時とは異なる。削除反映はフルダンプ再同期が必要になる場合がある。
- 公開Listenデータ/テキストはCC0だが、提供APIの運用ルール、その他メタデータの権利、プライバシー、安全な公開方法を別途尊重する。

### MusicBrainz（補助）
- recording MBID/artist MBIDと名称・リリース関連メタデータの補助。聴取の人気指標として使用しない。
- APIは非商用利用は無料、商用はプラン/連絡を確認。正しいUser-Agent、**IP当たり平均1リクエスト/秒以内**、503時バックオフ。
- `recording_mbid`や`artist_mbids`が欠落する場合、タイトル文字列だけで自動的に統合しない。`mbid_mapping`は解決結果であり推定一致、`additional_info`は投稿者由来で未検証の場合がある。

### Fixture
- すべて架空のアーティスト、曲、識別子。外部サービスに接続しない。CIはfixtureで完走。

## 2. 利用ゲート
データソースごとに `source_id`, `license_reference`, `data_class`, `allow_storage`, `allow_derived_metrics`, `allow_public_display`, `allow_llm_input`, `allow_model_training`, `allow_export`, `retention_policy`, `attribution`, `reviewed_at`, `expires_at`, `privacy_reviewed` を保持。不明は拒否。**CC0ライセンスだから個人情報を無制限公開してよいわけではない。**

## 3. 個人リスナー保護
MVPはサイト全体集計APIのみを利用し、ユーザー名・個人listen履歴は収集しない。将来のダンプ版では識別子を集計処理の短期領域に限定、必要なら鍵付きHMACで内部仮名化、公開マートへユーザーIDを持ち込まない。小集団・単独個人へ遡れる出力を避ける。公開範囲・保存期間・削除再同期・地域法規を別途審査する。

## 4. Gemini向けデータ
許可された集計数値、曲名/アーティスト名、期間、出典のみを送信する。個人のリスニングイベント、ユーザー名、認証情報、音源を送らない。プロンプトと出力ログも保存範囲を最小化。

## 5. 参照・検査
https://listenbrainz.org/data/
https://listenbrainz.readthedocs.io/en/latest/users/api/statistics.html
https://listenbrainz.readthedocs.io/en/latest/users/listenbrainz-dumps.html
https://listenbrainz.readthedocs.io/en/latest/users/json.html
https://musicbrainz.org/doc/MusicBrainz_API
https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting

**リリースゲート:** ソースの更新規則・利用条件・個人情報への影響を再確認したら初めて本番スケジュールを有効化する。
