# JSONスキーマ（バリデーション用）

## JSONスキーマ定義

Core Domain Charts マッピング出力の妥当性検証用JSONスキーマです。

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DDD Core Domain Charts Mapping",
  "type": "object",
  "required": ["items"],
  "properties": {
    "items": {
      "type": "array",
      "items": { "$ref": "#/definitions/Item" }
    },
    "chart": {
      "type": "object",
      "required": ["items", "vegaLite"],
      "properties": {
        "items": {
          "type": "array",
          "items": { "$ref": "#/definitions/ChartItem" }
        },
        "vegaLite": {
          "type": "object",
          "required": ["$schema", "data", "mark", "encoding"],
          "properties": {
            "$schema": { "type": "string" },
            "data": { "type": "object" },
            "mark": { "type": "object" },
            "encoding": { "type": "object" }
          }
        }
      }
    }
  },
  "definitions": {
    "Item": {
      "type": "object",
      "required": [
        "name",
        "purpose",
        "capabilities",
        "integration_points",
        "custom_logic_thickness",
        "scores",
        "classification",
        "change_freq",
        "evidence",
        "uncertainties",
        "rationale"
      ],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1
        },
        "purpose": {
          "type": "string",
          "minLength": 1
        },
        "capabilities": {
          "type": "array",
          "items": { "type": "string" },
          "minItems": 1
        },
        "integration_points": {
          "type": "array",
          "items": { "type": "string" }
        },
        "custom_logic_thickness": {
          "type": "string",
          "enum": ["thin", "medium", "thick"]
        },
        "scores": {
          "type": "object",
          "required": ["S", "Δ", "X"],
          "properties": {
            "S": {
              "type": "integer",
              "minimum": 1,
              "maximum": 5
            },
            "Δ": {
              "type": "integer",
              "minimum": 1,
              "maximum": 5
            },
            "X": {
              "type": "integer",
              "minimum": 1,
              "maximum": 5
            }
          },
          "additionalProperties": false
        },
        "classification": {
          "type": "string",
          "enum": ["Core", "Supporting", "Generic"]
        },
        "change_freq": {
          "type": "number",
          "minimum": 0
        },
        "evidence": {
          "type": "array",
          "items": { "type": "string" },
          "minItems": 1
        },
        "uncertainties": {
          "type": "array",
          "items": { "type": "string" }
        },
        "rationale": {
          "type": "string"
        }
      },
      "additionalProperties": false
    },
    "ChartItem": {
      "type": "object",
      "required": ["name", "S", "Δ", "X", "classification", "change_freq"],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1
        },
        "S": {
          "type": "integer",
          "minimum": 1,
          "maximum": 5
        },
        "Δ": {
          "type": "integer",
          "minimum": 1,
          "maximum": 5
        },
        "X": {
          "type": "integer",
          "minimum": 1,
          "maximum": 5
        },
        "classification": {
          "type": "string",
          "enum": ["Core", "Supporting", "Generic"]
        },
        "change_freq": {
          "type": "number",
          "minimum": 0
        }
      },
      "additionalProperties": false
    }
  }
}
```

## 使用方法

このスキーマを使用して、生成されたJSONの妥当性を検証できます。

### Node.js例

```javascript
const Ajv = require("ajv");
const ajv = new Ajv();
const schema = require("./schema.json");
const validate = ajv.compile(schema);

const data = { /* generated JSON */ };
const valid = validate(data);

if (!valid) {
  console.error(validate.errors);
}
```

### Python例

```python
import json
import jsonschema

with open("schema.json") as f:
    schema = json.load(f)

data = { # generated JSON }

try:
    jsonschema.validate(instance=data, schema=schema)
    print("Valid JSON")
except jsonschema.exceptions.ValidationError as e:
    print(f"Invalid JSON: {e.message}")
```