# 結合テスト

| 段階 | 業務の確認 | フレームワーク連携の確認 | 主な手段 |
| --- | --- | --- | --- |
| 結合 | 検索ボタンで取得が始まり、成功・結果なし・失敗に応じた表示になる。読み取り結果が呼び出し元の入力欄に反映される。 | 画面、実際の Pinia Store、Orval 生成クライアント、Zod をつなぎ、HTTP 応答だけを差し替える。画面制御と取得結果がそれぞれの Store に反映されることを確認する。読み取りの結合ではプラグインの応答を差し替える。 | Vitest と Vue Test Utils。実ブラウザが必要な場合は Playwright |

在庫 PoC では `msw/node` で応答を切り替え、送信クエリ、画面の表示、検証失敗時のデータ保持を確認する。

## 初回の疎通画面

- 画面テストではプラグインの応答を差し替え、ボタンからの呼び出し、成功・失敗の表示、撮影のキャンセル（`OS-PLUG-CAMR-0006`）、権限拒否（`OS-PLUG-CAMR-0003`）、その他の撮影エラー、Web の案内を確認する。
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

モックのセットアップを読み込んだ結合テストで、実際の一覧画面を操作する。以下のマウント用ヘルパーは、Pinia・Vuetify を登録した PoC のテスト用ヘルパーを想定する。

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
