# TanStack Query 導入判断の再レビュー

Claude Code（Herdr `w3:p2`）が、現行 PoC の Pinia 継続、AC3 の条件別保持との整合、前回指摘への対応を確認し、must 0件で同意した。追加の should 1件と nit 2件は、独立した API 結果の鮮度・再取得を並行管理する要件がないこと、再検討条件、Store 設計とインデックスからの参照に反映した。要件の変更や実装は行っていない。

[検討資料](../../../04-detailed-design/ht-tanstack-query-pinia.md) · [前回の指摘対応](../02-response/README.md)
