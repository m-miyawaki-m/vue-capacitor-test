# 単体テスト

| 段階 | 業務の確認 | フレームワーク連携の確認 | 主な手段 |
| --- | --- | --- | --- |
| 単体 | 入庫・在庫管理の検索条件を正しく組み立てる。カードや詳細に必要な項目を取り出す。 | 画面制御 Store が開閉状態、実行済み条件の保存と切り替えを扱う。レスポンス Store の action が条件ごとに保存し、取得失敗時に前回成功データを保持する。生成 Zod スキーマが必須項目欠落や型違いを検出する。 | Vitest。部品の props・イベントは Vue Test Utils、部品単独の表示状態や操作は Storybook |

共通 UI は Storybook で、HT の縦幅でのフッター、アコーディオンの開閉、長い商品名のカード、遷移候補が多い一覧を確認する。必要な操作は story の `play` に記述し、Vitest アドオンで実行する。採用する Vue・Vite の構成で Vitest と Storybook が動くことを導入時に確認する。

複数の部品を実際につないでイベントの受け渡しを見る場合は、[Storybook の UI コンポーネント間連携](../06-integration-test/README.md#storybook-で確認する-ui-コンポーネント間の連携) に置く。

[Android 疎通と撮影の単体テスト](#android-疎通と撮影の単体テスト) では、Web 版プラグインと撮影結果の判定を確認する。

[Vue のテストガイド](https://vuejs.org/guide/scaling-up/testing.html) · [Storybook の Vitest アドオン](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon/index) · [在庫 PoC の受入基準](../08-poc/ht-inventory-poc.md#受入基準とテスト)

## Vitest と Storybook の連携手順

### 設定と実行

1. Storybook を Vue 3・Vite 向けに設定し、`.storybook/main.ts` のフレームワークに `@storybook/vue3-vite` を使う。`.storybook/preview.ts` からアプリと同じ Vuetify 設定を登録する。`MainLayout` を描画する story は、Vuetify のレイアウトを初期化するため preview のデコレーターで `VApp` に包む。
2. Storybook の導入後、`npx storybook add @storybook/addon-vitest` を実行する。生成・変更された設定を確認し、Vitest の `storybook` プロジェクトに `storybookTest` プラグインと Playwright プロバイダーの Browser Mode があることを確認する。プロジェクト名が違う場合は実行コマンドの名前を合わせる。
3. 通常の Vitest プロジェクトには Store・ロジックのテストと Vue Test Utils の設定を置く。Storybook プロジェクトには stories とブラウザの設定を置き、jsdom 専用の設定は通常のプロジェクト内に留める。
4. Storybook の画面で story を開き、表示と操作を確認する。自動実行は `npx vitest run --project=storybook`、通常の単体テストはプロジェクトの `npm run test:unit -- --run` を使う。通常のスクリプトから Storybook プロジェクトも実行される場合は、通常用のプロジェクト名で絞る。
5. Storybook のアドオンが要求するブラウザを Windows 側で準備する。アドオンの自動設定は通常 Playwright 管理の Chromium を使う。在庫 PoC の E2E で Windows の既存 Chrome を指定する設定は別に管理する。

### story にテスト用の値と操作を置く

検索フォームを閉じた状態から始め、見出しをクリックすると入力欄が表示される例。props・ラベルは実装に合わせる。

```ts
import type { Meta, StoryObj } from '@storybook/vue3-vite'
import { expect, fn } from 'storybook/test'
import { ref } from 'vue'
import InventorySearchForm from './InventorySearchForm.vue'

const meta = {
  component: InventorySearchForm,
} satisfies Meta<typeof InventorySearchForm>

export default meta
type Story = StoryObj<typeof meta>

export const Closed: Story = {
  args: { productCode: '', expanded: false, 'onUpdate:expanded': fn() },
  render: (args) => ({
    components: { InventorySearchForm },
    setup: () => ({ args, expanded: ref(args.expanded) }),
    template: `
      <InventorySearchForm
        :product-code="args.productCode"
        v-model:expanded="expanded"
        @update:expanded="args['onUpdate:expanded']?.($event)"
      />
    `,
  }),
  play: async ({ canvas, userEvent, args }) => {
    const panel = canvas.getByRole('button', { name: '商品コード' })
    await expect(panel).toHaveAttribute('aria-expanded', 'false')
    await userEvent.click(panel)
    await expect(args['onUpdate:expanded']).toHaveBeenCalledWith(true)
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

## 単体で立証すること

対象の関数・Store・単独部品について、決められた入力や初期状態から期待する戻り値、状態、イベントが得られることを立証する。外部の取得関数やコールバックは差し替え、対象の振る舞いを絞って確認する。

- 関数は入力の正規化、境界値、失敗時の分類を確認する。API 処理の単体では生成クライアントを差し替え、HTTP 判定と Zod 検証を実行する。
- Store は条件別の保存、同じキーの再取得失敗時の保持、応答順が逆転したときの更新を確認する。
- UI 部品単独では Storybook の固定 props で表示・操作・通知を確認する。

**作成時点**は、詳細設計で契約や期待結果を決めた時点で観点とケースを用意する。テストコードは対象の実装と同じ変更で作り、レビューに出す時点で実行結果を確認する。

| ケース ID | 対象と根拠 | 初期状態・差し替え | 入力・操作 | 期待結果 | 実行手段・結果 |
| --- | --- | --- | --- | --- | --- |
| UT-INV-01 | レスポンス Store・AC3 | A001 の成功結果2件。次の API 呼び出しはエラー。 | 同じキーで再取得。 | データ2件と前回の取得時刻が残り、エラーが記録される。 | Vitest・作成予定 |
| UT-INV-02 | レスポンス Store・AC3 | API が A001 と B002 で別の成功データを返す。 | A001、B002 の順に取得する。 | キーごとに対応する結果を参照できる。 | Vitest・作成予定 |
| UT-INV-03 | レスポンス Store・AC3 | 同一キーで先発・後発の要求を開始し、応答順を制御する。 | 後発を成功させた後、先発を成功または失敗させる。 | 後発の結果・状態・取得時刻が変わらない。 | Vitest・作成予定 |
| UT-INV-04 | API 境界・AC3 | 生成クライアントが HTTP 200 と、在庫数を文字列にした本文を返す。 | 在庫取得関数を呼ぶ。 | `kind: 'contract'` のエラーになる。 | Vitest・作成予定 |
| UT-INV-05 | 検索フォーム・AC4 | 閉じたフォームに固定 props を渡す。 | story を開き、見出しを押す。 | パネルが開き、商品コード入力欄が表示される。 | Storybook・作成予定 |
| UT-INV-06 | `InventoryResultCard` と `InventorySearchResults`・AC4 | カード用データ、空、エラーの story をそれぞれ準備する。 | 各 story を表示する。 | 商品名・在庫数、結果なし、取得エラーがそれぞれの状態で表示される。 | Storybook・作成予定 |


## Android 疎通と撮影の単体テスト

Vitest で、Web 版 `HtBridge.ping` の固定応答と空入力の拒否を確認する。撮影処理ではプラットフォーム判定と公式 Camera を差し替え、成功時に表示用の `webPath` を取得すること、キャンセル・権限拒否・その他の失敗を分類できること、Web では Camera を呼ばないことを確認する。

Java 側の単体テストには JUnit を使う。入力検証・固定応答の生成を純粋な Java の処理として切り出し、Android SDK の実行時 API、`PluginCall`、`android.util.Log` との接続はプラグイン側が担当する。JUnit では正常な入力から返す値と、空入力を拒否する判定を確認する。テストは `android/app/src/test/java/` に置き、`android/` で Gradle Wrapper の `testDebugUnitTest` を実行する。Windows は `gradlew.bat`、Linux は `./gradlew` を使う。

[Android のローカル単体テスト](https://developer.android.com/training/testing/local-tests)

[Android アーキテクチャの立証範囲](../03-overview-design/ht-android-architecture.md#機能ごとの立証範囲)
