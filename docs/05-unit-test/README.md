# 単体テスト

| 段階 | 業務の確認 | フレームワーク連携の確認 | 主な手段 |
| --- | --- | --- | --- |
| 単体 | 入庫・在庫管理の検索条件を正しく組み立てる。カードや詳細に必要な項目を取り出す。 | 画面制御 Store が画面状態を更新する。レスポンス Store の action が条件ごとに保存し、取得失敗時に前回成功データを保持する。生成 Zod スキーマが必須項目欠落や型違いを検出する。 | Vitest。部品の props・イベントは Vue Test Utils、表示状態やブラウザでの部品操作は Storybook |

共通 UI は Storybook で、HT の縦幅でのフッター、アコーディオンの開閉、長い商品名のカード、遷移候補が多い一覧を確認する。必要な操作は story の `play` に記述し、Vitest アドオンで実行する。採用する Vue・Vite の構成で Vitest と Storybook が動くことを導入時に確認する。

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
