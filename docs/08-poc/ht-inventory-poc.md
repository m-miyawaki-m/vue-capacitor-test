# 在庫管理の画面・API・テスト PoC 計画

## 着手条件

- Git の既定ブランチに、Vue 3・Vuetify 4・Vite・Vue Router、lint・型チェック・Vitest が動作する Web 基盤を先行して統合する。PoC のブランチはその統合コミットから作る。Android 疎通の変更と同じ設定・ロックファイルを並行で変更せず、共通設定の変更が必要な場合は順番に取り込む。
- GitHub を利用できる場合は既定ブランチを起点とする PR で進める。準備前はリポジトリの開発フローに従ってローカルに記録する。
- PoC は Node.js 22 LTS（22.22.2 以上の 22 系）を使う。`package.json` の `engines.node` を `^22.22.2` とし、`.npmrc` の `engine-strict=true` で検証する。既存の Node.js が要件未満なら切り替えてから依存を導入する。
- Windows の PowerShell で Node.js 22 と npm、インストール済みの Google Chrome を利用できることを確認する。

## Windows での起動とテスト

WSL2 の作業コピーをソース編集に使う。Git の準備後、Windows 側のローカルドライブにも作業コピーを用意し、同じ Git コミットをチェックアウトして検証する。Windows の PowerShell で依存をインストールし、API 生成、Vite、Storybook、Vitest、Playwright を実行する。依存ディレクトリは OS ごとに管理し、テスト結果に検証したコミットを記録する。

手動確認には Windows の Google Chrome を使う。Playwright は `channel: 'chrome'` で Windows にインストール済みの Google Chrome をテスト用セッションとして起動する。Storybook のブラウザテストで追加のブラウザが必要かは、採用するアドオンの設定に合わせて確認する。

[Microsoft の WSL ファイル配置ガイド](https://learn.microsoft.com/en-us/windows/wsl/setup/environment#file-storage) · [Playwright の Google Chrome 設定](https://playwright.dev/docs/browsers#google-chrome--microsoft-edge)

[PoC のアーキテクチャ・Store の再利用・検証フロー（draw.io）](ht-inventory-poc-architecture.drawio)

## 方針と範囲

在庫管理の参照業務を題材に、業務メニューから検索画面を開き、検索ボタンによる HTTP リクエスト、モック応答の検証と Pinia への保存、結果表示までをブラウザで確認する。初回の Android 疎通・撮影確認とは別の作業単位とし、Vue 3・Vuetify 4・Vite・Vue Router が動く Web 基盤を前提とする。Android の検証完了は着手条件に含めない。

| 対象 | 進め方 |
| --- | --- |
| 画面 | `MainLayout` にヘッダー、メイン領域、フッターを置く。業務メニューから在庫検索へ遷移し、商品コードを条件アコーディオンに入力してフッターの「検索」を押す。結果カードは商品コード・商品名・在庫数を表示する。 |
| API | ルートの OpenAPI 3.0.3 の `open.yaml` に `GET /api/inventory?productCode=...` を定義する。商品コードは前後の空白を除いて必須とし、仕様でも `required`、`minLength: 1`、空白以外の文字を求める `pattern` を指定する。成功本文は `items` 配列とし、空配列も許容する。商品コード・商品名・在庫数を項目とし、在庫数は0以上の整数とする。正常例を先に置き、空の例、HTTP 500 を定義する。 |
| 生成・検証 | Orval で Fetch クライアント、TypeScript 型、Zod スキーマ、Faker ファクトリ、MSW ハンドラを生成する。API 境界で HTTP ステータスを判定し、成功本文を生成済み Zod スキーマの `safeParse` で一度検証する。 |
| モック | 開発・自動テストでは明示的に MSW を起動し、Orval の生成ハンドラでリクエストを受ける。正常の自動テストでは生成 Faker ファクトリに固定2件の `items` 配列全体を渡して上書きする。開発時は生成 Faker データの表示も確認する。生成 API が配列の上書きを受け付けることを最初に検証する。空、500、契約不一致はケースごとに応答を差し替える。 |
| Store | 画面制御 Store は検索パネルの開閉と実行済み条件を保持する。レスポンス Store は前後の空白を除いた商品コード別の成功本文・取得状態・エラーを保持する。検索失敗時は同じキーの前回結果と取得エラーを表示する。同じキーでは最新要求の結果だけを反映する。 |
| 対象外 | 入庫登録・在庫の更新、実バックエンド、外部モックサーバー、認証、詳細画面、バーコード・OCR、Android ビルドや SDK、永続化。 |

## モックとテスト環境

- 開発サーバーでは `VITE_API_MOCK=msw|msw-fixed|off` を切り替える。MSW は開発時にだけ動的 import し、起動完了後に画面をマウントする。`msw` では生成 Faker データを返す。`msw-fixed` では固定2件の `items` で上書きしたハンドラを登録する。Playwright の `webServer` は `msw-fixed` を設定した開発サーバーを起動する。
- MSW ワーカーはライブラリの CLI で生成して Git 管理し、開発用ディレクトリからだけ配信する。アプリの配布ビルドにワーカーと MSW・Faker のモックコードが含まれないことを、生成アセット一覧とバンドルの依存モジュールで確認する。
- Vitest の結合テストは jsdom と Vuetify のセットアップを使う。Fetch の相対 URL は呼び出し境界で絶対 URL に解決し、ブラウザでは現在の origin、Vitest では固定のテスト origin を使う。生成クライアントが `msw/node` に届くことと、生成ハンドラが動く MSW の版を最初に確かめる。
- Storybook の story は props で状態を渡す。Vuetify の初期化を共通設定に置き、`play` は Storybook の Vitest アドオンで自動実行する。Playwright と Storybook で仮の縦画面 360×640 を確認する。

ブラウザの MSW と `msw/node` では `/api/` 配下の未定義リクエストをエラーにする。固定モードでは `request:start` のメソッドと URL をブラウザ側のテスト用配列に記録し、Playwright から `page.evaluate` で読み出す。

MSW では対象のメソッド・パス・クエリをテストで明示的に検証する。OpenAPI との受信データの整合は、アプリの API 境界で生成 Zod スキーマを使って確認する。

## 受入基準とテスト

| ID | 確認内容 | 方法 |
| --- | --- | --- |
| AC1 | メニューから在庫検索に進み、商品コードを入力してフッターのボタンを押すと、対応するクエリで GET が1回発生し、固定2件のカードが表示される。 | Windows の Playwright で Chrome を起動し、MSW の固定モードで開発サーバーを起動する。捕捉したメソッド・パス・クエリと件数を確認し、固定2件のカードを検証する。 |
| AC2 | 空白のみの入力は送信しない。入力の前後の空白を除いた商品コードをクエリに送る。空配列は結果なし、HTTP 500 は取得エラーとして表示する。 | Vitest と Vue Test Utils の結合テストで、画面・Store・生成クライアント・Zod を接続し、HTTP 側を MSW で差し替える。 |
| AC3 | 型違いなどの契約不一致を検出する。失敗した再取得で同じ条件の前回結果を上書きせず、別条件の結果は別に保持する。同一キーの古い成功・失敗は最新結果を上書きしない。契約不一致は取得エラーとして表示する。 | Vitest の API 境界と Store のテスト。Store の入力・期待値には固定データを使う。契約不一致時の画面表示は結合テストでも確認する。 |
| AC4 | 検索フォームと結果カードを単独で表示でき、検索パネルの開閉、空・エラーなど一覧の表示状態を確認できる。 | Storybook に story を置き、検索パネルの開閉を `play` で確認する。 |
| AC5 | 同じ仕様から生成物を再生成できる。 | `npm run api:generate` を2回実行して出力差分がないことを確認し、生成物のコミット後は再生成で Git 差分がないことも確かめる。 |

## 業務フローとシナリオ

| 業務フロー ID | 利用者 | 目的 | 開始条件 | 主な手順と分岐 | 完了状態 |
| --- | --- | --- | --- | --- | --- |
| BF-INV-01 | 在庫を確認する担当者 | 対象商品の在庫数を確認する。 | 業務メニューを開いている。 | 在庫検索を開き、商品コードで検索する。入力不備なら修正し、結果なし・取得失敗なら表示を確認して条件変更や再検索ができる。 | 対象商品の在庫数を確認できた。結果が得られない場合は状態を把握して終了または再検索する。 |

| シナリオ ID | 前提 | 操作 | 観察できる結果 | 受入基準と主な確認 |
| --- | --- | --- | --- | --- |
| SC-INV-01 在庫が見つかる | A001 の結果が2件ある。 | メニューから在庫検索を開き、A001 で検索する。 | 商品名と在庫数が載ったカードが2件表示される。 | AC1・Playwright |
| SC-INV-02 入力を修正する | 検索画面を開いている。 | 空白だけ入力し、次に ` A001 ` を入力して検索する。 | 空白だけでは検索ボタンが有効にならない。入力を直して検索すると A001 の結果が表示される。 | AC2・Storybook と Vitest＋MSW |
| SC-INV-03 結果がない | API が空配列を返す。 | 商品コードを入力して検索する。 | 「結果なし」が表示される。 | AC2・Vitest＋MSW |
| SC-INV-04 取得に失敗する | 同じ条件の成功結果があり、次の応答が 500 または仕様不一致。 | 同じ条件で再検索する。 | 取得エラーが表示され、前回のカードを引き続き確認できる。 | AC2・AC3・Vitest＋MSW。保持の分岐は Vitest 単体でも確認。 |

AC4 は部品の表示と操作を Storybook で確認する。AC5 は生成結果の再現性を確認する。業務シナリオに加えて、この2つの技術上の基準も受入判定に含める。

## 受入基準とケースの対応

テストコードは作成・実行前。テスト実装時にケース ID と実行結果を対応づける。

| 受入基準 | ケース | 確認資料 |
| --- | --- | --- |
| AC1 | ST-INV-01 | [総合テスト](../07-system-test/README.md#業務シナリオで立証すること) |
| AC2 | IT-INV-01、IT-INV-03〜05 | [結合テスト](../06-integration-test/README.md#結合で立証すること) |
| AC3 | UT-INV-01〜04、IT-INV-02 | [単体テスト](../05-unit-test/README.md#単体で立証すること) · [結合テスト](../06-integration-test/README.md#結合で立証すること) |
| AC4 | UT-INV-05〜06 | [単体テスト](../05-unit-test/README.md#単体で立証すること) |
| AC5 | GEN-INV-01 | PoC の生成担当が同じ仕様で2回生成し、出力差分と、コミット後の再生成で Git 差分がないことを記録する。 |

| ケース ID | 前提 | 操作 | 期待結果 | 実行手段・結果 |
| --- | --- | --- | --- | --- |
| GEN-INV-01 | 同じ `open.yaml`、Orval 設定、ロック済みの依存を使う。 | 生成を2回実行して比較する。生成物をコミットした後に再生成する。 | 2回の出力が一致し、コミット後の再生成でも `src/api/generated/` に Git 差分がない。 | PoC の生成担当・実行予定 |

## 変更予定と進め方

1. Web 基盤の依存関係を確認し、Orval・Zod・MSW・Faker、Vitest、Storybook、Playwright の採用版を npm ロックファイルで固定し、`npm audit` の結果を記録する。
2. `open.yaml`、`orval.config.ts`、手書きの API 境界とモック起動設定を用意する。Fetch クライアントの生成は `mode: "split"` とし、`src/api/generated/client/` 内でクライアント・型と MSW・Faker のモック出力を別ファイルにする。モック側のファイル名は生成結果に合わせる。モックは開発時の動的 import とテストからだけ参照する。Zod は `src/api/generated/zod/` に別出力し、生成物をコミットする。
3. `src/router/`、`src/layouts/`、`src/views/`、`src/components/`、`src/stores/` にメニュー・検索画面・Store を用意する。
4. `.storybook/` と stories、Vitest のテスト、Playwright の設定と E2E を追加する。生成、lint、型チェック、Vitest、Storybook のビルドと操作確認、Playwright、アプリのビルドの結果を作業記録に残す。

PoC の項目名とパスはサンプル契約とする。実際に採用した依存版、Node.js と実行環境、検証コマンド、結果はこの計画書に追記する。

[要件定義](../01-requirements/ht-design.md) · [単体テスト](../05-unit-test/README.md) · [結合テスト](../06-integration-test/README.md) · [総合テスト](../07-system-test/README.md) · [API 生成方針](../04-detailed-design/ht-api.md) · [Orval の生成設定](https://orval.dev/docs/reference/configuration/output/)

[在庫管理 PoC のレビューと対応の経過](../reviews/README.md#在庫管理-poc)

## Node.js 22 環境の確認

Node.js は 22.22.2 以上の 22 系とする。Capacitor 8、Vite 8、Orval 8、Faker 10 の公称要件を満たす。依存導入時には `engine-strict` を有効にして推移依存まで確認し、npm とライブラリの具体版を固定する。 [Capacitor 8](https://capacitorjs.com/docs/updating/8-0) · [Vite 8](https://v8.vite.dev/guide/) · [Orval](https://orval.dev/docs/installation/) · [Faker 10](https://v10.fakerjs.dev/guide/upgrading)

## 方針確定

Claude Code（Herdr `w3:p2`）がモック出力の分離と Playwright の固定応答モードを再確認し、must の残件は 0 件、方針に同意すると回答した。依存の固定、生成とテストの実走は着手後に行う。

## Web PoC の決定状況

| 決めること | 状況 | 現時点の内容 | 残っている判断・作業 |
| --- | --- | --- | --- |
| 検証する範囲 | 決定済み | 業務メニュー → 商品コード検索 → 取得・検証・保存 → 結果カード。詳細・入庫登録・Android は別作業。 | 本番用の業務ルールは要件定義で扱う。 |
| サンプル API | 決定済み | `GET /api/inventory`、必須の `productCode`、商品コード・商品名・在庫数の `items`。 | `open.yaml` を実装し、コード生成物を確認する。 |
| モックと検証 | 決定済み | Orval 生成の MSW・Faker を使う。E2E は固定2件。Zod で成功本文を検証する。 | 生成ハンドラと上書き引数を実走し、配布ビルドにモックが含まれないことを確認する。 |
| 合格条件 | 決定済み | AC1〜AC5。画面シナリオ、応答別の表示、Store の保持、部品の操作、生成の再現性。 | Windows で Vitest、Storybook、Playwright、再生成を実行して結果を記録する。 |
| 作業開始の準備 | 要確認 | Web 基盤を先に統合し、Windows で起動・テストする方針。 | Git の進め方を決め、Windows の Node.js・Chrome を確認し、Web 基盤を作成する。 |
