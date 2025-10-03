# DDD Core Domain Charts マッピングエージェント

このディレクトリには、コードリポジトリのメタ情報からDDD（Domain-Driven Design）のCore Domain Chartsを生成するためのAIエージェント定義が含まれています。

## 概要

これらのエージェントは、ソフトウェアプロジェクトのモジュール構造を分析し、各サブドメインを以下の3つのカテゴリに分類します：

- **Core Domain**: ビジネスの競争優位性の源泉となる中核領域
- **Supporting Domain**: ビジネスに必要だが差別化要素ではない支援領域
- **Generic Domain**: 汎用的で外部サービスで代替可能な領域

## エージェント一覧

### 基本エージェント

1. **[00-orchestrator.md](./00-orchestrator.md)**
   - 全体統括と共通規約を定義
   - 他のエージェントの連携を管理

2. **[01-analyzer.md](./01-analyzer.md)**
   - サブドメイン候補の抽出と要約
   - ビジネス目的と技術的特性の分析

3. **[02-scorer.md](./02-scorer.md)**
   - S（戦略的重要度）、Δ（差別化余地）、X（外部化適性）の採点
   - Core/Supporting/Genericへの分類

4. **[03-contradiction-checker.md](./03-contradiction-checker.md)**
   - 分析結果の矛盾検出と自己修正
   - 一貫性の確保

5. **[04-chart-formatter.md](./04-chart-formatter.md)**
   - Vega-Lite形式でのチャート生成
   - 可視化用データの整形

### 統合・補助

6. **[05-all-in-one.md](./05-all-in-one.md)**
   - 全エージェントを統合した一括実行版
   - ワンショットで完全な分析を実行

7. **[06-json-schema.md](./06-json-schema.md)**
   - 出力JSONのバリデーション用スキーマ
   - 形式の妥当性検証

8. **[07-few-shot-example.md](./07-few-shot-example.md)**
   - 各エージェントの入出力例
   - 学習・デバッグ用サンプル

9. **[08-guardrails.md](./08-guardrails.md)**
   - 出力の安定性を保証するルール集
   - エラー防止とフォーマット遵守

10. **[09-minimal-version.md](./09-minimal-version.md)**
    - 最短版のプロンプト
    - クイックスタート用

## 使用方法

### 基本的な使い方

1. **個別エージェントの連鎖実行**
   ```
   Analyzer → Scorer → Contradiction-Checker → Chart-Formatter
   ```

2. **一括実行（推奨）**
   - `05-all-in-one.md` のプロンプトを使用
   - 入力データを準備してワンショットで実行

### 入力データ形式

```json
{
  "modules": [
    {
      "name": "module-name",
      "loc": 1234,
      "change_freq": 42,
      "dep_in": 18,
      "dep_out": 3,
      "top_terms": ["term1", "term2"],
      "evidence_fragments": ["file.py: function()"]
    }
  ],
  "dependencies": [
    {"from": "module1", "to": "module2", "weight": 10}
  ]
}
```

### 出力データ形式

```json
{
  "items": [
    {
      "name": "string",
      "purpose": "string",
      "capabilities": ["string"],
      "integration_points": ["string"],
      "custom_logic_thickness": "thin|medium|thick",
      "scores": {"S": 1-5, "Δ": 1-5, "X": 1-5},
      "classification": "Core|Supporting|Generic",
      "change_freq": number,
      "evidence": ["string"],
      "uncertainties": ["string"],
      "rationale": "string"
    }
  ],
  "chart": {
    "items": [...],
    "vegaLite": {...}
  }
}
```

## 分類ルール

固定の分類規則：
- **Core**: (S + Δ) >= 8 AND X <= 3
- **Supporting**: S >= 3 AND 2 <= X <= 4 AND NOT Core
- **Generic**: X >= 4 OR (S <= 2 AND Δ <= 3)

## スコアリング基準

### S（Strategic Importance）戦略的重要度
- 5: 直接KPI/収益/競争力の中核
- 3: 重要だが代替可能
- 1: 補助的/内部運用中心

### Δ（Differentiation Potential）差別化余地
- 5: 独自アルゴリズム/データで差別化
- 3: 実装工夫で改善余地あり
- 1: 定型的でコモディティ

### X（Externalizability）外部化適性
- 5: SaaS/OSSでほぼ代替可能
- 3: 半分外部化可能
- 1: 外部化困難（独自ロジック厚い）

## 推奨設定

AIモデル設定：
- temperature: 0.0
- top_p: 1
- presence_penalty: 0
- frequency_penalty: 0
- response_format: json_object

## ライセンス

このプロジェクトのライセンスに従います。