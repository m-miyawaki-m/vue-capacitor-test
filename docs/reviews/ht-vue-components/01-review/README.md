# Vue 3・Vuetify 4 のコンポーネント実装方針レビュー

Claude Code（Herdr `w3:p2`）が、基本設計、Store 設計、在庫 PoC 計画、Vitest・Storybook の例との整合を確認した。初回は must 1件、should 4件、nit 3件。M1 が解消すれば同意との回答。

| ID | 指摘と修正案 |
| --- | --- |
| M1 | 基本設計の図と詳細設計で `MainLayout` と `RouterView` の入れ子が逆。ルート画面が `MainLayout` を描画して本文とフッターを渡す構成に揃える。 |
| S1 | Storybook の開閉初期値と操作確認をフォームの named model に揃える。 |
| S2 | 固定応答には同じ商品コードが複数ある。カードの描画キーと、後続の詳細画面に必要な識別子を明記する。 |
| S3 | `MainLayout` を含むテストでは `VApp` を用意する。 |
| S4 | 開閉状態の所有者が画面制御 Store で、View がフォームに接続すると明記する。 |
| N1 | Storybook で空白のみの入力と検索時の正規化を確認する。 |
| N2 | 画面制御 Store の単体テスト観点を具体化する。 |
| N3 | 初回の開閉状態は HT の操作性を確認して見直せるようにする。 |

[対応記録](../02-response/README.md) · [コンポーネント実装方針](../../../04-detailed-design/ht-vue-components.md)
