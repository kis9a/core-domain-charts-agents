# Few-shot（最小例・校正用）

## Example 1: Scorer エージェント

### SYSTEM
あなたは「Scorer」です。次の1件を採点・分類し、JSONで返す。

### USER
```json
[{
  "name":"billing-adapter",
  "purpose":"決済ゲートウェイへの接続と課金イベントの転送",
  "custom_logic_thickness":"thin",
  "integration_points":["stripe-sdk","ledger"],
  "metrics":{"change_freq":12,"dep_in":5,"dep_out":9,"loc":2100},
  "evidence":["billing/adapter.ts","ADR-006-billing-integration.md"],
  "uncertainties":[]
}]
```

### ASSISTANT Expected
```json
[{
  "name":"billing-adapter",
  "scores":{"S":3,"Δ":2,"X":5},
  "classification":"Generic",
  "rationale":"外部SaaS依存が強くコモディティ性が高い。カスタム薄い。",
  "risks_or_unknowns":[]
}]
```

---

## Example 2: Analyzer エージェント

### SYSTEM
あなたは「Analyzer」です。以下の入力から各サブドメイン候補を要約。

### USER
```json
{
  "modules": [
    {
      "name": "inventory-tracker",
      "loc": 8500,
      "change_freq": 156,
      "dep_in": 12,
      "dep_out": 5,
      "owners": ["warehouse-team@corp"],
      "top_terms": ["stock","quantity","threshold","alert"],
      "readme_excerpt": "Real-time inventory tracking with ML-based demand forecasting",
      "issue_titles": ["Improve forecast accuracy", "Add multi-warehouse support"],
      "public_api": ["GET /v1/inventory/status", "POST /v1/inventory/forecast"],
      "evidence_fragments": ["inventory/forecast.py: train_model()", "inventory/alerts.py: check_thresholds()"]
    }
  ],
  "dependencies": [
    {"from":"inventory-tracker","to":"ml-platform","weight":8},
    {"from":"order-service","to":"inventory-tracker","weight":15}
  ]
}
```

### ASSISTANT Expected
```json
[{
  "name": "inventory-tracker",
  "purpose": "在庫量のリアルタイム追跡と需要予測による最適化",
  "capabilities": [
    "リアルタイム在庫監視",
    "機械学習による需要予測",
    "閾値ベースのアラート",
    "複数倉庫対応"
  ],
  "integration_points": ["ml-platform", "order-service"],
  "custom_logic_thickness": "thick",
  "externalization_notes": "ML予測部分は外部化可能だが、在庫管理ルールは事業固有",
  "evidence": [
    "inventory/forecast.py: train_model()",
    "inventory/alerts.py: check_thresholds()",
    "README: ML-based demand forecasting"
  ],
  "uncertainties": [
    "ML予測精度の実績データが不明",
    "複数倉庫対応の実装状況"
  ]
}]
```

---

## Example 3: 完全な処理フロー

### INPUT
```json
{
  "modules": [
    {
      "name": "recommendation-engine",
      "loc": 15000,
      "change_freq": 89,
      "dep_in": 25,
      "dep_out": 3,
      "top_terms": ["personalization","ranking","collaborative-filtering","embedding"],
      "evidence_fragments": ["reco/model.py: train_cf()", "reco/ranker.py: personalized_sort()"]
    },
    {
      "name": "email-notifier",
      "loc": 3000,
      "change_freq": 8,
      "dep_in": 2,
      "dep_out": 12,
      "top_terms": ["sendgrid","template","campaign"],
      "evidence_fragments": ["notifier/sender.py: send_batch()"]
    }
  ]
}
```

### EXPECTED OUTPUT
```json
{
  "items": [
    {
      "name": "recommendation-engine",
      "purpose": "パーソナライズされた商品推薦による購買体験向上",
      "capabilities": ["協調フィルタリング", "個別ランキング", "埋め込み生成"],
      "integration_points": ["user-service", "catalog-service"],
      "custom_logic_thickness": "thick",
      "scores": {"S": 5, "Δ": 5, "X": 2},
      "classification": "Core",
      "change_freq": 89,
      "evidence": ["reco/model.py: train_cf()", "reco/ranker.py: personalized_sort()"],
      "uncertainties": [],
      "rationale": "推薦アルゴリズムは競争優位性の源泉。独自実装が厚く外部化困難。"
    },
    {
      "name": "email-notifier",
      "purpose": "メール通知の送信管理",
      "capabilities": ["バッチ送信", "テンプレート管理"],
      "integration_points": ["sendgrid", "user-service"],
      "custom_logic_thickness": "thin",
      "scores": {"S": 2, "Δ": 1, "X": 5},
      "classification": "Generic",
      "change_freq": 8,
      "evidence": ["notifier/sender.py: send_batch()"],
      "uncertainties": [],
      "rationale": "SendGrid依存のラッパー。差別化要素なく完全に外部化可能。"
    }
  ],
  "chart": {
    "items": [
      {
        "name": "recommendation-engine",
        "S": 5,
        "Δ": 5,
        "X": 2,
        "classification": "Core",
        "change_freq": 89
      },
      {
        "name": "email-notifier",
        "S": 2,
        "Δ": 1,
        "X": 5,
        "classification": "Generic",
        "change_freq": 8
      }
    ],
    "vegaLite": {
      "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
      "data": {"name": "items"},
      "mark": {"type": "circle"},
      "encoding": {
        "x": {"field": "X", "type": "quantitative", "scale": {"domain": [0, 5]}, "title": "Externalizability (X)"},
        "y": {"field": "S", "type": "quantitative", "scale": {"domain": [0, 5]}, "title": "Strategic Importance (S)"},
        "size": {"field": "change_freq", "type": "quantitative", "title": "Change Frequency"},
        "color": {"field": "classification", "type": "nominal"}
      }
    }
  }
}
```