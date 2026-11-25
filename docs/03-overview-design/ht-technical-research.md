# Android HT アプリの概要設計

## アプリ全体の構成

| 項目 | 現時点の方針 | 次に決めること |
| --- | --- | --- |
| 状態管理 | Pinia を使い、画面制御 Store とレスポンス Store を分ける。取得済みレスポンスを画面間で再利用する。 | 保持する範囲（本文、ヘッダー、ステータス）、検索条件ごとの保持、再取得・破棄の条件 |
| 通信 | `open.yaml` を仕様の原本とし、Orval で通信クライアント、型、Zod スキーマを生成する。 | API の利用範囲、モックの扱い、通信エラーと検証エラーの扱い |

VS Code を開発環境とする。TypeScript・Vite を使い、初回から Vue Router で画面を表示する。業務画面は後続でルートを追加する。VS Code の推奨拡張と業務用ライブラリは導入時に決める。

## 初回の環境と構成

- Node.js 22 LTS（22.22.2 以上の 22 系）、npm、`package-lock.json` を採用する。Vue 3、Vuetify 4、Vue Router、Vite 8、create-vue のテンプレートが採用する TypeScript、Capacitor 8、`@capacitor/camera` 8.1 以上の 8 系を採用し、具体版は初期化時にロックする。
- Web と Android を対象とする。Android は Capacitor 8 のテンプレートに従い minSdk 24、compileSdk・targetSdk 36、JDK 21 を使う。VS Code を開発環境とし、Android Studio を利用する場合は 2025.2.1 以降とし、利用の有無やエミュレーターの構成はネイティブ検証準備時に決める。
- 初回の構成は `src/` にルーター設定、疎通画面、プラグインの型・Web 実装、`android/` に Android プロジェクトと Java 実装、リポジトリルートに Capacitor 設定を置く。Vue Router は `createWebHashHistory` を使い、初期ルート `/` に疎通画面を割り当てる。未定義ルートは `/` に戻す。

[Vue の初期セットアップ](https://vuejs.org/guide/quick-start.html) · [Vite 8 のガイド](https://v8.vite.dev/guide/) · [Capacitor 8 の移行ガイド](https://capacitorjs.com/docs/updating/8-0) · [Capacitor の環境要件](https://capacitorjs.com/docs/getting-started/environment-setup)

## 設計を進める順序

1. 設計レビュー反映後の方針を確認し、初回の疎通画面を設計する。GitHub の準備前の作業記録は `docs/records/` に置く。
2. 後続で対象業務を一つ選び、検索・一覧・詳細と関連遷移の範囲を決める。
3. 画面図と照合し、画面一覧・遷移・操作・表示項目を確定する。
4. 対象機能の部品、Store、API 契約とテストケースを具体化する。

## レビュー記録

[設計レビューと対応の経過](../reviews/README.md#設計書)

## 開発基盤と環境の決定状況

| 決めること | 状況 | 現時点の内容 | 残っている判断・作業 |
| --- | --- | --- | --- |
| Web・Android の技術構成 | 決定済み | Vue 3、Vuetify 4、Vue Router、Vite 8、TypeScript、Capacitor 8。 | 実際に導入する依存の版を固定し、互換性を確認する。 |
| Node.js・パッケージ管理 | 決定済み | Node.js 22 LTS の 22.22.2 以上、npm、`package-lock.json`。 | Windows 側のバージョンを確認し、必要に応じて切り替える。 |
| Git と作業コピー | 未決 | Web 基盤を先に統合し、PoC はそのコミットから作る。Windows と WSL2 は同じコミットを使う方針。 | Git の初期化と既定ブランチ、ローカルで始めるか GitHub を起点にするかを決める。 |
| Web PoC の実行環境 | 方針あり | WSL2 で編集、Windows PowerShell で起動・テスト。既存の Windows Chrome を使う。 | Windows 側の作業コピー、Node.js・Chrome の実動確認と方針のレビュー。 |
| Android ビルド・起動環境 | 方針あり | Capacitor 8 の SDK レベルと JDK 21 を使う。 | Gradle・Android SDK・adb を動かす環境、エミュレーターと撮影用実機を決める。 |
| メーカー SDK | 後続 | 初回の Java プラグインは呼び出しの疎通まで。撮影は公式 Camera を使う。 | 実業務の読み取り方式が決まった段階で SDK の利用要否を決める。 |
