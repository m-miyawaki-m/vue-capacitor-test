# 在庫管理 PoC アーキテクチャ図のレビュー

対象は draw.io の4ページ。Codex が PoC 計画・API 設計・Store 設計と照合し、Claude Code（Herdr `w3:p2`）が XML を独立に確認した。Claude の判定は **must 1件、should 3件、nit 4件**。M1 を修正すれば図の方針に同意するとの回答。draw.io 上での表示は未確認。

[アーキテクチャ図](../../../08-poc/ht-inventory-poc-architecture.drawio) · [PoC 計画](../../../08-poc/ht-inventory-poc.md) · [Store 設計](../../../04-detailed-design/ht-store.md) · [API 設計](../../../04-detailed-design/ht-api.md)

| ID | 重要度 | 箇所 | 問題と修正案 |
| --- | --- | --- | --- |
| M1 | must | 1ページ目 `action`、`lookup` | 画面制御 Store が Fetch クライアントを呼ぶように見える。画面からレスポンス Store の取得 action を呼び、手書きの API 処理を経て生成クライアントへつなぐ。画面制御 Store へは確定条件の保存を別の枝で示す。 |
| S1 | should | 1ページ目 `invalid`、`failure` | 失敗がレスポンス Store を通らず画面に出るように見える。Store が同じキーのエラーを記録し、前回成功データを保持することと、画面が両方を参照することを示す。 |
| S2 | should | 1ページ目 `response`、`validate` | MSW の HTTP 応答が検証へ直接入るように見える。応答は Fetch クライアントを経由して手書きの API 処理に戻り、そこで HTTP 判定と Zod 検証を行うと示す。URL 解決もこの処理の責務。 |
| S3 | should | 4ページ目 `detail`、`navigate` | 図は一覧の1件を参照する案だが、Store 設計は ID 別の詳細保持も想定する。一覧から再利用できる項目と、詳細専用データを取得する条件を区別する。現行の PoC スキーマには一意の項目 ID がないため、識別子は後続で定義すると明記する。戻り時の条件・スクロール位置の復元は画面制御 Store が担当する。 |
| N1 | nit | 1ページ目 `failure` | 失敗の種類に通信失敗を加える。 |
| N2 | nit | 3ページ目 AC1・AC2・AC4・AC5 | AC1 の HTTP メソッド、AC2 の空白入力と正規化、AC4 の空・エラー表示、AC5 のコミット後の再生成差分確認を追記する。 |
| N3 | nit | 2ページ目 生成物 | MSW ハンドラは Orval が生成し、ブラウザ用ワーカーは MSW CLI が生成することを示す。 |
| N4 | nit | 2ページ目 Faker と MSW | 生成ファイルの分かれ方は採用版の出力に従うと明記する。併せて Orval から Faker への生成矢印を追加し、Faker から MSW への利用関係と区別する。 |

自動生成ラベルと配布ビルドからのモック除外、PoC は一覧までで詳細は後続という範囲は計画と一致している。
