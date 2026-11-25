# Vue 3・Vuetify 4 のコンポーネント実装方針

## 共通ルール

- Vue の単一ファイルコンポーネントは Composition API と `<script setup lang="ts">` で書く。コンポーネント名とファイル名は PascalCase とする。
- 画面は `src/views/`、レイアウトは `src/layouts/`、複数業務で共用する UI は `src/components/` に置く。業務固有の部品は業務ごとにまとめる。PoC の在庫用部品は `src/components/inventory/` に置く。
- `VBtn`、`VTextField`、`VCard`、`VExpansionPanels` などは直接使う。複数画面に共通する見た目、ラベル、アクセシビリティ上の挙動が揃った箇所を部品化する。業務の意味を持つ表示は業務コンポーネントが担当する。
- 親から渡す表示データや状態は型付き `defineProps`、操作の通知は型付き `defineEmits` を使う。親が所有する入力値を子が編集する場合は `v-model` と `defineModel` を使う。入力の初期値は親で設定する。
- `MainLayout` が `VAppBar`・`VMain`・フッターの枠を配置する。アプリのルートに `VApp` と、その内側に `RouterView` を置く。業務のルート画面が `MainLayout` を描画し、レイアウトは画面ごとのタイトルと既定スロットの本文、名前付き `footer` スロットのボタンを受け取る。
- ルート画面が Store と Router に接続し、部品に表示データを渡す。表示・入力用の部品は props、model、イベントで動かす。画面間で復元する状態と検索結果は既存方針に沿って、それぞれ画面制御 Store とレスポンス Store が持つ。

[Vue の script setup](https://vuejs.org/api/sfc-script-setup.html) · [Vue の props](https://vuejs.org/guide/components/props.html) · [Vue のイベント](https://vuejs.org/guide/components/events)

## 在庫検索 PoC の部品と契約

| 部品 | 入力・受け取り | 通知・責務 |
| --- | --- | --- |
| `InventorySearchView` | Vue Router の在庫検索ルート、画面制御 Store、レスポンス Store | `MainLayout` の本文と `footer` を組む。画面制御 Store の開閉状態をフォームの model に接続し、入力途中のコードはローカルに持つ。検索イベントを受け、確定条件を画面制御 Store に保存し、レスポンス Store の取得 action を呼ぶ。現在の確定条件から結果を取り出す。 |
| `InventorySearchForm` | `v-model:product-code`、`v-model:expanded` | `VExpansionPanels` と商品コード入力を描画し、編集・開閉を model に返す。 |
| `InventorySearchFooter` | `productCode: string`、`pending: boolean`。ルート画面が `MainLayout` の `footer` に置く。 | `VBtn` を描画する。正規化後の商品コードが空、または `pending` が真なら無効。押すと `search({ productCode })` を1回通知する。 |
| `InventorySearchResults` | 確定条件に対応する取得状態、結果、エラーを props で受ける。 | 取得中・結果なし・エラーを表示し、結果をカードに渡す。状態ごとの表示は下の表に従う。 |
| `InventoryResultCard` | `productCode: string`、`productName: string`、`quantity: number` | `VCard` で表示する。現在の PoC は一覧表示までを扱う。 |

フォームの named model は `productCode: string` と `expanded: boolean` とする。PoC のカードは並び替えや個別操作を持たない表示専用なので、描画キーは現在の結果内の配列インデックスを使う。商品コードは同一検索内でも重複し得る。詳細画面を作る段階で API の一意 ID を決め、描画キーと詳細の識別に使う。画面制御 Store の開閉初期値は閉じた状態とし、商品コード入力の初期値は空文字とする。実機で開閉の操作回数を確かめ、初回から開くかを見直せる。入力を変更しても、検索操作までは前回の確定条件の結果を表示する。後続の詳細画面から一覧を作り直す際は、画面制御 Store に残した確定条件を入力欄の初期値に使う。

取得状態に応じて、次の表示を行う。現在の確定条件に対応する状態をルート画面から渡す。

| 状態 | 表示 |
| --- | --- |
| 検索前 | 検索条件を入力できる。結果なしの表示は検索成功後に行う。 |
| 取得中 | 取得中であることを表示し、同じ条件に前回成功したカードがあれば保持する。 |
| 成功 | 結果カードを表示する。成功本文の `items` が空なら「結果なし」と表示する。 |
| 失敗 | 取得エラーを表示する。同じ条件の前回カードがあれば同時に表示し、再検索できる。 |

## 確認方法

Storybook では検索パネルの開閉、空白入力時の検索ボタン、検索イベント、結果カードと空・エラー表示を固定値で確認する。検索イベントから取得、Zod 検証、Store 更新、再描画までの連携は Vitest＋MSW で確認する。

[UI コンポーネント間の連携](../06-integration-test/README.md#storybook-で確認する-ui-コンポーネント間の連携) · [Store の状態管理](ht-store.md)
