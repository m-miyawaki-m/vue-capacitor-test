# レビューと対応の記録

## レビュー記録表

| 対象 | レビュー | 指摘への対応 | 再レビュー・最終確認 | 残っていること |
| --- | --- | --- | --- | --- |
| Android HT アプリの設計書 | [レビュー済み](ht-design/01-review/ht-design-review.md) | [必須指摘は対応済み](ht-design/02-response/ht-design-review-response.md) | [Claude 同意・must 0件](ht-design/02-response/ht-design-review-response.md#claude-の最終確認) | Git の初期化方法と Android 実行環境は実装準備時に決める。 |
| 在庫管理 PoC の MSW 構成 | [再レビュー済み](ht-inventory-poc/03-rereview/ht-inventory-poc-msw-review.md) | [必須指摘は対応済み](ht-inventory-poc/02-response/README.md) | [Claude 同意・must 0件](../08-poc/ht-inventory-poc.md#方針確定) | 依存の導入、コード生成、テストの実行は着手後に確認する。 |
| PoC の Windows・既存 Chrome 実行方針 | [未レビュー](../08-poc/ht-inventory-poc.md#windows-での起動とテスト) | — | — | MSW 構成のレビュー後に追加した方針。 |
| 在庫管理 PoC のアーキテクチャ図 | [初回レビュー済み](ht-poc-architecture/01-review/README.md) | [対応済み](ht-poc-architecture/02-response/README.md) | [Claude 同意・must 0件](ht-poc-architecture/03-rereview/README.md) | draw.io 上での表示は未確認。 |
| 資料の8フェーズ分割 | [レビュー済み・must 0件](ht-document-split/01-review/ht-document-split-review.md) | **未対応** | — | should 5件、nit 4件。 |
| 公開前の資料チェック | [レビュー済み](ht-publication/01-review/README.md) | [必須指摘は対応済み](ht-poc-architecture/02-response/README.md) | [must 0件](ht-poc-architecture/03-rereview/README.md) | GitHub 認証と公開作業は未完了。 |
| 自動生成・モック切り替え・Storybook の手順 | [未レビュー](../README.md#技術手順インデックス) | — | — | 生成関数と設定例の照合、Windows での実行は未確認。 |
| Vitest の Store 単体・Storybook の UI 連携の追記 | [初回レビュー済み](ht-test-methods/01-review/README.md) | [対応済み](ht-test-methods/02-response/README.md) | [Claude 同意・must 0件](ht-test-methods/03-rereview/README.md) | コード例の実行は未確認。 |
| Vue 3・Vuetify 4 のコンポーネント実装方針 | [初回レビュー済み](ht-vue-components/01-review/README.md) | [対応済み](ht-vue-components/02-response/README.md) | [Claude 同意・must 0件](ht-vue-components/03-rereview/README.md) | PoC 実装とテスト例の実行は未確認。 |
| テスト方針と業務シナリオ | [初回レビュー済み](ht-test-strategy/01-review/README.md) | [対応済み](ht-test-strategy/02-response/README.md) | [Claude 同意・must 0件](ht-test-strategy/03-rereview/README.md) | テストコードの作成・実行は未実施。 |

「—」は指摘対応や再レビューの記録がない項目を示す。

## 設計書

1. [初回レビュー](ht-design/01-review/ht-design-review.md)
2. [初回指摘への対応](ht-design/02-response/ht-design-review-response.md)
3. [再レビュー](ht-design/03-rereview/ht-design-rereview.md)
4. [再レビューへの対応](ht-design/02-response/ht-design-review-response.md#再レビューへの対応)
5. [Claude の最終確認](ht-design/02-response/ht-design-review-response.md#claude-の最終確認)。必須指摘は0件。

## 在庫管理 PoC

1. [初回レビュー](ht-inventory-poc/01-review/ht-inventory-poc-review.md)
2. [初回指摘への対応と方針変更](ht-inventory-poc/02-response/README.md)
3. [MSW 構成の再レビュー](ht-inventory-poc/03-rereview/ht-inventory-poc-msw-review.md)
4. [再レビュー対応後の最終確認](../08-poc/ht-inventory-poc.md#方針確定)。MSW 構成の必須指摘は0件。

## PoC アーキテクチャ図

1. [初回レビュー](ht-poc-architecture/01-review/README.md)。must 1件、should 3件、nit 4件。
2. [指摘への対応](ht-poc-architecture/02-response/README.md)
3. [再レビュー](ht-poc-architecture/03-rereview/README.md)。必須指摘は0件。

## 資料分割

1. [8フェーズ構成のレビュー](ht-document-split/01-review/ht-document-split-review.md)。必須指摘は0件。should・nit の対応は未実施。

[資料一覧へ戻る](../README.md)
