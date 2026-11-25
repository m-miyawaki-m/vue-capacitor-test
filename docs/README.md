# Android HT アプリの設計資料

## 最初に見る場所

- **何を作るか確認する**。下の [フェーズ別の資料](#フェーズ別の資料) から「要件定義」へ進む。PoC の範囲、着手条件、受入基準は「在庫管理 PoC 計画」を見る。
- **設計や技術手順を調べる**。画面・遷移、全体構成、API・Store の仕様はフェーズ別の資料から探す。自動生成やテストの具体的な手順は [技術手順インデックス](#技術手順インデックス) を見る。
- **レビューの進み具合を確認する**。[レビュー記録表](reviews/README.md#レビュー記録表) で、レビュー済み・対応済み・再確認済みの範囲と残件を見る。

## 実装に向けた決定状況

決定済みは方針や受入基準が資料に定義されている状態。実装・動作確認の完了とは別に扱う。方針ありは方向性が決まっているもの、未決は判断が必要なもの、後続は今回の範囲外。要確認は環境や実走で確かめるもの。

| 作るもの | 現時点の判断 | 参照先 |
| --- | --- | --- |
| 在庫検索の Web PoC | 検証範囲、サンプル API、モック、合格条件は定義済み。Git、Windows 環境、Web 基盤の準備が残る。 | [Web PoC の決定状況](08-poc/ht-inventory-poc.md#web-poc-の決定状況) |
| Android 疎通・撮影 | Java プラグインと撮影の合格条件は定義済み。Android のビルド・検証環境を具体化する。 | [開発基盤と環境](03-overview-design/ht-technical-research.md#開発基盤と環境の決定状況) · [実装仕様](04-detailed-design/ht-native-and-screen.md#実装仕様の決定状況) |
| 入庫・在庫管理の業務アプリ | 対象業務と共通構成は整理済み。各業務の操作・項目、実 API、読み取り方式には未決事項がある。 | [要件](01-requirements/ht-design.md#要件の決定状況) · [画面と操作](02-basic-design/ht-layout-components.md#画面と操作の決定状況) · [実装仕様](04-detailed-design/ht-native-and-screen.md#実装仕様の決定状況) |

## フェーズ間で決定事項を引き継ぐ流れ

上から順に読む。枠内はその段階で決めること・確かめること、矢印は次の段階に渡す内容を示す。PoC の結果やテストで見つかった差分は、該当する設計や受入基準に戻して見直す。

[フェーズごとの検討と引き継ぎ（PlantUML）](ht-phase-sequence.puml)

**在庫検索 PoC** は、業務アプリ全体に先立ち、検索1本で「画面 → API → モック応答 → Store → 表示」を試す。設計上の仮説を選び、実際の生成・実装・テストで確かめてから、結果を概要設計・詳細設計・テスト方針へ反映する。 [在庫管理 PoC 計画](08-poc/ht-inventory-poc.md)

## 資料を修正したら

1. 該当フェーズの資料を修正する。複数の資料に影響する場合は、それぞれの本文と参照先を確認する。
2. このページの概要とリンクを見直す。資料名・配置・見出しや要約が変わった場合は、このページと参照元を更新し、リンクが開くことを確認する。
3. レビューを行う場合は、[レビュー記録表](reviews/README.md#レビュー記録表) に対象と結果を記録する。指摘に対応したら対応内容へリンクし、再レビューの結果を受け取った時点で再確認状況と残件を更新する。レビュー後に変更した範囲は、再確認するまで未レビューとして扱う。

## 技術手順インデックス

| 知りたいこと | 概要 |
| --- | --- |
| [Orval・Zod・Faker・MSW の自動生成](04-detailed-design/ht-api.md#在庫-poc-の自動生成手順) | `open.yaml` と Orval 設定を用意し、`npm run api:generate` で Fetch クライアント、型、Zod スキーマ、モックを生成する。 |
| [テスト用の値と応答の差し替え](06-integration-test/README.md#msw-によるテストデータの切り替え) | MSW で固定2件、空配列、HTTP 500、Zod 検証に失敗する本文を返す。再取得に失敗しても前回の結果が残ることを Vitest で確認する。 |
| [Vitest と Storybook の連携](05-unit-test/README.md#vitest-と-storybook-の連携手順) | Storybook の Vitest アドオンを設定し、`args` で部品の初期状態、`play` で操作と期待結果を指定する。 |
| [Vitest でのレスポンス Store 単体テスト](05-unit-test/README.md#レスポンス-store-の単体テスト) | テストごとに Pinia を作り、API を差し替えて取得 action と保存状態を確認する。HTTP 判定・Zod 検証の単体観点も参照できる。 |
| [Storybook での UI 結合テスト](06-integration-test/README.md#storybook-で確認する-ui-コンポーネント間の連携) | 検索フォームとフッターを組み合わせ、操作と親へのイベント通知を確認する。Pinia が必要な場合の設定方針も参照できる。 |
| [Playwright の業務シナリオ](08-poc/ht-inventory-poc.md#受入基準とテスト) | Windows の Chrome でメニューから在庫検索を実行し、GET のクエリと固定2件の結果表示を確認する。 |

## フェーズ別の資料

| フェーズ | 資料 |
| --- | --- |
| 01 要件定義 | [対象業務、初回範囲、受入基準](01-requirements/ht-design.md) |
| 02 基本設計 | [レイアウトとコンポーネント](02-basic-design/ht-layout-components.md) · [画面・遷移図](02-basic-design/ht-screen-layout.drawio) |
| 03 概要設計 | [構成と技術方針](03-overview-design/ht-technical-research.md) |
| 04 詳細設計 | [API 生成と実行時検証](04-detailed-design/ht-api.md) · [Pinia の状態管理](04-detailed-design/ht-store.md) · [画面とネイティブ疎通](04-detailed-design/ht-native-and-screen.md) |
| 05 単体テスト | [ロジック・Store・コンポーネントの確認](05-unit-test/README.md) |
| 06 結合テスト | [画面・Store・API 境界の連携確認](06-integration-test/README.md) |
| 07 総合テスト | [ブラウザ E2E と Android 受入確認](07-system-test/README.md) |
| 08 PoC | [在庫管理 PoC 計画](08-poc/ht-inventory-poc.md) · [アーキテクチャ・Store の再利用・検証フロー](08-poc/ht-inventory-poc-architecture.drawio) |

基本設計では利用者から見た画面・操作・遷移を扱い、概要設計ではアプリ全体の構成と技術方針を扱う。


レビュー記録のファイル名と行番号はレビュー当時の資料を指す。
