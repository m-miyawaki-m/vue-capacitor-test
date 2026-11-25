# 総合テスト

| 段階 | 業務の確認 | フレームワーク連携の確認 | 主な手段 |
| --- | --- | --- | --- |
| E2E | 業務メニューから入庫または在庫管理を開き、検索 → 一覧 → 詳細 → 戻るを通せる。戻ったときに検索条件と結果が復元され、関連業務一覧から選択した業務へ進める。 | ブラウザで起動したアプリを画面操作し、Router の遷移、MainLayout のフッター操作、Store をまたぐ状態復元を観察する。外部 API を差し替えるケースでは、アプリの起動から操作完了までを確認する。 | Playwright |
| ネイティブ受入確認 | アプリから Android 端末のカメラを起動して撮影し、結果を受け取れる。 | Java プラグインを含めてビルドでき、画面から Capacitor ブリッジを経由して Java 側に到達したことをログで確認する。公式 Camera プラグインは実機で確認する。 | Android エミュレーター・Android 実機 |

在庫 PoC では Windows の Chrome で目視確認し、Playwright で業務メニューから在庫検索、リクエスト、結果カード表示までを確認する。実装対象が広がった時点で、入庫登録や在庫更新のケースを追加する。

[在庫 PoC の受入基準](../08-poc/ht-inventory-poc.md#受入基準とテスト) · [Playwright の API モック](https://playwright.dev/docs/mock)

## 初回のネイティブ受入確認

ビルドとテストには `npm run lint`、`npm run type-check`、`npm run test:unit -- --run`、`npm run build`、`npx cap sync android`、Android ディレクトリで `./gradlew assembleDebug` を実行する。

| 操作 | 期待結果 |
| --- | --- |
| ブラウザで「疎通確認」を押す | 画面と開発者コンソールに `{ value: 'pong', platform: 'web' }` が出る |
| Android エミュレーターで「疎通確認」を押す | Java 実装を含むビルドが成功し、画面に `{ value: 'pong', platform: 'android' }` が出る。`adb logcat -s HtBridge` で `ping received` を確認できる |
| Android エミュレーターで入力を空にして「疎通確認」を押す | Java 実装が `INVALID_MESSAGE` を返し、再試行の案内が表示される |
| Android 実機で「撮影」を押して撮影する | 公式 Camera の `takePhoto` が返した `webPath` で撮影画像を表示できる |
| 撮影をキャンセルする（`OS-PLUG-CAMR-0006`） | 元の画面に戻り、既存の画像を保持する。キャンセルの通知は表示しない |
| カメラ使用が拒否される（`OS-PLUG-CAMR-0003`） | 撮影できない旨と、端末設定でカメラの利用可否を確認する案内を表示する。再操作できる |
| その他の撮影エラー | 撮影できない旨と再試行の案内を表示する |
| Web で「撮影」を押す | Android 実機で確認する機能である旨を表示する |

[Capacitor Camera 公式プラグイン](https://github.com/ionic-team/capacitor-camera)

- 手動の受入確認は、上の表に沿ってブラウザ、Android エミュレーター、Android 実機を使う。Java 側の不正入力はエミュレーターで空入力を送って確認し、カメラ権限拒否は実機で確認する。権限を拒否する手順はプラグイン詳細設計を参照する。実測結果は日付、環境（機種または仮想端末、OS・API レベル）、ビルド種別、操作、期待結果、実測・合否、必要なログを記録する。

[撮影権限の宣言と拒否の再現手順](../04-detailed-design/ht-native-and-screen.md#初回の疎通画面とプラグイン)
