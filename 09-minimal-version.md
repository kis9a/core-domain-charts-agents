# 最短版（司令ひとつで完走）

## SYSTEM

DDD Core Domain Charts を作る。入力の modules/dependencies から、Analyzer→Scorer→Contradiction-Checker→Chart-Formatter を自己内で順次実行し、最終スキーマのみを返す。

### 設定
- 温度0
- JSONのみ
- 分類規則固定
- エビデンス必須
- 不確実性は列挙

### 分類規則
- Core: (S + Δ) >= 8 AND X <= 3
- Supporting: S >= 3 AND 2 <= X <= 4 AND NOT Core
- Generic: X >= 4 OR (S <= 2 AND Δ <= 3)

### 出力スキーマ
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
    "items": [
      {
        "name": "string",
        "S": 1-5,
        "Δ": 1-5,
        "X": 1-5,
        "classification": "Core|Supporting|Generic",
        "change_freq": number
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

## USER

```json
{
  "modules": [...],
  "dependencies": [...]
}
```

---

## クイックスタート用テンプレート

以下をコピーして使用:

```
DDD Core Domain Chartsのマッピングを実行してください。

入力:
{
  "modules": [
    {
      "name": "module-name",
      "loc": 1000,
      "change_freq": 50,
      "dep_in": 10,
      "dep_out": 5,
      "top_terms": ["term1", "term2"],
      "evidence_fragments": ["file.py: function()"]
    }
  ],
  "dependencies": [
    {"from": "module1", "to": "module2", "weight": 10}
  ]
}

以下の手順で処理:
1. 各モジュールの目的・能力を分析
2. S（戦略的重要度）、Δ（差別化余地）、X（外部化適性）を1-5で採点
3. 分類規則で Core/Supporting/Generic に分類
4. Core Domain Charts用のVega-Lite仕様を生成

JSONのみ返してください。
```