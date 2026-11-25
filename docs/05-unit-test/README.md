# 単体テスト

| 段階 | 業務の確認 | フレームワーク連携の確認 | 主な手段 |
| --- | --- | --- | --- |
| 単体 | 入庫・在庫管理の検索条件を正しく組み立てる。カードや詳細に必要な項目を取り出す。 | 画面制御 Store が画面状態を更新する。レスポンス Store の action が条件ごとに保存し、取得失敗時に前回成功データを保持する。生成 Zod スキーマが必須項目欠落や型違いを検出する。 | Vitest。部品の props・イベントは Vue Test Utils、部品単独の表示状態や操作は Storybook |

共通 UI は Storybook で、HT の縦幅でのフッター、アコーディオンの開閉、長い商品名のカード、遷移候補が多い一覧を確認する。必要な操作は story の `play` に記述し、Vitest アドオンで実行する。採用する Vue・Vite の構成で Vitest と Storybook が動くことを導入時に確認する。

複数の部品を実際につないでイベントの受け渡しを見る場合は、[Storybook の UI コンポーネント間連携](../06-integration-test/README.md#storybook-で確認する-ui-コンポーネント間の連携) に置く。

Web 版の `HtBridge` は Vitest で固定応答と不正入力を確認する。

[Vue のテストガイド](https://vuejs.org/guide/scaling-up/testing.html) · [Storybook の Vitest アドオン](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon/index) · [在庫 PoC の受入基準](../08-poc/ht-inventory-poc.md#受入基準とテスト)

## Vitest と Storybook の連携手順

### 設定と実行

1. Storybook を Vue 3・Vite 向けに設定し、`.storybook/main.ts` のフレームワークに `@storybook/vue3-vite` を使う。`.storybook/preview.ts` からアプリと同じ Vuetify 設定を登録する。
2. Storybook の導入後、`npx storybook add @storybook/addon-vitest` を実行する。生成・変更された設定を確認し、Vitest の `storybook` プロジェクトに `storybookTest` プラグインと Playwright プロバイダーの Browser Mode があることを確認する。プロジェクト名が違う場合は実行コマンドの名前を合わせる。
3. 通常の Vitest プロジェクトには Store・ロジックのテストと Vue Test Utils の設定を置く。Storybook プロジェクトには stories とブラウザの設定を置き、jsdom 専用の設定は通常のプロジェクト内に留める。
4. Storybook の画面で story を開き、表示と操作を確認する。自動実行は `npx vitest run --project=storybook`、通常の単体テストはプロジェクトの `npm run test:unit -- --run` を使う。通常のスクリプトから Storybook プロジェクトも実行される場合は、通常用のプロジェクト名で絞る。
5. Storybook のアドオンが要求するブラウザを Windows 側で準備する。アドオンの自動設定は通常 Playwright 管理の Chromium を使う。在庫 PoC の E2E で Windows の既存 Chrome を指定する設定は別に管理する。

### story にテスト用の値と操作を置く

検索フォームを閉じた状態から始め、見出しをクリックすると入力欄が表示される例。props・ラベルは実装に合わせる。

```ts
import type { Meta, StoryObj } from '@storybook/vue3-vite'
import { expect } from 'storybook/test'
import InventorySearchForm from './InventorySearchForm.vue'

const meta = {
  component: InventorySearchForm,
} satisfies Meta<typeof InventorySearchForm>

export default meta
type Story = StoryObj<typeof meta>

export const Closed: Story = {
  args: { productCode: '', initiallyExpanded: false },
  play: async ({ canvas, userEvent }) => {
    const panel = canvas.getByRole('button', { name: '商品コード' })
    await expect(panel).toHaveAttribute('aria-expanded', 'false')
    await userEvent.click(panel)
    await expect(canvas.getByRole('textbox', { name: '商品コード' })).toBeVisible()
  },
}
```

結果カードの長い商品名、一覧の空・エラーなどはそれぞれ別の story を作り、`args` で固定値を渡す。操作が必要な story だけに `play` を追加する。上の例では、テスト結果を Storybook の Interactions パネルと Vitest CLI の両方で確認できる。Store の同時取得やエラー時のデータ保持は通常の Vitest で扱う。

[Storybook の Vue・Vite 構成](https://storybook.js.org/docs/get-started/frameworks/vue3-vite) · [Storybook の Vitest アドオン](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon/index) · [Storybook の操作テスト](https://storybook.js.org/docs/writing-tests/interaction-testing)

## レスポンス Store の単体テスト

Pinia をテストごとに作り、Store の取得 action を直接呼ぶ。API 境界の関数を差し替え、返すデータや失敗のタイミングをテストで決める。Store が公開する参照方法で、結果・取得状態・エラーを確認する。

| 観点 | 前提と操作 | 期待結果 |
| --- | --- | --- |
| 正常取得 | `A001` の検索に成功する | `A001` のキーに成功データが保存される |
| 検索キーの正規化 | `' A001 '` で検索し、`A001` として参照する | 正規化したキーに保存し、API にも `A001` を渡す |
| 条件ごとの保持 | `A001`、`B002` の順に取得する | 各キーに対応する結果が残る |
| 再取得の失敗 | `A001` の取得に成功した後、同じキーで取得に失敗する | 最後に成功したデータと取得時刻を保持し、取得エラーを記録する |
| 同じキーの同時取得 | 同じキーで先発・後発の要求を開始し、後発を先に成功させる | 先発の成功・失敗をあとから受け取っても、後発のデータと状態を維持する |

以下は再取得失敗の例。API 境界と Store の公開メソッド名は実装に合わせる。この例では Store が取得失敗を状態に記録し、action 内で処理する契約を想定する。

```ts
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, expect, it, vi } from 'vitest'
import { ApiError, fetchInventory } from '@/api/inventory'
import { useInventoryResponseStore } from '@/stores/inventory-response'

vi.mock('@/api/inventory', async (importOriginal) => ({
  ...await importOriginal<typeof import('@/api/inventory')>(),
  fetchInventory: vi.fn(),
}))
const fetchMock = vi.mocked(fetchInventory)

beforeEach(() => {
  setActivePinia(createPinia())
  fetchMock.mockReset()
})

it('同じ条件の再取得に失敗しても前回の結果を保持する', async () => {
  const store = useInventoryResponseStore()
  const criteria = { productCode: 'A001' }
  const lastSuccess = {
    items: [{ productCode: 'A001', productName: '商品A', quantity: 12 }],
  }

  fetchMock
    .mockResolvedValueOnce(lastSuccess)
    .mockRejectedValueOnce(new ApiError('http', '在庫の取得に失敗しました'))

  await store.searchInventory(criteria)
  const fetchedAt = store.getSearchState(criteria)?.fetchedAt
  expect(fetchedAt).toBeDefined()

  await store.searchInventory(criteria)

  expect(fetchMock).toHaveBeenNthCalledWith(2, criteria)
  const state = store.getSearchState(criteria)
  expect(state?.data).toEqual(lastSuccess)
  expect(state?.status).toBe('error')
  expect(state?.error).toMatchObject({ kind: 'http' })
  expect(state?.fetchedAt).toBe(fetchedAt)
})
```

同時取得はタイマーで待つ方法より、テスト側から解決・拒否できる Promise を2つ用意して応答順を制御すると再現しやすい。Orval クライアント・HTTP・Zod まで確認するケースは、[MSW を使った結合テスト](../06-integration-test/README.md#msw-によるテストデータの切り替え) に置く。

[Pinia のテストガイド](https://pinia.vuejs.org/cookbook/testing.html)


## API 境界の単体テスト

Vitest で生成 Fetch クライアントの呼び出しを差し替える。手書きの API 境界と生成済み Zod スキーマは実物を使う。

| 前提 | 期待結果 |
| --- | --- |
| 200 と正常な本文 | 検証済みの本文が返る。空配列も成功として返る。 |
| 500 とエラー本文 | `kind: 'http'` の `ApiError` で拒否される。 |
| 200 と必須項目欠落または型違いの本文 | `kind: 'contract'` の `ApiError` で拒否される。 |
| クライアントの通信失敗 | `kind: 'network'` の `ApiError` で拒否される。 |

[在庫 PoC の成功と失敗の契約](../04-detailed-design/ht-api.md#在庫-poc-の成功と失敗の契約)
