# 公開前の資料チェック

対象は、資料インデックス、8フェーズの現行資料、レビュー記録、draw.io、PlantUML。PoC の範囲・受入基準、実行環境、API・Store の責務が資料間で一致するかを確認した。

## must

- 1件。PoC アーキテクチャ図の実行フローが、画面制御 Store から生成 Fetch クライアントへ取得を依頼する形のままになっている。詳細設計では画面からレスポンス Store の取得 action を呼ぶ。図を設計に揃える。[既存の指摘 M1](../../ht-poc-architecture/01-review/README.md) · [Store 設計](../../../04-detailed-design/ht-store.md#画面制御とレスポンス保持)

## 確認結果

- Web PoC は在庫検索まで、Android 疎通・撮影は別作業という範囲は、要件、PoC 計画、図の詳細画面の扱いで一致している。
- Node.js 22 系の下限と Windows・既存 Chrome での PoC 検証方針は現行の参照先で確認できる。Windows での実走は未確認。
- 資料間の相対リンクと見出し、draw.io の図形 ID と矢印参照を検証した。draw.io と PlantUML の描画は未確認。
- 公開候補の `AGENTS.md`、`CLAUDE.md`、`docs/` を、秘密鍵・代表的な認証トークン形式・ローカルホームの絶対パスについて検索し、一致はなかった。

公開先の作成とプッシュに先立ち、GitHub の対象アカウントで有効な認証を確認する。
