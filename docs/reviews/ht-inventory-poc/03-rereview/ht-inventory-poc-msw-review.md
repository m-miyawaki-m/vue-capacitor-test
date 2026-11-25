# 方針レビュー記録（在庫管理 PoC：Node.js 22・Orval 生成 Faker/MSW のみ）

- レビュー日：2026-11-25
- レビュー担当：Claude Code（Herdr w3:p2）
- 依頼元：Codex（Herdr w3:p3）
- 対象：`docs/records/ht-inventory-poc.md`（Prism を外した最新版）、`docs/ht-design.md`（Node.js 22.22.2 以上の 22 系）
- 重点：生成、ブラウザーの MSW、Vitest の `msw/node`、Playwright でのクエリ観測、依存要件。must を優先して確認した。
- 結論：**現時点では同意しない**。must 2 件を反映すれば同意する。
- 件数：must 2 件、should 3 件

## 前提と根拠

- Prism の確認は、依頼どおり中止した。前回 `ht-inventory-poc-review.md` の M3（Prism と Node.js 24 の不一致）は、Prism の除外によって対象がなくなった。
- 依存要件の判断には、Prism 除外の連絡を受ける前に作業用ディレクトリで解決した試験用ロックファイルの結果を使った。依存は導入していない。この試験には、計画の依存一式（Vue、Vuetify、Router、Pinia、Zod、Vite 8、Vitest、jsdom、ESLint、Orval、MSW、Faker、Storybook、Playwright、Capacitor 8、create-vue）を含めた。このロックファイルは Prism 5.14.2 も含んでいたが、Prism を外しても Node.js の要件が厳しくなることはない。
  - 推移依存を含む 852 パッケージのうち、Node.js の要件（`engines.node`）を宣言している 495 件すべてが Node.js 22.22.2 で要件を満たした。
  - 下限が 22.22.2 ちょうどのパッケージがある。`jsdom@31.0.1`、`w3c-xmlserializer@6.0.0`、`nopt@10.0.1`、`abbrev@5.0.0` が `^22.22.2 || ...` を、`vue-router` の推移依存 `chokidar@6.0.0` が `>= 22.22` を要求している。このため 22.22.2 という下限は妥当で、これより下げられない。
  - Node.js 22.22.2 に付属する npm は 10.9.7 である。Faker が要求する `npm >=10` を満たす。
- 生成コードの中身と実際の動作は確かめていない。依頼どおり、着手後の確認項目として整理した。

## must

### M1. 生成した MSW・Faker のコードを本番ビルドから外せる出力構成になっていない

- 場所：`ht-inventory-poc.md:26`、`:45`
- 問題：:45 は、Fetch クライアント・型・MSW・Faker を同じ `src/api/generated/client/` に出力するとしている。Orval の既定の出力モード（`mode: 'single'`）では、モックのハンドラと Faker ファクトリがクライアントと同じファイルに出力される。その場合、画面がクライアントを import した時点で `msw` と `@faker-js/faker` も本番ビルドに入る。:26 の「配布ビルドに MSW・Faker のモックコードが含まれない」と両立しない。
- 修正案：`orval.config.ts` の出力を `mode: 'split'`（または `tags-split`）にし、モックを別ファイル（`*.msw.ts`）に分ける。モックのファイルは、`VITE_API_MOCK` が有効なときに動的 import するモック起動処理と、テストからだけ import する。Zod は今の計画どおり `src/api/generated/zod/` に別の出力先として生成する。着手後の確認として、本番ビルドの成果物に `msw`・`faker` の文字列が含まれないことを検索で確かめる手順を、:47 の検証項目に加える。

### M2. E2E でブラウザーに固定 2 件を返す方法が決まっていない

- 場所：`ht-inventory-poc.md:19`、`:25`、`:36`
- 問題：AC1 は「固定 2 件のカード」を期待値にしている。:25 は「固定応答が必要な E2E は専用モードで起動する」としているが、切り替えの値は `msw|off` の 2 つしかなく、専用モードの起動方法が書かれていない。Vitest ではテストのコードから固定値で上書きできる。しかしブラウザーの MSW は、アプリの中で起動するため外から上書きできない。このままでは、開発時と同じ乱数の Faker データが返り、AC1 を毎回同じ結果で判定できない。
- 修正案：次のどちらかに決めて明記する。
  - (a) モードを追加する。`VITE_API_MOCK=msw|msw-fixed|off` とし、`msw-fixed` では固定 2 件で上書きしたハンドラを登録する。Playwright の `webServer` は、このモードで開発サーバーを起動する。
  - (b) テストから上書きする。開発時だけ `window` に `worker` を公開し、Playwright から `page.evaluate` で `worker.use(...)` を呼んで固定応答に差し替える。

  変更する箇所が少なく、E2E の前提が起動コマンドで分かるため (a) を推奨する。

## should

### S1. Playwright で MSW のイベントを受け取る方法

- 場所：`ht-inventory-poc.md:36`
- 問題：`page.on("request")` で観測できなかったときの代わりに MSW の `request:start` で数えるとしている。しかしこのイベントはブラウザーの中で発生するため、Playwright のテスト側にどう渡すかが書かれていない。
- 修正案：代わりの方法を使う場合の受け渡しを明記する。例：モック起動処理が `request:start` で URL を `window.__mswRequests` に追加し、テストは `page.evaluate` でその配列を読んで、メソッド・パス・`productCode` と件数を確認する。代わりの方法をあらかじめ用意しておけば、`page.on("request")` で観測できるかどうかに左右されない。

### S2. MSW が処理しないリクエストの扱いと、E2E の起動先

- 場所：`ht-inventory-poc.md:25-26`
- 問題：次の 2 点が書かれていない。
  - `onUnhandledRequest` の設定。既定は警告だけなので、想定外のリクエストが出ても気づけない。
  - E2E を開発サーバーに対して実行するのか、本番ビルドのプレビューに対して実行するのか。ワーカーは開発時だけ配信するので、本番ビルドのプレビューでは MSW が動かない。
- 修正案：次の 2 点を明記する。
  - ブラウザーと `msw/node` の両方で、`/api/` 配下の未定義のリクエストは `onUnhandledRequest: 'error'` で失敗させる。
  - Playwright は開発サーバー（M2 のモード）に対して実行する。

### S3. 依存の脆弱性の確認を着手後の手順に入れる

- 場所：`ht-inventory-poc.md:44`、`:57`
- 問題：試験用ロックファイルの `npm audit` では、Prism 以外に `@vue/test-utils` の推移依存（`js-beautify`、`editorconfig`、`minimatch`、`glob`）で中程度の警告が出た。いずれも開発用の依存で、PoC の判定は妨げない。
- 修正案：依存を固定したら `npm audit` の結果を作業記録に残す、と :44 の手順に加える。開発用の依存だけの警告は記録にとどめてよい。

## 着手後に行う確認

これらは方針の確定を妨げない。着手後、最初に行い、結果を作業記録に残す。

1. Node.js 22.22.2 以上の 22 系に切り替え、`engine-strict=true` で `npm ci` が通ることを確かめる。このマシンの Node.js は現在 v20.20.0 のため、切り替えが必要。
2. `npm run api:generate` を 2 回実行して、差分がないことを確かめる（AC5）。`split` モードでモックが別ファイルになることも確かめる（M1）。
3. 生成した Faker ファクトリが `items` 配列全体の上書きを受け付けることを確かめる（:19）。
4. 採用した MSW の版で生成ハンドラが動き、固定のテスト origin を使った絶対 URL の要求が `msw/node` に届くことを確かめる（:27）。
5. ブラウザーで `msw` と `msw-fixed` の各モードでワーカーが起動し、未定義のリクエストがエラーになることを確かめる（M2、S2）。
6. Playwright の `page.on("request")` で、Service Worker が処理した GET とクエリを観測できるかを確かめる。観測できない場合は S1 の方法に切り替える。
7. 本番ビルドの成果物に、MSW のワーカー、`msw`、`faker` が含まれないことを確かめる（M1、:26）。

## 問題がないと判断した点

- Node.js の下限 22.22.2 は、推移依存の要件から見て最小で妥当な値である。`ht-design.md:31` と PoC の `:7` で揃っている。
- 対象外に「外部モックサーバー」を加え、AC5 から Prism を外したことで、Prism の参照は計画から消えている。
- `:30` の方針（MSW ではメソッド・パス・クエリをテストで明示的に確認し、仕様との整合は API 境界の Zod で確認する）は、MSW のハンドラがクエリを検証しないことを補う形として妥当である。
- Vitest で相対 URL を固定のテスト origin を使った絶対 URL に解決する方針（`:27`）は、`msw/node` で受けるための前提を満たす。

## 結論

現時点では同意しない。M1・M2 を反映すれば同意する。S1〜S3 と着手後の確認 1〜7 は、方針の確定を妨げない。
