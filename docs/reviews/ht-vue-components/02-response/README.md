# コンポーネント実装方針への指摘対応

- M1・S4。アプリの `RouterView` が業務画面を描画し、各画面が `MainLayout` に本文とフッターを渡す。画面は Store と部品をつなぐ。基本設計と draw.io に反映した。
- S1・S3・N1。Storybook の開閉を named model に揃え、`VApp` を使う前提と、空白入力・正規化を確認する例を追加した。
- S2。表示専用の PoC カードには結果配列内のインデックスを使用し、詳細を作る際に API の一意 ID を決める。
- N2・N3。画面制御 Store の観点を具体化し、パネルの初期状態は HT での操作感を踏まえて見直す旨を追記した。

[初回レビュー](../01-review/README.md) · [コンポーネント実装方針](../../../04-detailed-design/ht-vue-components.md)
