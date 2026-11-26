# 結合テスト

| 段階 | 業務の確認 | フレームワーク連携の確認 | 主な手段 |
| --- | --- | --- | --- |
| 結合 | 検索ボタンで取得が始まり、成功・結果なし・失敗に応じた表示になる。読み取り結果が呼び出し元の入力欄に反映される。 | 画面、実際の Pinia Store、Orval 生成クライアント、Zod をつなぎ、HTTP 応答だけを差し替える。画面制御と取得結果がそれぞれの Store に反映されることを確認する。読み取りの結合ではプラグインの応答を差し替える。 | Vitest と Vue Test Utils。実ブラウザが必要な場合は Playwright |

在庫 PoC では `msw/node` で応答を切り替え、送信クエリ、画面の表示、検証失敗時のデータ保持を確認する。

## 初回の疎通画面

- Vitest と Vue Test Utils で、画面とアプリ側の呼び出し処理をつなぎ、プラグイン応答を差し替える。疎通確認ボタンで入力値が渡り、応答またはエラーが表示されることを確認する。
- 撮影ボタンで固定の `webPath` を受け取り、画像表示が更新されることを確認する。キャンセル（`OS-PLUG-CAMR-0006`）では前回画像を保持し、権限拒否（`OS-PLUG-CAMR-0003`）とその他の撮影エラーでは対応する案内を表示する。Web での撮影操作では実機で確認する機能である旨を表示する。
- 初期ルート `/` から疎通確認画面が開くことを確認する。

[在庫 PoC の受入基準](../08-poc/ht-inventory-poc.md#受入基準とテスト)

## MSW によるテストデータの切り替え

開発時は生成 MSW ハンドラが Faker の応答を返す。自動テストでは件数や表示内容を再現できる固定値を使う。

| 確認したい挙動 | 応答の作り方 |
| --- | --- |
| 正常に2件表示する | 生成 Faker ファクトリの出力に固定2件の `items` を重ね、生成 MSW ハンドラの引数として渡す |
| 検索結果が空になる | 生成 MSW ハンドラに `{ items: [] }` を渡す |
| HTTP 500 になる | 同じパスの手書き MSW ハンドラでステータス500を返す |
| Zod 検証で失敗する | 手書きハンドラで、必須項目を省いた本文や `quantity` を文字列にした本文を HTTP 200 で返す |

### 固定の正常応答とケース別の上書き

以下は設定例。生成関数名と import パスは採用バージョンの出力を確認して合わせる。

```ts
import { http, HttpResponse } from 'msw'
import { setupServer } from 'msw/node'
import { afterAll, afterEach, beforeAll } from 'vitest'
import { getListInventoryResponseMock } from './generated/inventory.faker'
import { getListInventoryMockHandler } from './generated/inventory.msw'

const fixedResponse = {
  ...getListInventoryResponseMock(),
  items: [
    { productCode: 'A001', productName: '商品A', quantity: 12 },
    { productCode: 'A001', productName: '商品A', quantity: 3 },
  ],
}

const url = 'http://inventory.test/api/inventory'
const server = setupServer(getListInventoryMockHandler(fixedResponse))

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

// 空のケースのテスト内で使用
server.use(getListInventoryMockHandler({ items: [] }))

// HTTP 500 のケースのテスト内で使用
server.use(http.get(url, () => new HttpResponse(null, { status: 500 })))

// 契約不一致のケースのテスト内で使用
server.use(http.get(url, () =>
  HttpResponse.json({
    items: [{ productCode: 'A001', productName: '商品A', quantity: '12' }],
  }),
))
```

- 上書きは対応するテスト内で1つずつ登録する。再取得失敗時のテストは、正常応答で初回表示してから500または不正本文に切り替え、同じ条件で検索を実行する。エラーが出て、前回のカードが残ることを確認する。
- Node の Fetch が使う絶対 URL と手書きハンドラの URL を揃える。クエリは `new URL(request.url).searchParams.get('productCode')` で読み取り、入力を正規化した値と比較する。
- ブラウザの開発モードは `setupWorker`、Vitest の結合テストは `setupServer` を使う。Playwright の固定モードでは、同じ固定正常応答を登録したブラウザ用ワーカーを起動する。
- 固定値で応答本文全体が決まらないスキーマに拡張した場合は、期待値に使う項目も固定する。

[Orval の Faker ガイド](https://orval.dev/docs/guides/faker/) · [Orval の MSW ガイド](https://orval.dev/docs/guides/msw/) · [MSW の resetHandlers](https://mswjs.io/api/setup-server/reset-handlers)

### Vitest で再取得失敗を確認する例

モックのセットアップを読み込んだ結合テストで、実際の一覧画面を操作する。以下のマウント用ヘルパーは、Pinia・Vuetify を登録し、`MainLayout` を含む画面を `VApp` に包む PoC のテスト用ヘルパーを想定する。

```ts
it('再取得に失敗しても前回の結果を表示する', async () => {
  const view = mountInventorySearch()
  await view.search('A001')
  await vi.waitFor(() => expect(view.resultCards()).toHaveLength(2))

  server.use(http.get(url, () => new HttpResponse(null, { status: 500 })))
  await view.search('A001')
  await vi.waitFor(() => expect(view.errorMessage()).toContain('取得'))

  expect(view.resultCards()).toHaveLength(2)
})
```

`search` はフォーム入力と検索ボタンの操作、`resultCards` と `errorMessage` は画面に描画された要素の取得を担当する。応答を不正本文に差し替えたケースも同様に確認する。

## Storybook で確認する UI コンポーネント間の連携

検索フォームを `MainLayout` のメイン領域に置き、フッターの枠には画面側から検索ボタンを渡す。story の隣に置くテスト専用の `InventorySearchPanel` で、実際の `MainLayout`・`InventorySearchForm`・`InventorySearchFooter` を組み合わせる。ラッパーが入力と開閉の model を保持し、フッターの検索イベントをそのまま親に返す。preview のデコレーターで `VApp` に包む。アコーディオンを開いて商品コードを入力し、フッターの検索ボタンを押すと、親に検索イベントが届くことを確認する。この例の確認範囲は、部品の操作から親へのイベント通知まで。

```ts
import type { Meta, StoryObj } from '@storybook/vue3-vite'
import { expect, fn } from 'storybook/test'
import InventorySearchPanel from './InventorySearchPanel.vue'

const meta = {
  component: InventorySearchPanel,
} satisfies Meta<typeof InventorySearchPanel>

export default meta
type Story = StoryObj<typeof meta>

export const SearchFromFooter: Story = {
  args: { onSearch: fn() },
  play: async ({ canvas, userEvent, args }) => {
    const submit = canvas.getByRole('button', { name: '検索' })
    await expect(submit).toBeDisabled()

    await userEvent.click(canvas.getByRole('button', { name: '商品コード' }))
    const input = await canvas.findByRole('textbox', { name: '商品コード' })
    await userEvent.type(input, '   ')
    await expect(submit).toBeDisabled()
    await userEvent.clear(input)
    await userEvent.type(input, ' A001 ')
    await expect(submit).toBeEnabled()

    await userEvent.click(submit)
    await expect(args.onSearch).toHaveBeenCalledTimes(1)
    await expect(args.onSearch).toHaveBeenCalledWith({
      productCode: 'A001',
    })
  },
}
```

この例では、空入力時に検索ボタンが無効で、親へ条件オブジェクトを通知する UI を仮定している。ボタンの有効条件とイベントの引数は [在庫検索の部品契約](../04-detailed-design/ht-vue-components.md#在庫検索-poc-の部品と契約) に従う。実行手順は [Vitest と Storybook の連携手順](../05-unit-test/README.md#vitest-と-storybook-の連携手順) を参照する。

### Storybook と Pinia の使い分け

- 表示用の部品は `args` で固定の props を渡す。例えば、前回の結果2件と取得エラーを渡し、カードとエラーが同時に表示されることを確認する。
- 対象コンポーネントが Pinia を直接参照し、共有状態に応じた UI が確認対象なら、story ごとに新しいテスト用 Pinia を用意する。`createTestingPinia` の `initialState` で Store ID ごとに状態をセットする。操作が action に届くことまで確認する場合は action をスタブしたまま呼び出しを検証する。
- Store の取得・保存処理は Vitest で確認する。実際の Store、Orval、Zod をつないだ検索全体は Vitest＋MSW で確認する。

テスト用 Pinia の登録例。Store を使う story の `parameters.piniaInitialState` に、実際の Store ID をキーとする初期状態を置く。`.storybook/preview.ts` の `setup` でその状態を受け取り、レンダリング用の Vue アプリに登録する。既存の Vuetify 登録は同じ設定内に維持する。

```ts
import { createTestingPinia } from '@pinia/testing'
import { setup } from '@storybook/vue3-vite'
import { fn } from 'storybook/test'

setup((app, context) => {
  const initialState = context?.parameters.piniaInitialState
  if (initialState) {
    app.use(createTestingPinia({ initialState, createSpy: fn }))
  }
})
```

採用版の Storybook で `setup` に story のコンテキストが渡ることと、`@pinia/testing` が Pinia の採用版と互換であることを導入時に確認する。

[Storybook の操作テスト](https://storybook.js.org/docs/writing-tests/interaction-testing) · [Pinia のテスト用 Store](https://pinia.vuejs.org/cookbook/testing.html)

## 通信クライアントを確認する範囲

Store の単体テストでは API 関数を差し替え、保存やエラー時の状態を確認する。API 境界の単体テストでは生成クライアントを差し替え、HTTP 判定・Zod 検証を確認する。いずれも MSW の起動は不要。

在庫 PoC では、画面からの結合テストで生成クライアントも実行する。Node.js の MSW が Fetch を捕捉し、`GET /api/inventory?productCode=A001` が組み立てられたことを確認して応答を返す。Zod 検証、Store 更新、結果やエラーの表示までを確認する。この結合ケースを通信クライアントの送信内容の確認にも使うため、通信クライアント専用のテストは追加しない。

[PoC のテストアーキテクチャ図](../08-poc/ht-inventory-poc-architecture.drawio)

## 結合で立証すること

実際に組み合わせた部品・レイヤーの間で、操作、値、状態が設計どおりに渡ることを立証する。ケースごとに実物をつなぐ範囲と差し替える境界を記録する。

| 結合の範囲 | 立証すること | 手段 |
| --- | --- | --- |
| UI コンポーネント間 | フォームの入力がフッターの有効状態に反映され、検索イベントが親へ正しい値で届く。 | Storybook の `play`。親の取得処理はスパイに差し替える。 |
| 画面・データ処理間 | 画面から送信した条件が API に届き、応答の検証、Store の更新、結果やエラーの再表示につながる。 | Vitest＋Vue Test Utils＋MSW。画面・Store・Orval・Zodをつなぎ、HTTP 応答を差し替える。 |

**作成時点**は、基本設計で画面間・部品間の連携を決めるときに観点を挙げ、詳細設計でイベント・API・状態の契約が決まった時点で前提と期待結果を確定する。関連部品がつながる変更でテストコードを作り、結合を確認できる時点で実行する。

| ケース ID | 対応する基準・シナリオ | 実際につなぐ範囲 | 差し替え・前提 | 操作 | 期待結果 | 実行手段・結果 |
| --- | --- | --- | --- | --- | --- | --- |
| IT-INV-01 | AC2・SC-INV-02 | フォーム、MainLayout、フッター | 親の検索ハンドラーをスパイにする。 | 空白だけ入力し、次に ` A001 ` を入力して検索。 | 空白時はボタンが無効。検索時に `A001` が親へ1回届く。 | Storybook・作成予定 |
| IT-INV-02 | AC3・SC-INV-04 | 画面、Store、Orval、Zod | MSW で初回200、再検索時は不正な200本文。 | 同じ商品コードで再検索。 | 契約不一致で取得エラーを表示し、前回のカードも表示する。 | Vitest＋MSW・作成予定 |
| IT-INV-03 | AC2・SC-INV-02 | 画面、Store、Orval、Zod | MSW で正常データを返す。 | 空白だけ入力し、次に ` A001 ` を入力して検索。 | 空白だけでは API に送信しない。検索時のクエリに `productCode=A001` が入る。 | Vitest＋MSW・作成予定 |
| IT-INV-04 | AC2・SC-INV-03 | 画面、Store、Orval、Zod | MSW で `200 { items: [] }` を返す。 | 有効なコードを検索。 | カードがなく、「結果なし」が表示される。 | Vitest＋MSW・作成予定 |
| IT-INV-05 | AC2・SC-INV-04 | 画面、Store、Orval、Zod | 同じキーで成功後、MSW を 500 に切り替える。 | 同じコードで再検索。 | 取得エラーを表示し、前回のカードを保持する。 | Vitest＋MSW・作成予定 |
