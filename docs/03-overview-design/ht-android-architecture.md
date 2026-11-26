# Android HT アプリのアーキテクチャ

[アプリのレイヤー・呼び出し経路・環境別の確認範囲（draw.io）](ht-android-architecture.drawio)

## 構成方針

Android は Capacitor 8 の WebView で Vue 3・Vuetify 4 のアプリを実行する。業務機能を組み込む段階では、画面・画面遷移・状態管理・業務 API の呼び出しコードを Web と Android で共用する。Java の担当は、Capacitor の橋渡しを通じて Android 固有の機能を提供すること。

| 層 | 担当 | 配置 | 初回 Android 疎通での扱い |
| --- | --- | --- | --- |
| 業務画面 | Vue Router、MainLayout、入力・表示・画面遷移 | `src/views/`、`src/layouts/`、`src/components/` | 疎通確認画面のみ。業務画面は Web PoC で別途確認する。 |
| 状態・業務 API | Pinia で画面制御と取得結果を保持する。Orval・Zod を使う API 呼び出しコードは Web と Android で共用する。 | `src/stores/`、`src/api/` | 対象外。Web PoC で別途確認する。 |
| 端末機能の呼び出し | TypeScript の型付きプラグイン契約と Web 実装。カメラの成功・失敗を画面で扱える形にする。 | `src/native/` | 対象。 |
| Capacitor・Android | 初期 Activity が WebView をホストする。独自 Java プラグインを登録し、公式 Camera プラグインを利用する。 | ルートの Capacitor 設定、`android/` | 対象。 |

[Capacitor の Android カスタムコード](https://capacitorjs.com/docs/android/custom-code) · [Camera プラグイン](https://github.com/ionic-team/capacitor-camera)

## 初回の2つの呼び出し経路

| 画面操作 | Web | Android |
| --- | --- | --- |
| 疎通確認 | Vue → `HtBridge.ping` の Web 実装 → 固定応答を画面と開発者コンソールに表示 | Vue → Capacitor ブリッジ → アプリ内の Java プラグイン → 固定応答・画面表示。Java の到達は Logcat で確認 |
| 撮影 | Android 実機で確認する機能として画面に案内する | Vue → 公式 `@capacitor/camera` の `takePhoto` → 撮影結果の `webPath` を画面に表示。キャンセル・権限拒否・その他の失敗を区別する |

`HtBridge.ping` は、Vue から Java まで呼び出して固定応答が戻ることを確認するため、設計上で付けた仮のプラグイン名・メソッド名。Capacitor やメーカー SDK が提供する API ではない。

初回の独自 Java プラグインはアプリ内のローカルプラグインとし、Capacitor 経由で Java に到達することを確かめる疎通確認専用とする。Android の Activity には登録処理を置き、プラグインが入力検証・ログ出力・応答を担当する。撮影確認は公式 Camera の経路で行う。入力・応答・権限・エラーコードは詳細設計に従う。

[初回の画面とプラグインの契約](../04-detailed-design/ht-native-and-screen.md#初回の疎通画面とプラグイン) · [Android の受入確認](../07-system-test/README.md#初回のネイティブ受入確認)

## 機能ごとの立証範囲

| 機能 | 単体で立証すること | 結合で立証すること | Android で立証すること |
| --- | --- | --- | --- |
| 独自プラグインの疎通 | Vitest で `HtBridge` の Web 実装を直接呼ぶ。有効な入力で固定応答を返し、空入力を `INVALID_MESSAGE` で拒否する。Java 側は JUnit で入力検証と応答生成を確認する。 | Vue の疎通画面と型付き呼び出しをつなぐ。プラグインの応答を差し替え、押下時の引数、成功・失敗の表示を確認する。 | エミュレーターで APK を起動し、画面操作が Capacitor ブリッジ経由で Java に届くことを応答と Logcat で確認する。空入力で Java が返すエラーも確認する。 |
| 撮影成功 | Vitest でアプリ側の撮影処理を確認する。公式 Camera の応答を差し替え、返された `webPath` を表示用の値として受け取る。 | 撮影ボタンからアプリ側の撮影処理までつなぐ。固定 `webPath` の応答で画像表示が更新されることを確認する。 | 実機でカメラが起動し、撮影画像をアプリ内で表示できることを確認する。 |
| 撮影のキャンセル・拒否・失敗 | Vitest でエラーコードからキャンセル、権限拒否、その他の失敗を分類できることを確認する。 | Camera の応答を各エラーに差し替える。キャンセルは前回画像を維持し、権限拒否とその他の失敗では対応する案内を表示する。 | 実機で撮影をキャンセルし、権限を拒否して、元の画面への復帰と表示を確認する。 |
| Web での撮影操作 | Web・Android の判定を差し替え、Web では Camera の呼び出しを行わないことを確認する。 | ブラウザ想定で「撮影」を押すと、実機で確認する機能である旨を表示する。 | Web 用の挙動としてブラウザで確認する。 |

単体・結合テストはアプリ側のロジックと画面連携を立証する。Capacitor のネイティブ登録、Java 実装への到達、実際のカメラ画像と端末権限は Android で確認する。テストの詳細は各フェーズの資料に記載する。

[単体の確認項目](../05-unit-test/README.md#android-疎通と撮影の単体テスト) · [結合の確認項目](../06-integration-test/README.md#初回の疎通画面) · [実機・エミュレーターの受入確認](../07-system-test/README.md#初回のネイティブ受入確認)

## ビルドと検証の境界

`npm run build` で Web アセットを生成し、Capacitor 設定の `webDir` を生成先に合わせる。`npx cap sync android` で Web アセットとプラグインを Android プロジェクトに反映し、Gradle で APK を作る。ブラウザの Web PoC と Android の疎通・撮影確認は別の受入範囲とする。配布用アセットの確認では Web PoC の開発用モックが含まれないことを確認する。

[Capacitor の開発ワークフロー](https://capacitorjs.com/docs/basics/workflow) · [在庫検索の Web PoC](../08-poc/ht-inventory-poc.md)

## VS Code とエミュレーターでの開発

Android Studio を使わず、VS Code の統合ターミナルから JDK・Android SDK のコマンドラインツール・Gradle Wrapper・エミュレーター・`adb` を使う想定。実行ホストを決めたら、同じホスト側で SDK、仮想端末、ビルドと `adb` を揃えて確認する。

1. SDK の CLI で必要なプラットフォーム、Platform-Tools、エミュレーター、システムイメージを用意し、`sdkmanager --licenses` でライセンスを確認する。仮想端末を作る前に、実行ホストで `emulator -accel-check` を実行して Windows の WHPX や Linux の KVM が利用できるか確認する。
2. Web アセットをビルドし、`npx cap sync android` で同期する。`android/` の Gradle Wrapper で `testDebugUnitTest` と `assembleDebug` を実行する。
3. `emulator` で仮想端末を起動する。`adb devices` で接続を確認し、`adb install -r` で APK を入れて起動する。
4. 画面から疎通確認を操作し、Java の入力検証・固定応答が実際に動くことを確認する。画面への戻り値と、統合ターミナルの `adb logcat` で観測する。

Android 用の Capacitor プラグインは、プラグインを組み込んでインストールした Android アプリを起動して実行する。Chrome では Web 対応の実装を使う。エミュレーターでも Android 用プラグインは実行でき、Android API や仮想端末が提供する機能は設定に応じて確認できる。メーカー SDK が対象端末のサービスや実際の付属機器を必要とする場合は、対象 HT と付属機器の組み合わせで業務動作を確認する。

Logcat には Android のログが蓄積される。疎通確認専用のログは、Java 側で `Log.d("HtBridge", "ping received")` のようにタグ付きで出す。VS Code の統合ターミナルで `adb devices` に対象が表示されることを確かめ、`adb logcat -s HtBridge:D` を実行してから画面のボタンを押す。複数端末がある場合は `adb -s <端末ID> logcat -s HtBridge:D` で対象を指定する。

JUnit は Java の入力検証と応答生成を自動で判定する。Capacitor の登録や WebView からの呼び出しが通ることは、APK を起動して画面の応答と Logcat で確認する。

[Android の Logcat コマンド](https://developer.android.com/tools/logcat)

[Android SDK のコマンドラインツール](https://developer.android.com/tools) · [Android のコマンドラインビルド](https://developer.android.com/build/building-cmdline)

## 付属機器の利用経路

端末内蔵カメラは公式 Capacitor Camera で扱う。付属機器は、選定したメーカー SDK を Android 側で呼び出す構成を想定する。Vue からは付属機器用の独自 Capacitor プラグインを呼び、Java 側が SDK の初期化・操作・結果やエラーの受け渡しを担当する。機器・SDK・接続方式と、プラグインの具体的な操作は後続で決める。

`Vue の業務画面 → 付属機器用 Capacitor プラグイン → Android 側のメーカー SDK → 付属機器`

## 後続で決めること

- Android で業務 API に接続する際のベース URL、認証、CORS と開発時の接続先を決める。Web PoC の現在の origin による相対 URL の解決は、Web の開発・テスト環境で使う。Android 側では WebView の origin と API の origin を区別する。
- 対象 HT の WebView の実際のバージョンを確認し、Capacitor 8 の動作条件、Vite のビルド対象、Vuetify の表示と操作が合うか検証する。
- 付属機器の種類、接続方式、利用するメーカー SDK、バーコード・QR・OCR の対応範囲。SDK の選定後に Java プラグインの契約と実機での確認項目を決める。
- 撮影中に Android がアプリのプロセスを破棄した場合の復元。業務で撮影結果を保持する要件がある場合は、Capacitor App の `appRestoredResult` を使った復元と、結果の保存・破棄の条件を設計する。
- Android SDK・Gradle・adb を実行するホスト、アプリ ID、配布・署名の方針はセットアップや配布の段階で決める。

[Capacitor App の復元結果](https://capacitorjs.com/docs/apis/app#addlistenerapprestoredresult-)
