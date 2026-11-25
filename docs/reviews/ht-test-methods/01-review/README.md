# Vitest と Storybook のテスト手順レビュー

Claude Code（Herdr `w3:p2`）が、Store の単体テスト、Storybook の UI コンポーネントテスト、資料インデックスの追記を確認した。判定は同意、must 0件、should 4件、nit 4件。S1・S2 は着手前の反映を推奨。コード例の実行は未確認。

| ID | 重要度 | 対象 | 指摘と対応案 |
| --- | --- | --- | --- |
| S1 | should | 単体テストの Store 例・API 設計 | API 境界が例外を投げるか結果型を返すかを定め、失敗時のモックと揃える。 |
| S2 | should | 単体テストの観点表 | API 境界の 200・500・契約不一致と、検索キーの正規化の観点を加える。正規化の担当も明記する。 |
| S3 | should | 結合テストの Storybook と Pinia | テスト用 Pinia を Vue アプリに登録する方法と、`createSpy: fn` の指定を示す。採用する `@pinia/testing` と Pinia の互換性は導入時に確認する。 |
| S4 | should | 結合テストの `InventorySearchPanel` | フッターの枠を MainLayout が持つ設計と整合するよう、画面側がフォームとフッターボタンを渡す構成を明記する。 |
| N1 | nit | Store の単体テスト例 | `beforeEach` で `setActivePinia(createPinia())` を用いて、各テストの状態を分離する書き方も検討する。 |
| N2 | nit | 再取得失敗の期待結果 | `fetchedAt` が前回の成功時のままか確認する。 |
| N3 | nit | Storybook のアサーション | `toHaveBeenCalledExactlyOnceWith` が採用版で使えるか確認する。呼び出し回数と引数の検証に分けてもよい。 |
| N4 | nit | 単体・結合の分類 | 部品単独の表示と、複数部品を組み合わせた UI コンポーネントテストの違いを明記する。 |

Store 単体で API 関数を差し替える範囲と、MSW を使って実際の Store・Orval・Zod をつなぐ範囲は整合している。追加したインデックスのリンク先も確認した。

[単体テストの資料](../../../05-unit-test/README.md) · [結合テストの資料](../../../06-integration-test/README.md) · [技術手順インデックス](../../../README.md#技術手順インデックス)
