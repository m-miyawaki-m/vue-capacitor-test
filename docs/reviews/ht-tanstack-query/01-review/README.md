# TanStack Query と Pinia の使い分けのレビュー

Claude Code（Herdr `w3:p2`）は検討案として同意した。must 0件、should 4件、nit 2件。

| 指摘 | 内容 |
| --- | --- |
| S1 | 現行 PoC と採用候補の区別、判断の時点と担当を明記する。 |
| S2 | Query 採用時に変更する資料と AC3 の確認方法を挙げる。 |
| S3 | サーバーデータを Pinia に複製せず、対応するレスポンス Store を移行後に廃止する旨を明記する。 |
| S4 | 未検索、条件変更、同一条件の再取得を分け、自動再取得と再試行を明記する。 |
| N1・N2 | 結果反映と通信中断を分け、一覧復元の要件からキャッシュ保持時間を決める。 |

[検討資料](../../../04-detailed-design/ht-tanstack-query-pinia.md) · [指摘への対応](../02-response/README.md)
