# 資料分割レビュー記録（8 フェーズ構成）

- レビュー日：2026-11-25
- レビュー担当：Claude Code（Herdr w3:p2）
- 依頼元：Codex（Herdr w3:p3）
- 対象：`docs/README.md` と、`01-requirements/` から `08-poc/` までの各資料
- 観点：
  - フェーズをまたぐ本文が残っていないか。
  - 分割で、確定事項・受入基準・テスト項目を取りこぼしていないか。
  - リンクと、各資料の責務が整合しているか。
- 結論：**分割に同意する**。must は 0 件。
- 件数：must 0 件、should 5 件、nit 4 件

## 分割前の資料との比較範囲

Git の履歴がないため、分割前の版との厳密な差分比較はできない。代わりに、このセッションで私が読んだ分割前の資料と照合した。

| 分割前の資料 | 照合に使った版 |
| --- | --- |
| `ht-design.md` | 再レビュー時の全文。その後の R-M1 対応部分（CAMERA 権限の宣言、拒否の再現手順、エラーコード）は、最終確認のときに grep で見た行。 |
| `ht-testing.md` | 再レビュー時の全文と、R-M1 対応後の :22、:28、:30。 |
| `ht-layout-components.md`、`ht-store.md`、`ht-api.md`、`ht-technical-research.md` | 再レビュー時の全文。 |
| `records/ht-inventory-poc.md` | MSW 版の最終確認のときに読んだ :25-26、:47、:59 と、MSW 再レビュー時の全文。 |

次の 3 点は、分割前の本文を見ていない。このため、`reviews/ht-design-review-response.md:22-25` の対応内容と、分割後の本文が一致するかだけを確かめた。

- R-S2：Vuetify 4 で疎通画面を構成する。
- R-S3：入力欄から空の値を送り、Java 側の `INVALID_MESSAGE` を確認する。
- R-N2：Router に `createWebHashHistory` を使い、未定義のルートは `/` に戻す。

照合の結果は次のとおり。

- R-S2 は `04-detailed-design/ht-native-and-screen.md:14` にある。
- R-S3 は `ht-native-and-screen.md:14` と `07-system-test/README.md:20` にある。
- R-N2 は `03-overview-design/ht-technical-research.md:16` にある。

`ht-inventory-poc.md` の Windows での実行に関する記述（:8、:10-16、:46）と、`07-system-test/README.md:8` の「Windows の Chrome で目視確認」は、私がこれまで見ていない新しい内容である。分割レビューの範囲外として内容は評価しておらず、S1 で扱いを指摘した。

## 確認した結果

### リンク

`docs/` 配下のすべての `.md` について、相対リンクの先にファイルがあるか、`#` 付きのリンク先の見出しがあるかを機械的に確かめた。リンク切れは 0 件だった。

### 確定事項と受入基準の移動先

| 分割前の内容 | 分割後の場所 | 判定 |
| --- | --- | --- |
| 要求、初回の範囲、初回の対象外、決めること | `01-requirements/ht-design.md:5-35` | 欠落なし |
| 初回の環境と構成（Node.js、SDK、配置、Router） | `03-overview-design/ht-technical-research.md:12-16` | 欠落なし |
| 検証コマンド | `07-system-test/README.md:14` | 欠落なし |
| `HtBridge.ping` の契約、疎通画面の構成、CAMERA 権限、拒否の再現手順 | `04-detailed-design/ht-native-and-screen.md:14-16` | 欠落なし |
| ネイティブ疎通の受入表（ブラウザ、エミュレーター、実機、キャンセル、拒否、Web） | `07-system-test/README.md:16-25`。要件側には要約（`01:23-28`）とリンクがある。 | 欠落なし。分割前より「空入力」と「その他の撮影エラー」の 2 行が増えている。 |
| 画面構成、コンポーネントの責務、復元の方針 | `02-basic-design/ht-layout-components.md` | 欠落なし |
| Store の方針、失敗時の扱い、同時取得 | `04-detailed-design/ht-store.md` | 欠落なし |
| API 生成、Zod の扱い、業務 API で決めること | `04-detailed-design/ht-api.md` | 欠落なし |
| 単体・結合・E2E・ネイティブの確認範囲の表 | `05`、`06`、`07` の各 README に 1 行ずつ | 欠落なし |
| 初回の疎通テスト（Web の固定応答、画面テスト、初期ルート、記録項目） | `05:9`、`06:11-12`、`07:29` | 欠落なし |
| 在庫 PoC の M1・M2 の反映（`split`、`msw-fixed`、`webServer`、未定義リクエストのエラー化、`request:start`） | `08-poc/ht-inventory-poc.md:33-38`、`:55` | 欠落なし |

## should

### S1. Windows で実行する内容が、方針確定後に未レビューのまま加わっている

- 場所：`08-poc/ht-inventory-poc.md:8`、`:10-16`、`:46`、`:69-71`、`07-system-test/README.md:8`
- 問題：PoC の実行環境が、WSL2 から「Windows の PowerShell とローカルドライブの作業コピー、Playwright の `channel: 'chrome'`」に変わっている。:46 の AC1 の方法も、Windows の Chrome を使う形に変わった。ところが :69-71 の「方針確定」には、私の同意（M1・M2 の確認）がそのまま記録されている。このため、Windows で実行する内容まで同意済みに見える。この変更は分割とは別の方針変更である。
- 修正案：:69-71 に「この同意は MSW 構成（M1・M2）が対象で、Windows で実行する内容は対象外」と注記する。Windows で実行する内容は、方針の変更として別途レビューに回す。

### S2. テストの役割分担の方針が分割で消えている

- 場所：`05-unit-test/README.md`、`06-integration-test/README.md`、`07-system-test/README.md`
- 問題：分割前の `ht-testing.md` にあった次の 2 つの段落が、どの資料にも残っていない。各フェーズの表は移っているが、フェーズをまたぐ判断基準がなくなった。
  - :11「個別の分岐は Vitest、共通 UI の状態やブラウザ上の局所操作は Storybook、複数画面を通る代表経路は Playwright を主担当とする」
  - :24「業務ルールの細かな入力パターンは単体で網羅し、結合で受け渡しを確かめ、E2E では利用者が完了できる代表経路を確認する」
- 修正案：`docs/README.md` に「テストの役割分担」の節を設け、この 2 文を移す。各テストフェーズの README からそこへリンクする。

### S3. 在庫 PoC の受入基準と、テストフェーズの対応が書かれていない

- 場所：`05-unit-test/README.md:11`、`06-integration-test/README.md:7`、`:14`、`07-system-test/README.md:8`、`:10`
- 問題：3 つのテスト資料は、どれも PoC の受入基準の表へリンクしているだけである。どの AC をどのフェーズで確認するかが書かれていない。特に AC5（同じ仕様から生成物を再生成できること）は、どのテスト資料にも出てこない。
- 修正案：各 README に、担当する AC の ID を明記する。例：05 は AC3・AC4、06 は AC2、07 は AC1。AC5 は生成の確認として、PoC の計画書か 05 のどちらかに担当を書く。

### S4. 作業記録の置き場所が古いまま

- 場所：`03-overview-design/ht-technical-research.md:22`
- 問題：「GitHub の準備前の作業記録は `docs/records/` に置く」とあるが、`docs/records/` はもうない。PoC の計画書は `08-poc/` に移っている。
- 修正案：作業記録の置き場所を、今の構成に合わせて書き直す（例：`08-poc/` または `docs/records/` を作り直す）。あわせて「設計を進める順序」の 1 番目にある「設計レビュー反映後の方針を確認し」は完了済みなので、現在の状態に更新する。

### S5. メッセージ表示の方針が一部消えている

- 場所：`02-basic-design/ht-layout-components.md:23`、`03-overview-design/ht-technical-research.md:5-8`
- 問題：分割前の概要設計の表にあった「メッセージ」の行のうち、次の方針がどの資料にも残っていない。
  - 操作結果や失敗の通知は画面側で表示する。
  - 開発ログはコンソールで確認する。

  表示する状態（読み込み中、結果なし、通信失敗、読み取り失敗）は `02:23` に移っている。
- 修正案：この方針を `03-overview-design` の「アプリ全体の構成」の表に、行として戻す。

## nit

### N1. ファイル名とフェーズの対応

- 場所：`01-requirements/ht-design.md`、`03-overview-design/ht-technical-research.md`
- 修正案：見出しはそれぞれ「要件定義」「概要設計」だが、ファイル名は分割前のままである。`requirements.md`、`overview-design.md` などに改名すると、フェーズと内容が一致する。改名する場合は README、各資料、レビュー記録からのリンクを合わせて直す。

### N2. 基本設計に技術的な記述が残っている

- 場所：`02-basic-design/ht-layout-components.md:7`
- 修正案：「業務画面は Vue Router で切り替え、`RouterView` に表示する」は、README:14 の定義では概要設計で扱う内容である。`03-overview-design` に移す。

### N3. 設計の進め方が概要設計の資料にある

- 場所：`03-overview-design/ht-technical-research.md:20-25`
- 修正案：「設計を進める順序」は、アプリの構成ではなくフェーズ全体の進め方なので、`docs/README.md` へ移す。

### N4. 導入時の確認から Playwright が抜けている

- 場所：`05-unit-test/README.md:7`
- 修正案：分割前の `ht-testing.md:28` は「Vitest、Storybook、Playwright が同じ Vue・Vite 構成で動くか」を確認するとしていた。分割後は Vitest と Storybook だけになっている。Playwright の確認を `07-system-test/README.md` に加える。

## 結論

分割に同意する。must は 0 件で、確定事項と受入基準の取りこぼしは見つからなかった。リンクも 0 件の切れで整合している。S2〜S5 は分割で生じたずれで、反映を推奨する。S1 は分割とは別の、未レビューの方針変更の扱いである。
