# API 生成と実行時検証

## Orval と Zod の連携

1. リポジトリルートの `open.yaml` に API のパス、要求、成功・エラーレスポンスを定義する。
2. Orval で Fetch クライアント、TypeScript 型、Zod スキーマを生成する。生成コマンドは `package.json` に定義し、仕様変更時に再生成する。
3. API の呼び出し境界で HTTP ステータスを確認し、対象レスポンスを生成済み Zod スキーマで検証する。検証を通過した成功レスポンスをレスポンス Store に格納する。通信失敗、HTTP エラー、契約不一致は区別し、最後に成功したデータを保持する。

最初は API 境界での明示的な `safeParse` を候補とする。Orval の Fetch 実行時検証も候補にできる。採用バージョンで HTTP ステータスごとの検証、カスタム mutator 利用時の挙動、エラーの型を確認して、検証箇所を一つに決める。

Zod の通常のオブジェクトスキーマは未定義キーを除去する。レスポンスを画面間で使い回すため、仕様に定義された成功レスポンス本文は全項目を保持する。仕様外のキーは既定で除去し、テストでも除去を確認する。仕様外の項目や HTTP ヘッダー・ステータスまで保存する要件がある場合は、保持方法と検証ルールを明示する。

[Orval のクライアントと Zod スキーマ生成](https://orval.dev/docs/guides/client-with-zod/) · [Orval の生成設定](https://orval.dev/docs/reference/configuration/output/) · [Zod の基本](https://zod.dev/basics) · [Zod のオブジェクト定義](https://zod.dev/api#objects)

## 業務 API の詳細設計で決めること

通信失敗・オフライン・タイムアウト・認証切れ（HTTP 401）・その他の HTTP エラー・Zod の契約不一致ごとに、表示メッセージ、再操作、エラー本文の検証要否を決める。初回の疎通確認では業務 API を扱わない。

## 導入時の確認

- サンプル API で通信コードと Zod スキーマを再現可能に生成できるか。正常、必須項目欠落、想定外の項目、HTTP エラーを確認する。

## 在庫 PoC の自動生成手順

Windows 側の作業コピーで Node.js 22 を使う。生成コマンドは依存をロックしたプロジェクトから実行する。

1. ルートの `open.yaml` に `GET /api/inventory` を定義し、`operationId: listInventory` を付ける。必須の `productCode` クエリと、JSON の 200 応答を定義する。200 応答の `items` は配列とし、各要素の `productCode`・`productName`・`quantity` を必須にする。正常・空の例と HTTP 500 も仕様に置く。
2. `orval.config.ts` に同じ入力を参照する Fetch 用と Zod 用の設定を置く。Fetch 側で MSW と Faker の生成を有効にする。
3. `package.json` の `scripts` に `"api:generate": "orval --config orval.config.ts"` を追加し、`npm run api:generate` を実行する。
4. 生成されたクライアント、型、Zod の応答スキーマ、MSW ハンドラ、Faker ファクトリのエクスポートと引数を確認する。生成物は手で変更せず、仕様または設定を変更して再生成する。
5. 2回生成して出力差分がないことを確認する。生成物をコミットした後は、再生成して `git status --short -- src/api/generated` が空であることも確認する。

### Orval の設定例

```ts
import { defineConfig } from 'orval'

export default defineConfig({
  inventoryClient: {
    input: './open.yaml',
    output: {
      mode: 'split',
      client: 'fetch',
      target: './src/api/generated/client/inventory.ts',
      schemas: './src/api/generated/client/models',
      mock: {
        generators: [{ type: 'msw' }, { type: 'faker' }],
      },
    },
  },
  inventoryZod: {
    input: './open.yaml',
    output: {
      client: 'zod',
      target: './src/api/generated/zod/inventory.zod.ts',
    },
  },
})
```

- アプリ本体はクライアント・型・Zod 出力を参照する。MSW と Faker の出力は開発時のモック起動処理とテストで読み込む。
- API 境界では、生成クライアントが返す HTTP ステータスを判定してから、成功レスポンスの本文を生成済み Zod スキーマの `safeParse` に渡す。成功なら `result.data` を Store に渡し、失敗なら契約不一致として扱う。実際のレスポンス型とエクスポート名は生成物に合わせる。
- 下記のテスト用ファイルと固定データは手書きで管理する。Orval の設定・生成の実走は PoC 着手時に行う。

[Orval の出力オプション](https://orval.dev/docs/reference/configuration/output/) · [Orval の Zod 生成](https://orval.dev/docs/guides/zod/) · [Zod の基本](https://zod.dev/basics) · [MSW によるテストデータの切り替え](../06-integration-test/README.md#msw-によるテストデータの切り替え)

### 在庫 PoC の成功と失敗の契約

`fetchInventory(criteria)` は、成功時に Zod で検証した本文を返し、失敗時に `ApiError` を投げる。例のコンストラクターは `new ApiError(kind, message)` とし、`kind` で失敗の種類を識別する。

| 応答・失敗 | API 境界の結果 |
| --- | --- |
| HTTP 200、仕様に適合する本文 | `safeParse` の成功データを返す。空の `items` も成功。 |
| HTTP 500 などの非成功ステータス | `ApiError('http', message)` を投げる。成功本文用のスキーマは適用しない。 |
| HTTP 200、必須項目がない、または型が不正 | `ApiError('contract', message)` を投げる。 |
| HTTP 応答を受け取れない通信失敗 | `ApiError('network', message)` を投げる。 |

レスポンス Store の取得 action はこのエラーを受け、最新要求であれば対象キーに失敗状態を記録する。最後に成功した本文と取得時刻は保持する。この API 境界の契約は在庫 PoC に適用する。
