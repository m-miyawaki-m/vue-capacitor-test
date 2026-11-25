# 方針 v1 レビュー記録（在庫管理の画面・API・テスト PoC）

- レビュー日：2026-11-25
- レビュー担当：Claude Code（Herdr w3:p2）
- 依頼元：Codex（Herdr w3:p3）
- 対象：`docs/records/ht-inventory-poc.md`（方針 v1）
- 参照：`docs/ht-design.md`、`docs/ht-api.md`、`docs/ht-store.md`、`docs/ht-testing.md`
- 観点：受入基準、生成方式、モック境界、再現性、テスト、環境や依存の実現性
- 結論：**現時点では同意しない**。must 3 件を反映すれば同意できる。
- 件数：must 3 件、should 6 件、nit 4 件

## 確認に使った情報

npm レジストリで 2026-11-25 時点の公開版と、Node.js の要件（`engines.node`）を確認した。

| パッケージ | latest | engines.node / peer |
| --- | --- | --- |
| `orval` | 8.49.0 | `>=22.18.0` |
| `@orval/mock` | 8.49.0 | peer: `msw >=2`、`@faker-js/faker >=8` |
| `msw` | 3.0.7（3.0.0 の公開は 2026-09-28） | `>=22.12.0` |
| `@faker-js/faker` | 10.6.0 | `>=24.0.0` などを含む |
| `zod` | 4.6.7 | — |
| `@stoplight/prism-cli` | 5.16.0 | **`>=24.18.0`**（5.15.11 は `>=24.14.0`、5.14.2 は `>=18.20.1`） |
| `storybook`、`@storybook/vue3-vite`、`@storybook/addon-vitest` | 10.6.9 | vue3-vite の peer: `vite ^5〜^8` |
| `msw-storybook-addon` | 3.0.3 | peer: `msw >=2` |
| `@playwright/test` | 1.65.1 | `>=20` |
| `pinia` | 4.1.0 | — |
| `vue-router` | 5.5.1 | — |

このマシンの WSL 環境の Node.js は `v20.20.0` だった。

Orval・MSW・Prism などの生成コードやモックの挙動は、次のとおりに分けて扱う。

- 公開版と要件だけを確認した。
- 生成コードの中身は実際に生成して確かめていない。このため、関連する指摘では「着手時に確認」としている。

## must

### M1. 着手時点のリポジトリの状態と、Web 基盤をどの作業単位で作るかが決まっていない

- 場所：`ht-inventory-poc.md:5`、`:28-31`
- 問題：方針は「Vue 3・Vuetify 4・Vite・Vue Router が動く Web 基盤」を前提にし、「Android の検証完了は着手条件に含めない」としている。ところが現時点のリポジトリには Vue プロジェクトも Git もない。Web 基盤は初回の作業（Android 疎通、`ht-design.md:21`）で作る予定になっている。このため、PoC が次のどれから始まるのか決まっていない。
  - (a) 初回の PR がマージされた後に始める。
  - (b) 初回の作業のうち Web 基盤の部分だけを先にマージしてから始める。
  - (c) PoC 自身が基盤を作る。

  決まっていないと、2 つの作業が `package.json`、ロックファイル、`src/router/`、`src/main.ts` を同時に変更して衝突する。CLAUDE.md の「既定ブランチから作業ブランチを作る」も実行できない。
- 修正案：「着手条件」の節を追加し、次の 3 点を明記する。
  - 起点にするブランチとコミット。例：初回の PR のうち、Web 基盤（create-vue、Vuetify、Router、lint・型チェック・Vitest）の部分がマージ済みであること。
  - Git とリモートが準備済みであること。
  - 初回の作業と並行して進める場合に、どちらがどのファイルを変更するか。例：`package.json` の依存追加は PoC 側の PR で行い、ロックファイルの衝突は後からマージする側が解消する。

### M2. Faker を使った正常データが再現できない

- 場所：`ht-inventory-poc.md:12`、`:20`
- 問題：Orval が生成する Faker ファクトリは、配列の要素数も乱数で決める。また 3 項目すべてを固定値にすると、Faker が生成した値は 1 つも使われなくなる。「Faker ファクトリを使う」と「結果を再現可能にする」の両立方法が決まっておらず、AC1 のカード件数や表示内容を E2E で毎回同じ値で検証できない。
- 修正案：次のどちらかに決めて明記する。
  - (a) 固定値で上書きする。生成されたファクトリに `items` 配列全体（例：2 件）を上書きとして渡す。Faker は、上書きしていない項目の型と形を作るためだけに使う。AC1 は固定の 2 件で検証する。
  - (b) 乱数を固定する。テストの準備処理とブラウザのモック起動時に `faker.seed(<固定値>)` を呼ぶ。期待値はその乱数に依存すると明記する。

  Orval の `mock.useExamples` で `open.yaml` の example を返す方式も候補にできる。その場合は Prism と同じデータになる。

  E2E の安定性を考えると (a) を推奨する。生成されたファクトリが上書きを受け付ける形（引数の型）は、着手時に確認する。

### M3. Prism の Node.js 要件が、設計書で決めた Node.js の版と合わない

- 場所：`ht-inventory-poc.md:28`、`ht-design.md:29`
- 問題：設計書は「Node.js 24 LTS（24.12 以上）」としている。しかし `@stoplight/prism-cli` 5.16.0 は `engines.node >=24.18.0`、5.15.11 は `>=24.14.0` を要求する。Prism の版をロックファイルで固定すると、24.12〜24.17 の環境では要件を満たさない。npm の既定では警告だけで止まらないため、問題に気づきにくい。なお、このマシンの Node.js は今 `v20.20.0` で、Orval（`>=22.18.0`）も MSW（`>=22.12.0`）も要件を満たさない。
- 修正案：次のどちらかに決め、`ht-design.md:29` と PoC の方針を揃える。
  - (a) Node.js の下限を 24.18 以上に上げる。`.nvmrc` または `package.json` の `engines` と、`.npmrc` の `engine-strict=true` で強制する。
  - (b) Prism を `>=18.20.1` を要求する 5.14 系に固定する。

  あわせて、作業前に Node.js 24 系への切り替えが必要なことを、着手条件（M1）に入れる。

## should

### S1. Vitest の環境と、生成されたクライアントの URL の扱いが決まっていない

- 場所：`ht-inventory-poc.md:21`、`:29`
- 問題：AC2 では Vitest の中で MSW（`msw/node`）を使って HTTP を差し替える。しかし Orval の Fetch クライアントは、通常 `/api/inventory` のような相対 URL を生成する。Node の `fetch` は相対 URL を解決できず、テスト環境によっては失敗する。また、Vuetify を jsdom で動かすには `server.deps.inline: ['vuetify']` の設定や `ResizeObserver` などの代替実装が必要になることが多い。
- 修正案：次の 2 点を明記する。
  - ベース URL の付け方。Orval の `baseUrl` を使うか、手書きの API 境界（カスタム mutator）で `location.origin` を基準に解決する。
  - Vitest の環境（jsdom か happy-dom か）と、Vuetify 用のセットアップファイル。

  着手時に「生成したクライアントが MSW の Node 側で受けられる」ことを最初に確かめる。

### S2. MSW の起動・無効化の条件と、版の組み合わせが決まっていない

- 場所：`ht-inventory-poc.md:12`、`:20`
- 問題：「明示的に起動する」の方法が書かれていない。次の点も決まっていない。
  - ブラウザ用の Service Worker ファイル `public/mockServiceWorker.js` の生成とコミット。
  - 本番ビルドに MSW を含めないための条件。
  - Prism を使うときに MSW を無効にする切り替え方法。

  また、MSW の latest は 3.0.7 だが、Orval 8.49.0 の peer は `msw >=2` としか書かれていない。生成されるハンドラが MSW 3 で動くかは確認していない。
- 修正案：次の 3 点を明記する。
  - 切り替えは環境変数で行う。例：`VITE_API_MOCK=msw|prism|off`。`main.ts` で動的 import し、`msw` の場合だけ `worker.start()` を呼ぶ。
  - `mockServiceWorker.js` の生成方法とコミットするかどうか。
  - 本番ビルドの成果物に MSW が含まれないことの確認方法。

  MSW は、生成したハンドラが動くことを確かめた版（2 系か 3 系）を固定する。

### S3. Storybook の `play` の実行方法と、ブラウザの依存が決まっていない

- 場所：`ht-inventory-poc.md:23`、`:31`
- 問題：AC4 の「`play` で確認」を、手動で行うのか、`@storybook/addon-vitest` で自動実行するのかが決まっていない。`@storybook/addon-vitest` で自動実行する場合も、Playwright で E2E を行う場合も、Chromium の実行に必要な OS のライブラリを WSL に入れる必要がある（`npx playwright install --with-deps chromium` は sudo が必要）。また、story が Store や API を呼ぶ場合は、`msw-storybook-addon` と Pinia・Vuetify のデコレーターが必要になる。
- 修正案：次の 3 点を明記する。
  - story は props だけで表示する部品に限り、API は呼ばない。この場合 `msw-storybook-addon` は不要になる。
  - `play` は `@storybook/addon-vitest` で自動実行し、そのコマンドを検証コマンドに含める。
  - Playwright のブラウザと OS ライブラリを準備する手順を、着手条件に入れる。

### S4. Prism で確認する内容の細部が決まっていない

- 場所：`ht-inventory-poc.md:10`、`:12`、`:24`
- 問題：次の点が決まっていない。
  - `open.yaml` の OpenAPI の版（3.0 系か 3.1 系か）。Prism と Orval の両方で問題なく扱える版にする必要がある。
  - 正常と空の 2 つの example があるとき、Prism がどちらを返すか。指定しなければ先頭の example を返すため、AC5 の「正常応答」が確実に出るとは限らない。
  - 必須クエリが欠けたときに Prism が返す拒否の期待値。HTTP ステータス（通常は 422）と本文。
  - 「空白を除いて必須」は、OpenAPI の `required` だけでは表せない。空白だけの値を Prism が拒否しない。
- 修正案：次の 4 点を明記する。
  - OpenAPI 3.0.3 を推奨する。
  - 正常の example を先頭に置くか、`Prefer: example=<名前>` を使う。
  - AC5 の拒否の期待値を「HTTP 422」とする。
  - 空白だけの値は画面側で送信しない。仕様にも `minLength: 1` と `pattern: '\S'` を入れるかどうかを決める。

### S5. Orval の出力構成と、再生成の再現性を確認する方法が決まっていない

- 場所：`ht-inventory-poc.md:11`、`:24`、`:29`、`:33`
- 問題：次の点が決まっていない。
  - Fetch クライアントとモックの出力（`client: 'fetch'` と `mock`）と、Zod の出力（`client: 'zod'`）は、`orval.config.ts` で別々の出力先にするのが一般的である。その出力先と、ファイル名の対応。
  - 生成ファイルをコミットするかどうか。これが未定だと、AC5 の「繰り返して生成結果を確認」の判定方法も決まらない。
- 修正案：次の 2 点を明記する。
  - 出力先の構成。例：`src/api/generated/client/`、`src/api/generated/zod/`、`src/api/generated/mocks/`。
  - AC5 の判定方法。2 回生成した結果のディレクトリを比較し、差分がないことを確かめる。Git の準備後は、再生成した後に `git diff --exit-code src/api/generated` で確かめる。

  生成ファイルはコミットすることを推奨する。レビューのときに生成コードの差分を見られるためである。

### S6. Store の同時取得、検索キーの正規化、契約不一致の表示が受入基準にない

- 場所：`ht-inventory-poc.md:13`、`:22`、`ht-store.md:23`
- 問題：次の点が受入基準にない。
  - `ht-store.md:23` には「同じキーでは最新の要求だけが反映される」とあるが、AC3 にその確認がない。
  - 商品コードを前後の空白を除いてからキーにするのかが決まっていない。
  - 契約不一致のときに画面に何を表示するか。AC2 は 500 の表示だけを定め、AC3 は検出だけを定めている。
- 修正案：次の 3 点を加える。
  - AC3 に「同じキーで要求が重なったとき、後から出した要求の結果だけが反映される」を加える。
  - 検索キーは前後の空白を除いた商品コードとする、と明記する。
  - 契約不一致のときの画面表示を、500 と同じ「取得エラー」にするか、別の表示にするかを決める。

## nit

### N1. AC1 の「GET が 1 回」を数える方法

- 場所：`ht-inventory-poc.md:20`
- 修正案：Playwright の `page.on('request')` で数えるか、MSW のライフサイクルイベント（`request:start`）で数えるかを明記する。リクエストを Service Worker が処理する場合も Playwright の request イベントで数えられるかは、着手時に確認する。

### N2. 設計書からこの PoC へのリンク

- 場所：`ht-design.md:25`
- 修正案：設計書は Orval・Storybook・Playwright を「初回の対象外」としている。そこに「別作業の PoC として `records/ht-inventory-poc.md` で扱う」と一文添えると、資料間で矛盾して見えない。

### N3. HT の画面サイズ

- 場所：`ht-inventory-poc.md:20`、`:23`
- 修正案：Playwright の viewport と Storybook の表示サイズに、仮の HT 画面サイズ（例：360×640）を指定する。フッターの配置を実際に近い条件で確認できる。

### N4. 依存の固定と記録先

- 場所：`ht-inventory-poc.md:28`、`:33`
- 修正案：採用した版、Node.js の版、生成ファイルのコミット方針、Prism のポートを、どのファイルに記録するかを明記する。例：`docs/records/ht-inventory-poc.md` の末尾に節を設ける。

## 問題がないと判断した点

- 受入基準 AC1〜AC5 は、画面・API・Store・Storybook・Prism と対応している。目的（画面操作から表示までの経路を確認する）に対して過不足がない。
- 対象外（`:14`）が明確で、初回の Android の作業と範囲が重ならない。
- API 境界で `safeParse` を 1 回だけ行う方針は、`ht-api.md:9` の候補と一致する。
- 失敗時に前回の結果を保持する方針は、`ht-store.md:21` と一致する。
- MSW（開発・自動テスト）と Prism（独立した HTTP の確認）の役割分担と、両者を同時に動かさない切り替えの考え方は妥当である。

## 結論

現時点では同意しない。M1〜M3 を反映すれば同意する。S1〜S6 は、着手時の最初の確認（生成と MSW の Node 側での疎通）で解消できる内容も含むが、方針に記載しておくことを推奨する。
