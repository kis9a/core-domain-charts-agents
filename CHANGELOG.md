# 変更履歴

すべての重要な変更はこのファイルに記録されます。

このプロジェクトは[セマンティックバージョニング](https://semver.org/lang/ja/)に準拠しています。

## [Unreleased]

### 追加
- エラーハンドリング層の実装 (`10-error-handler.md`)
  - 共通エラーコード体系
  - リトライポリシー
  - 自動復旧戦略

- 設定の外部化 (`config.yaml`)
  - 環境別設定サポート
  - スコアリング基準のカスタマイズ
  - フィーチャーフラグ

- 監視・可観測性の向上 (`11-monitoring.md`)
  - メトリクス設計
  - 分散トレーシング
  - ダッシュボード設計
  - SLI/SLO定義

- テストフレームワークの構築
  - テストランナー仕様 (`test-cases/test-runner.md`)
  - ゴールデンテスト (`test-cases/golden-test.json`)
  - エッジケース集 (`test-cases/edge-cases.json`)

- ドキュメントの改善
  - アーキテクチャドキュメント (`ARCHITECTURE.md`)
  - トラブルシューティングガイド (`TROUBLESHOOTING.md`)
  - 変更履歴 (`CHANGELOG.md`)

### 改善
- エラー処理の堅牢性向上
- パフォーマンス最適化の仕組み
- テスタビリティの向上

## [1.0.0] - 2025-01-03

### 追加
- 初期リリース
- 基本エージェント実装
  - Orchestrator (`00-orchestrator.md`, `orchestrator.md`)
  - Analyzer (`01-analyzer.md`)
  - Scorer (`02-scorer.md`)
  - Contradiction Checker (`03-contradiction-checker.md`)
  - Chart Formatter (`04-chart-formatter.md`)

- 統合版エージェント
  - All-in-One (`05-all-in-one.md`)
  - Minimal Version (`09-minimal-version.md`)

- 補助ファイル
  - JSONスキーマ定義 (`06-json-schema.md`)
  - Few-shot例 (`07-few-shot-example.md`)
  - ガードレール (`08-guardrails.md`)

### 機能
- DDDのCore Domain Chartsマッピング
- S（戦略的重要度）、Δ（差別化余地）、X（外部化適性）スコアリング
- Core/Supporting/Generic分類
- Vega-Lite形式でのチャート生成

## [0.9.0] - 2024-12-15 (Beta)

### 追加
- プロトタイプ実装
- 基本的な分析機能
- シンプルなスコアリング

### 既知の問題
- エラーハンドリングが不十分
- テストカバレッジが低い
- ドキュメントが不完全

---

## バージョニング規則

### メジャーバージョン (X.0.0)
- 破壊的変更
- APIの大幅な変更
- 出力フォーマットの非互換変更

### マイナーバージョン (0.X.0)
- 後方互換性のある機能追加
- 新しいエージェントの追加
- パフォーマンス改善

### パッチバージョン (0.0.X)
- バグ修正
- セキュリティ修正
- ドキュメント修正

## アップグレードガイド

### 0.9.0 から 1.0.0 へ

**破壊的変更**:
- なし（初回安定版リリース）

**推奨事項**:
1. 新しいJSONスキーマに従った入力検証を実装
2. エラーコード体系を採用
3. config.yamlによる設定管理に移行

### 1.0.0 から次期バージョンへ

**準備中の機能**:
- GraphQL API サポート
- リアルタイム分析
- カスタムスコアラー拡張
- マルチ言語対応

## サポートポリシー

| バージョン | サポート状態 | サポート終了日 |
|-----------|------------|--------------|
| 1.0.x | ✅ Active | 2026-01-03 |
| 0.9.x | ⚠️ Limited | 2025-06-03 |
| < 0.9 | ❌ EOL | - |

**Active**: フル機能サポート、バグ修正、セキュリティ更新
**Limited**: セキュリティ更新のみ
**EOL**: サポート終了

## コントリビューター

- [@kis9a](https://github.com/kis9a) - 初期実装とアーキテクチャ設計
- AI Assistant - ドキュメント整備と改善提案

## ライセンス

このプロジェクトはプロジェクトライセンスに従います。

## リンク

- [GitHub リポジトリ](https://github.com/kis9a/core-domain-charts-agents)
- [Issue トラッカー](https://github.com/kis9a/core-domain-charts-agents/issues)
- [ドキュメント](./README.md)