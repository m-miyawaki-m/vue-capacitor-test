# 設計書 再レビュー記録（ht-design.md と参照設計書）

- レビュー日：2026-11-25
- レビュー担当：Claude Code（Herdr w3:p2）
- 依頼元：Codex（Herdr w3:p3）
- 対象：
  - `docs/ht-design.md`（最新方針）
  - `docs/reviews/ht-design-review.md`（前回の指摘）
  - `docs/reviews/ht-design-review-response.md`（対応表）
  - 参照資料：`docs/ht-technical-research.md`、`docs/ht-layout-components.md`、`docs/ht-store.md`、`docs/ht-testing.md`、`docs/ht-api.md`、`docs/ht-screen-layout.drawio`
- 重点：M1〜M4 が解消したか、初回から Vue Router を導入する変更が資料間で整合しているか。
- 結論：**条件付きで同意する**。新規の must 1 件（R-M1）を反映すれば「方針に同意」とする。
- 件数（新規・残存の指摘）：must 1 件、should 4 件、nit 4 件。ほかに後続で扱う項目が 7 件ある。

## 確認に使った情報

npm レジストリで 2026-11-25 時点の公開版を確認し、`@capacitor/camera@8.2.6` のパッケージの中身を読んだ。

| パッケージ | latest |
| --- | --- |
| `@capacitor/core` | 8.6.2 |
| `@capacitor/camera` | 8.2.6 |
| `vuetify` | 4.3.4 |
| `vite` | 8.3.8 |
| `vitest` | 5.1.2（peer の vite は `^6.4.0 \|\| ^7.0.0 \|\| ^8.0.0`） |
| `typescript` | 7.0.2 |
| `vue-tsc` | 3.3.14（peer の typescript は `>=5.0.0`） |

`@capacitor/camera@8.2.6` について、次の 4 点を確認した。

- `takePhoto(options): Promise<MediaResult>` が存在する（`dist/esm/definitions.d.ts:17`）。`getPhoto` は非推奨になっている（同 :83）。
- キャンセル時のエラーコードは `TakePhotoCancelled = "OS-PLUG-CAMR-0006"`、権限拒否は `CameraPermissionDenied = "OS-PLUG-CAMR-0003"`（同 :797、:809）。
- Android の `takePhoto` は、`AndroidManifest.xml` に CAMERA 権限が宣言されている場合だけ権限を要求する（`IonCameraFlow.kt:941-945`、コメント "if the manifest does not contain the camera permissions key, we don't need to ask the user"）。
- README の Android の節には「The Camera plugin requires no permissions, unless using `saveToGallery: true`」とある。

Capacitor 8 の環境要件（minSdk 24、compileSdk・targetSdk 36、JDK 21、Android Studio 2025.2.1 以降）は、公式ドキュメントを開いて照合していない。私の把握している内容とは矛盾しない。

## 前回の must の確認

| 指摘 | 判定 | 根拠 |
| --- | --- | --- |
| M1 初回の範囲と対象外 | 解消 | `ht-design.md:19-25` に範囲と対象外がある。「決めること」(:51-56) から初回の範囲が消えている。進める順序 (:85-88) も初回と後続に分かれている。 |
| M2 バージョンと構成 | 解消 | `ht-design.md:27-32` に Node.js 24 LTS、npm、ロックファイル、各メジャーバージョン、Android の SDK レベル、JDK、Android Studio、配置先、検証コマンドがある。公開版との矛盾もない。残る点は R-S2 と R-N1。 |
| M3 独自プラグインの受入基準 | 解消 | `ht-design.md:38`、`:42-43` に契約、固定入力、固定応答、`INVALID_MESSAGE`、ログタグ、確認コマンドがある。Java 側の不正入力の確認は R-S3。 |
| M4 Camera の受入基準 | 一部解消 | `ht-design.md:44-47` に成功・キャンセル・拒否・Web がそろい、`takePhoto` と `webPath` も実在する。ただし、拒否のケースは今の前提では発生しない（R-M1）。 |

## 前回の should と nit の確認

| 指摘 | 判定 | 備考 |
| --- | --- | --- |
| S1 | 解消 | `ht-layout-components.md:15`、`:21` と図の 2 ページ目 :109、:157 に将来案と注記されている。 |
| S2 | 解消 | `ht-design.md:62`、`ht-layout-components.md:13`、`ht-store.md:11`、図の 1 ページ目 :77 が「復元する」で揃っている。 |
| S3 | 解消 | `ht-store.md:19-23`。通信の中断は後続（L-2）。 |
| S4 | 後続へ移した（妥当） | `ht-api.md:15-17` で業務 API の詳細設計項目として明記されている（L-1）。 |
| S5 | 解消 | `ht-api.md:11`。 |
| S6 | 解消 | `ht-testing.md:26-30`。 |
| S7 | 解消 | `ht-design.md:64`、`ht-api.md:7`。 |
| S8 | 一部解消 | 記録先は `docs/records/` に決まった（`ht-design.md:85`）。Git の初期化は未定（R-S4）。 |
| N1 | 一部解消 | 置き場所はリポジトリルートに決まった（`ht-api.md:5`）。ファイル名は `open.yaml` のまま（L-7）。 |
| N2・N4・N5 | 解消 | 図の 1 ページ目のフッターは「検索」「読み取り」、詳細画面のフッターは削除、3 ページ目の ID は `related*` に改名されている。 |
| N3 | ほぼ解消 | カードの形になったが、注記に「行をタップ」が残っている（R-N3）。 |
| N6 | ほぼ解消 | `applyScan`・`selectRelated` の矢印が追加された。遷移先の配置に問題が残る（R-N4）。 |
| N7・N8 | 解消 | `ht-technical-research.md:10`、`ht-design.md:55`。 |

## Vue Router を初回から導入する変更の整合

整合している。

- `ht-design.md:21`（初回の範囲）、`:29`（採用技術）、`:31`（初期ルート `/` に疎通画面）、`:67`（後続でルートを追加）が互いに矛盾しない。
- `ht-layout-components.md:7`（業務画面は `RouterView` に表示）、`ht-testing.md:29`（`/` から疎通画面が開くことを確認）、`ht-testing.md:9` と `:21`（後続の Playwright で Router の遷移を確認）も同じ前提に立っている。
- 対応表の最終行（`ht-design-review-response.md:16`）とも一致している。
- 細かな点は R-N2（履歴モードと未定義ルート）と R-S2（疎通画面に MainLayout を使うか）に記載した。

## 新規・残存の指摘

### must

#### R-M1. 権限拒否の受入基準が、今の前提では発生しない

- 場所：`ht-design.md:46`、`:31`、`ht-testing.md:28`
- 問題：`@capacitor/camera` 8.2.6 の Android 版 `takePhoto` は、`AndroidManifest.xml` に `android.permission.CAMERA` が宣言されている場合だけ実行時の権限を要求する（`IonCameraFlow.kt:941-945`）。Capacitor のテンプレートには CAMERA 権限の宣言がなく、README も「権限は不要」としている。このままでは「カメラ使用が拒否される」ケースを実機で再現できず、受入基準を判定できない。
- 修正案：次のどちらかを方針に明記する。
  - (a) CAMERA 権限を宣言する。`android/app/src/main/AndroidManifest.xml` に `<uses-permission android:name="android.permission.CAMERA" />` を追加し、再現手順を書く。手順は「初回起動時の権限ダイアログで拒否する」と「端末設定でアプリのカメラ権限を取り消してから撮影する」の 2 つ。拒否は `CameraPermissionDenied`（`OS-PLUG-CAMR-0003`）で判定する。
  - (b) 権限を宣言しない。その場合は「拒否」の行を受入基準から外し、「権限を宣言しないため拒否は発生しない」と記録する。

  CLAUDE.md の「権限が拒否された場合を確認する」に合わせるなら (a) を推奨する。

### should

#### R-S1. キャンセルと拒否の判定方法を、エラーコードで決める

- 場所：`ht-design.md:45-46`、`ht-testing.md:28`
- 問題：「キャンセルなら通知しない」「拒否なら案内を出す」「それ以外は失敗として表示」を分けるには、`takePhoto` の reject を区別する必要がある。今はその方法が書かれていない。また、その他の失敗（`NO_CAMERA_AVAILABLE` など）を画面でどう表示するかも決まっていない。
- 修正案：`error.code` の `OS-PLUG-CAMR-0006`（キャンセル）と `OS-PLUG-CAMR-0003`（拒否）で分岐し、それ以外は「撮影に失敗した。再試行できる」と表示する、と明記する。Vitest の画面テストでも、この 3 つのコードの差し替えを確認ケースにする。

#### R-S2. 初回の疎通画面の構成と、Vuetify を初回に入れるかが決まっていない

- 場所：`ht-design.md:21`、`:29`、`:85`
- 問題：:29 は Vuetify 4 を初回の採用技術に含めている。一方、:21 の初回の範囲には UI ライブラリの導入が書かれていない。疎通画面に置く要素（「疎通確認」ボタン、「撮影」ボタン、応答の表示欄、画像の表示欄、メッセージ欄）と、`MainLayout` を使うかどうかも決まっていない。このままでは実装範囲がぶれる。
- 修正案：初回で Vuetify を導入して疎通画面に使うのか、導入を後続に回すのかを決める。あわせて疎通画面の要素の一覧を :38-47 の近くに追記する。`MainLayout` は初回の対象外とするのが妥当と考える。

#### R-S3. Java 側の `INVALID_MESSAGE` を確認する手段がない

- 場所：`ht-design.md:38`、`ht-testing.md:28`
- 問題：ボタンは常に `ping` を渡すため、画面からの手動確認では空のメッセージを送れない。Vitest で確認できるのは Web 実装だけになる。Java 実装の拒否が「定義したが、どこでも確認されない」状態になっている。
- 修正案：次のいずれかを選んで明記する。
  - (a) Java の単体テスト（JUnit）で `INVALID_MESSAGE` を確認する。
  - (b) 開発用の入力欄やボタンで空文字を送り、エミュレーターで手動確認する。
  - (c) Java 側の確認を初回の対象外と明記する。

#### R-S4. 実装を始める前の前提（Git、Android の実行環境）が未定

- 場所：`ht-design.md:3`、`:30`、`:32`、`:54`、`ht-design-review-response.md:12`
- 問題：次の 2 点が決まっていない。
  - CLAUDE.md の開発フローでは、既定ブランチから作業ブランチを作って PR を作る。ところが Git の初期化を「実装準備時に決める」としており、初回の実装に着手する条件がそろっていない。
  - 初回の検証には `./gradlew assembleDebug` とエミュレーターでの疎通確認が含まれる。しかし Android SDK とエミュレーターをどこで動かすか（WSL か Windows か）が決まっていない。WSL2 の上ではエミュレーターの起動が難しいことが多く、Windows 側のエミュレーターと WSL 側の `adb` を組み合わせる構成も考えられる。
- 修正案：初回の実装の前に、次の 2 点を「初回の環境と構成」に追記する。
  - Git の初期化と既定ブランチの名前（GitHub のリポジトリを作るかどうかを含む）。
  - Android SDK・Gradle ビルド・エミュレーター・`adb` を、それぞれ WSL と Windows のどちらで動かすか。

### nit

#### R-N1. 「TypeScript 5」の固定と最新版の関係

- 場所：`ht-design.md:29`
- 問題：TypeScript の latest は 7.0.2 である。「TypeScript 5」と固定する理由（`vue-tsc` や create-vue のテンプレートとの互換性など）が書かれていない。
- 修正案：create-vue のテンプレートが入れる版に従う、と書くか、5 系に固定する理由を一言添える。

#### R-N2. Router の履歴モードと未定義ルートの扱い

- 場所：`ht-design.md:31`
- 問題：`createWebHistory` と `createWebHashHistory` のどちらを使うかが書かれていない。未定義のパスに来たときの扱いも書かれていない。
- 修正案：履歴モードを明記し、未定義のパスは `/` へリダイレクトする、などを決める。初回はルートが 1 つだけなので、どちらでも実害はない。

#### R-N3. 図の注記の「行をタップ」

- 場所：`ht-screen-layout.drawio` 1 ページ目 `flow`（:74）
- 問題：結果がカードの形になったのに、注記は「行をタップ」のままになっている。
- 修正案：「カードをタップ」に直す。

#### R-N4. 関連業務の遷移先が一覧画面の枠の中に描かれている

- 場所：`ht-screen-layout.drawio` 3 ページ目 `relatedDestination`（:207）
- 問題：「選択した業務画面」が関連業務一覧の画面枠（`relatedFrame`）の中に置かれていて、一覧画面の一部のように見える。
- 修正案：枠の外に別の画面として描き、`selectRelated` の矢印をそちらへつなぐ。

## 後続で扱う項目（初回の対象外）

| ID | 内容 | 扱う時期 | 根拠 |
| --- | --- | --- | --- |
| L-1 | API のエラー種類ごとの表示・再操作・エラー本文の検証 | 業務 API の詳細設計 | `ht-api.md:15-17` |
| L-2 | 通信の中断（`AbortController`）を使うかどうか | API 導入時 | `ht-store.md:23` |
| L-3 | Orval の実行時検証と `safeParse` のどちらで検証するか | API 導入時 | `ht-api.md:9` |
| L-4 | 読み取り画面の構成、認識方式、メーカー SDK | SDK の選定後 | `ht-layout-components.md:21` |
| L-5 | 業務画面、業務 Store、Storybook、Playwright の導入 | 対象業務の決定後 | `ht-design.md:25`、`:86-88` |
| L-6 | Store の永続化、在庫更新後の無効化のルール | 業務実装時 | `ht-store.md:13`、`:27` |
| L-7 | 仕様ファイルの名前 `open.yaml`（一般的には `openapi.yaml`）の確認 | API 導入時 | `ht-api.md:5` |

## 結論

条件付きで同意する。R-M1 を反映すれば、初回の方針として「方針に同意」とする。R-S1〜R-S4 は実装の着手前に反映することを推奨する。R-N1〜R-N4 と L-1〜L-7 は、方針の確定を妨げない。
