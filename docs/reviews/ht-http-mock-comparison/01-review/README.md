# HTTP モックとテスト手段の比較レビュー

Claude Code（Herdr `w3:p2`）は比較と変更案に同意した。must 0件、should 3件、nit 2件。

| 指摘 | 対応 |
| --- | --- |
| S1 | `setupServer` は MSW の API と明記した。 |
| S2 | 手書きの固定データを生成型と Zod で確認し、パスと実際の送信内容の整合を結合テストで確認する案を記載した。 |
| S3 | 採用時の起動モード、Playwright、Orval 設定、資料の更新範囲を明記した。 |
| N1・N2 | ブラウザ版 MSW の起動主体を明記し、AC4 の固定 props を維持する旨を追記した。 |

現行 PoC の構成変更と実走は未実施。

[比較資料](../../../06-integration-test/ht-http-mock-comparison.md)
