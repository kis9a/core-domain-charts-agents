# テストランナー仕様

## 概要

DDD Core Domain Chartsエージェントのテストフレームワークです。単体テスト、統合テスト、回帰テスト、ゴールデンテストをサポートします。

## テスト戦略

### 1. テストピラミッド

```
         /\         E2E Tests (5%)
        /  \        - 完全なエンドツーエンドシナリオ
       /    \
      /──────\      Integration Tests (20%)
     /        \     - エージェント間連携
    /          \    - 外部サービス統合
   /────────────\
  /              \  Unit Tests (75%)
 /                \ - 個別エージェントロジック
/──────────────────\- バリデーション
```

## テスト実行

### 基本コマンド

```bash
# 全テスト実行
npm test

# 特定のテストスイート実行
npm test -- --suite=unit
npm test -- --suite=integration
npm test -- --suite=e2e

# 特定のエージェントのテスト
npm test -- --agent=analyzer
npm test -- --agent=scorer

# カバレッジ付き実行
npm test -- --coverage

# ウォッチモード
npm test -- --watch
```

## テスト設定

### test-config.yaml

```yaml
test_runner:
  parallel: true
  timeout: 30000
  retries: 2
  bail: false

coverage:
  enabled: true
  thresholds:
    statements: 80
    branches: 75
    functions: 80
    lines: 80

environments:
  - name: unit
    setup: ./setup/unit.js
    teardown: ./teardown/unit.js

  - name: integration
    setup: ./setup/integration.js
    teardown: ./teardown/integration.js
    services:
      - mock_api
      - test_db

  - name: e2e
    setup: ./setup/e2e.js
    teardown: ./teardown/e2e.js
    real_services: true

reporting:
  formats:
    - console
    - junit
    - html
    - json
  output_dir: ./test-results/
```

## テストヘルパー

### TestHelper クラス

```javascript
class TestHelper {
  /**
   * テスト用の入力データを生成
   */
  static generateInput(options = {}) {
    const defaults = {
      moduleCount: 5,
      withDependencies: true,
      withEvidence: true,
      validJson: true
    };

    const config = { ...defaults, ...options };

    return {
      modules: Array.from({ length: config.moduleCount }, (_, i) => ({
        name: `module-${i}`,
        loc: Math.floor(Math.random() * 10000),
        change_freq: Math.floor(Math.random() * 100),
        dep_in: Math.floor(Math.random() * 20),
        dep_out: Math.floor(Math.random() * 10),
        top_terms: [`term-${i}-1`, `term-${i}-2`],
        evidence_fragments: config.withEvidence
          ? [`file-${i}.py: function_${i}()`]
          : []
      })),
      dependencies: config.withDependencies
        ? this.generateDependencies(config.moduleCount)
        : []
    };
  }

  /**
   * 期待される出力を検証
   */
  static validateOutput(output, schema) {
    const ajv = new Ajv();
    const validate = ajv.compile(schema);
    const valid = validate(output);

    if (!valid) {
      throw new ValidationError(validate.errors);
    }

    return true;
  }

  /**
   * スコアリング規則を検証
   */
  static validateClassification(item) {
    const { S, Δ, X } = item.scores;
    const { classification } = item;

    // Core: (S + Δ) >= 8 AND X <= 3
    if ((S + Δ) >= 8 && X <= 3) {
      expect(classification).toBe('Core');
    }
    // Supporting: S >= 3 AND 2 <= X <= 4 AND NOT Core
    else if (S >= 3 && 2 <= X && X <= 4) {
      expect(classification).toBe('Supporting');
    }
    // Generic: X >= 4 OR (S <= 2 AND Δ <= 3)
    else if (X >= 4 || (S <= 2 && Δ <= 3)) {
      expect(classification).toBe('Generic');
    }

    return true;
  }

  /**
   * パフォーマンスを測定
   */
  static async measurePerformance(fn, iterations = 10) {
    const times = [];

    for (let i = 0; i < iterations; i++) {
      const start = process.hrtime.bigint();
      await fn();
      const end = process.hrtime.bigint();
      times.push(Number(end - start) / 1e6); // ミリ秒に変換
    }

    return {
      mean: times.reduce((a, b) => a + b) / times.length,
      median: times.sort()[Math.floor(times.length / 2)],
      min: Math.min(...times),
      max: Math.max(...times),
      p95: times.sort()[Math.floor(times.length * 0.95)]
    };
  }
}
```

## モック実装

### MockAgent

```javascript
class MockAgent {
  constructor(name, responses = {}) {
    this.name = name;
    this.responses = responses;
    this.callHistory = [];
  }

  async process(input) {
    this.callHistory.push({
      timestamp: new Date(),
      input: JSON.parse(JSON.stringify(input))
    });

    // レスポンスの選択
    const responseKey = this.getResponseKey(input);
    const response = this.responses[responseKey] || this.responses.default;

    if (!response) {
      throw new Error(`No mock response defined for ${responseKey}`);
    }

    // エラーシミュレーション
    if (response.error) {
      throw new Error(response.error);
    }

    // 遅延シミュレーション
    if (response.delay) {
      await new Promise(resolve => setTimeout(resolve, response.delay));
    }

    return response.data;
  }

  getResponseKey(input) {
    // 入力に基づいてレスポンスキーを決定
    return input.modules ? `modules_${input.modules.length}` : 'default';
  }

  // テスト用アサーション
  expectCalled(times) {
    expect(this.callHistory.length).toBe(times);
  }

  expectCalledWith(input) {
    const lastCall = this.callHistory[this.callHistory.length - 1];
    expect(lastCall.input).toEqual(input);
  }

  reset() {
    this.callHistory = [];
  }
}
```

## アサーションライブラリ

### カスタムマッチャー

```javascript
expect.extend({
  /**
   * スコアが有効範囲内かチェック
   */
  toHaveValidScores(received) {
    const pass = ['S', 'Δ', 'X'].every(key => {
      const score = received.scores[key];
      return Number.isInteger(score) && score >= 1 && score <= 5;
    });

    return {
      pass,
      message: () =>
        pass
          ? `Expected scores not to be valid`
          : `Expected scores to be valid (1-5 integers), got ${JSON.stringify(
              received.scores
            )}`
    };
  },

  /**
   * 分類が正しいかチェック
   */
  toHaveCorrectClassification(received) {
    const { S, Δ, X } = received.scores;
    const { classification } = received;

    let expected;
    if ((S + Δ) >= 8 && X <= 3) {
      expected = 'Core';
    } else if (S >= 3 && 2 <= X && X <= 4) {
      expected = 'Supporting';
    } else {
      expected = 'Generic';
    }

    const pass = classification === expected;

    return {
      pass,
      message: () =>
        pass
          ? `Expected classification not to be ${expected}`
          : `Expected classification to be ${expected}, got ${classification}`
    };
  },

  /**
   * 証拠が十分かチェック
   */
  toHaveSufficientEvidence(received, minCount = 1) {
    const pass = received.evidence && received.evidence.length >= minCount;

    return {
      pass,
      message: () =>
        pass
          ? `Expected insufficient evidence`
          : `Expected at least ${minCount} evidence items, got ${
              received.evidence ? received.evidence.length : 0
            }`
    };
  }
});
```

## フィクスチャ管理

### fixtures/index.js

```javascript
const fixtures = {
  // 正常系のテストデータ
  validInput: {
    simple: require('./valid/simple.json'),
    complex: require('./valid/complex.json'),
    large: require('./valid/large.json')
  },

  // 異常系のテストデータ
  invalidInput: {
    malformedJson: require('./invalid/malformed.json'),
    missingFields: require('./invalid/missing-fields.json'),
    outOfRange: require('./invalid/out-of-range.json')
  },

  // エッジケース
  edgeCases: {
    emptyModules: require('./edge/empty-modules.json'),
    circularDependencies: require('./edge/circular-deps.json'),
    maxValues: require('./edge/max-values.json')
  },

  // ゴールデンテスト用
  golden: {
    input: require('./golden/input.json'),
    expectedOutput: require('./golden/expected-output.json')
  }
};

module.exports = fixtures;
```

## スナップショットテスト

### snapshot.test.js

```javascript
describe('Snapshot Tests', () => {
  test('Analyzer output matches snapshot', async () => {
    const input = fixtures.validInput.simple;
    const output = await analyzer.process(input);
    expect(output).toMatchSnapshot();
  });

  test('Scorer output matches snapshot', async () => {
    const input = fixtures.validInput.complex;
    const output = await scorer.process(input);
    expect(output).toMatchSnapshot();
  });

  test('Full pipeline output matches snapshot', async () => {
    const input = fixtures.golden.input;
    const output = await pipeline.process(input);
    expect(output).toMatchInlineSnapshot(`
      {
        "items": [...],
        "chart": {
          "items": [...],
          "vegaLite": {...}
        }
      }
    `);
  });
});
```

## プロパティベーステスト

### property-based.test.js

```javascript
const fc = require('fast-check');

describe('Property-based Tests', () => {
  test('All scores are within valid range', () => {
    fc.assert(
      fc.property(
        fc.record({
          S: fc.integer({ min: 1, max: 5 }),
          Δ: fc.integer({ min: 1, max: 5 }),
          X: fc.integer({ min: 1, max: 5 })
        }),
        scores => {
          const result = classifier.classify(scores);
          expect(['Core', 'Supporting', 'Generic']).toContain(result);
        }
      )
    );
  });

  test('Classification is deterministic', () => {
    fc.assert(
      fc.property(
        fc.record({
          S: fc.integer({ min: 1, max: 5 }),
          Δ: fc.integer({ min: 1, max: 5 }),
          X: fc.integer({ min: 1, max: 5 })
        }),
        scores => {
          const result1 = classifier.classify(scores);
          const result2 = classifier.classify(scores);
          expect(result1).toBe(result2);
        }
      )
    );
  });
});
```

## CI/CD統合

### .github/workflows/test.yml

```yaml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install dependencies
        run: npm ci

      - name: Run linters
        run: npm run lint

      - name: Run unit tests
        run: npm test -- --suite=unit --coverage

      - name: Run integration tests
        run: npm test -- --suite=integration

      - name: Run E2E tests
        run: npm test -- --suite=e2e

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-umbrella

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test-results/
```

## レポート生成

### generate-report.js

```javascript
const generateTestReport = (results) => {
  const report = {
    summary: {
      total: results.numTotalTests,
      passed: results.numPassedTests,
      failed: results.numFailedTests,
      skipped: results.numPendingTests,
      duration: results.testResults.reduce(
        (sum, r) => sum + r.perfStats.runtime,
        0
      )
    },
    coverage: results.coverageMap ? {
      statements: results.coverageMap.getCoverageSummary().statements.pct,
      branches: results.coverageMap.getCoverageSummary().branches.pct,
      functions: results.coverageMap.getCoverageSummary().functions.pct,
      lines: results.coverageMap.getCoverageSummary().lines.pct
    } : null,
    failures: results.testResults
      .filter(r => r.numFailingTests > 0)
      .map(r => ({
        file: r.testFilePath,
        failures: r.testResults
          .filter(t => t.status === 'failed')
          .map(t => ({
            name: t.title,
            message: t.failureMessages.join('\n')
          }))
      }))
  };

  return report;
};
```