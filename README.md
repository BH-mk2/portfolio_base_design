# カメラ用SDK 基本設計書作成プロジェクト

C社製カメラを制御する「カメラ用SDK」を題材に、実装時に迷わない基本設計書を自力で作成するための学習・伴走ワークスペースです。

## ディレクトリ構成

- **`README.md`**: このファイル（進捗管理・目次）
- **`docs/templates/`**: 各セッションの解説・書き方ガイド・テンプレート
- **`docs/my_design/`**: あなたが実際に記述する設計書（ここを更新していきます）

---

## 学習ロードマップと進捗

各セッションのリンクをクリックして、解説の確認と執筆を進めましょう。

### [x] Session 1: 土台作りと全体像
- **対象項目**: 機能概要、ユースケース一覧
- **学習ガイド**: [Session 1 ガイド (session1_guide.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/templates/session1_guide.md)
- **あなたの設計書**: [Session 1 設計書 (session1.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/my_design/session1.md)
- **設計判断の問い**: 「なぜこの機能範囲にするのか？」「なぜこのユースケースが必要なのか？」

### [x] Session 2: インターフェース設計
- **対象項目**: API一覧、API入出力設計
- **学習ガイド**: [Session 2 ガイド (session2_guide.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/templates/session2_guide.md)
- **あなたの設計書**: [Session 2 設計書 (session2.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/my_design/session2.md)
- **設計判断の問い**: 「なぜこのAPI単位・粒度にしたのか？」

### [x] Session 3: 動的挙動の設計
- **対象項目**: シーケンス図、状態遷移図
- **学習ガイド**: [Session 3 ガイド (session3_guide.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/templates/session3_guide.md)
- **あなたの設計書**: [Session 3 設計書 (session3.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/my_design/session3.md)
- **設計判断の問い**: 「どこを同期/非同期処理にしたのか？」「なぜその状態遷移になるのか？」

### [/] Session 4: 堅牢性の設計
- **対象項目**: エラー設計、ログ設計、スレッド・排他制御方針
- **学習ガイド**: [Session 4 ガイド (session4_guide.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/templates/session4_guide.md)
- **あなたの設計書**: [Session 4 ドラフト (session4_draft.md)](file:///e:/workspace/job_change/portfolio_base_design/docs/my_design/session4_draft.md)
- **設計判断の問い**: 「どの異常系を考慮したのか？」「なぜその排他制御方式にするのか？」

### [ ] Session 5: 品質とテスト
- **対象項目**: 非機能要件・性能要件、テスト観点
- **設計判断の問い**: 「性能や保守性・拡張性をどう考えたのか？」「実装時に迷わない粒度になっているか？」
