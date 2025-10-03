# トラブルシューティングガイド

## よくある問題と解決方法

### 1. エラーメッセージ別対処法

#### E1001: 入力JSONが不正です

**症状**:
```json
{
  "error": "E1001",
  "message": "入力JSONが不正です"
}
```

**原因**:
- JSON構文エラー
- 不正な文字エンコーディング
- BOMの存在

**解決方法**:
1. JSONリンターで構文をチェック
```bash
# jqを使用した検証
cat input.json | jq '.' > /dev/null
```

2. UTF-8エンコーディングに変換
```bash
iconv -f ISO-8859-1 -t UTF-8 input.json > input_utf8.json
```

3. BOMを削除
```bash
sed '1s/^\xEF\xBB\xBF//' input.json > input_nobom.json
```

---

#### E1002: 必須フィールドが不足しています

**症状**:
```json
{
  "error": "E1002",
  "message": "必須フィールドが不足しています: modules"
}
```

**原因**:
- 必須フィールドの欠落
- フィールド名のタイポ

**解決方法**:
```json
// 最小限の有効な入力
{
  "modules": [
    {
      "name": "module-name",
      "loc": 1000,
      "change_freq": 50,
      "dep_in": 10,
      "dep_out": 5,
      "top_terms": ["term1"],
      "evidence_fragments": ["file.py: func()"]
    }
  ],
  "dependencies": []
}
```

---

#### E2001: スコア値が範囲外です

**症状**:
```json
{
  "error": "E2001",
  "message": "スコア値が範囲外です: S=6"
}
```

**原因**:
- スコアが1-5の範囲外
- 数値型でない値

**自動修正**:
- システムが自動的に範囲内に補正（6→5、0→1）
- ログに補正内容を記録

**予防策**:
- 入力検証を強化
- スコアリングロジックのレビュー

---

#### E4002: タイムアウト

**症状**:
```json
{
  "error": "E4002",
  "message": "タイムアウト: analyzer"
}
```

**原因**:
- LLM APIの応答遅延
- ネットワーク問題
- 過大な入力データ

**解決方法**:

1. タイムアウト値を調整
```yaml
# config.yaml
agents:
  analyzer:
    timeout: 30000  # 30秒に増加
```

2. 入力データを分割
```javascript
// 大規模データを分割処理
const chunks = splitIntoChunks(modules, 20);
const results = await Promise.all(
  chunks.map(chunk => processChunk(chunk))
);
```

3. リトライを実行
```bash
# CLIでリトライ
ddd-map analyze input.json --retry 3
```

---

### 2. パフォーマンス問題

#### 処理が遅い

**診断手順**:

1. ボトルネックの特定
```bash
# 各エージェントの処理時間を確認
curl -X GET http://localhost:9090/metrics | grep agent_processing_time
```

2. メトリクスの確認
```sql
-- 最も時間がかかっているエージェント
SELECT agent_name, AVG(processing_time_ms) as avg_time
FROM agent_metrics
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY agent_name
ORDER BY avg_time DESC;
```

**最適化方法**:

1. **キャッシュの有効化**
```yaml
cache:
  enabled: true
  ttl: 7200  # 2時間に延長
```

2. **並列処理の活用**
```javascript
// 独立したエージェントを並列実行
const [analysis, scoring] = await Promise.all([
  analyzer.process(data),
  scorer.process(data)
]);
```

3. **バッチサイズの調整**
```yaml
limits:
  max_modules_per_request: 30  # 小さくする
```

---

#### メモリ使用量が高い

**症状**:
- OOMエラー
- スワップ使用
- レスポンス遅延

**診断**:
```bash
# メモリ使用状況
docker stats --no-stream

# ヒープダンプ取得
jmap -dump:format=b,file=heap.bin <pid>
```

**解決方法**:

1. **メモリ制限の調整**
```yaml
# docker-compose.yml
services:
  analyzer:
    mem_limit: 1g
    mem_reservation: 512m
```

2. **ガベージコレクションの調整**
```bash
# Node.js
node --max-old-space-size=1024 agent.js

# Python
export PYTHONMALLOC=malloc
```

3. **メモリリークの確認**
```javascript
// メモリプロファイリング
const v8 = require('v8');
console.log(v8.getHeapStatistics());
```

---

### 3. 分類の問題

#### 期待と異なる分類結果

**症状**:
- Core であるべきものが Supporting/Generic に分類
- スコアと分類の不一致

**診断**:
```javascript
// 分類ルールの確認
const { S, Δ, X } = item.scores;
console.log(`S=${S}, Δ=${Δ}, X=${X}`);
console.log(`S+Δ=${S+Δ}, Expected: Core if >= 8 and X <= 3`);
```

**確認事項**:

1. **証拠の十分性**
```json
// 証拠が不足していないか確認
{
  "evidence": ["具体的なファイル名と関数名"],
  "uncertainties": ["追加で必要な情報"]
}
```

2. **スコアリング基準の理解**
- S (Strategic): ビジネス価値
- Δ (Differentiation): 差別化要素
- X (Externalizability): 外部化可能性

3. **カスタム設定の確認**
```yaml
# config.yaml で環境別ルールを確認
classification_rules:
  production:
    core:
      condition: "(S + Δ) >= 8 AND X <= 3"
```

---

### 4. 統合の問題

#### API接続エラー

**症状**:
```
Error: connect ECONNREFUSED 127.0.0.1:8080
```

**確認事項**:

1. **サービスの起動状態**
```bash
# サービスの確認
docker ps
systemctl status ddd-mapper

# ポートの確認
netstat -tlnp | grep 8080
```

2. **ファイアウォール設定**
```bash
# iptables確認
sudo iptables -L -n

# firewalld確認
sudo firewall-cmd --list-all
```

3. **環境変数**
```bash
# 必要な環境変数の確認
echo $API_ENDPOINT
echo $API_KEY
```

---

#### LLM API エラー

**症状**:
- Rate limit exceeded
- Invalid API key
- Model not found

**解決方法**:

1. **レート制限対策**
```yaml
# レート制限の調整
rate_limiting:
  max_requests_per_minute: 50  # 減らす
  retry_after_rate_limit: true
```

2. **APIキーの確認**
```bash
# APIキーの検証
curl -H "Authorization: Bearer $API_KEY" \
  https://api.openai.com/v1/models
```

3. **モデル指定の修正**
```yaml
ai_model:
  model: "gpt-4-turbo"  # 利用可能なモデルに変更
```

---

### 5. デバッグ手法

#### ログレベルの調整

```yaml
# 詳細ログを有効化
logging:
  level: "debug"

# 特定モジュールのみデバッグ
logging:
  modules:
    analyzer: "debug"
    scorer: "info"
```

#### トレースの有効化

```yaml
# 分散トレーシングを有効化
monitoring:
  tracing:
    enabled: true
    sampling_rate: 1.0  # 100%サンプリング（デバッグ時のみ）
```

#### ローカル実行でのデバッグ

```bash
# 単一エージェントのテスト
npm run test:agent -- --agent=analyzer --input=test.json

# ステップ実行
node --inspect-brk agent.js

# 環境変数でデバッグモード
DEBUG=* npm start
```

---

### 6. データ検証

#### 入力データの検証

```bash
# スキーマ検証
ajv validate -s schema.json -d input.json

# Python での検証
python -m json.tool input.json
```

#### 出力データの確認

```javascript
// 出力の妥当性チェック
function validateOutput(output) {
  const errors = [];

  // 必須フィールドチェック
  if (!output.items) errors.push("Missing 'items'");

  // スコア範囲チェック
  output.items.forEach(item => {
    ['S', 'Δ', 'X'].forEach(score => {
      if (item.scores[score] < 1 || item.scores[score] > 5) {
        errors.push(`Invalid score ${score}=${item.scores[score]}`);
      }
    });
  });

  return errors;
}
```

---

### 7. 復旧手順

#### サービスの再起動

```bash
# Docker Compose
docker-compose restart

# Kubernetes
kubectl rollout restart deployment/analyzer-agent

# システムサービス
sudo systemctl restart ddd-mapper
```

#### データベースの修復

```sql
-- 不整合データの確認
SELECT * FROM analysis_results
WHERE classification NOT IN ('Core', 'Supporting', 'Generic');

-- インデックスの再構築
REINDEX TABLE analysis_results;

-- 統計情報の更新
ANALYZE analysis_results;
```

#### キャッシュのクリア

```bash
# Redis キャッシュクリア
redis-cli FLUSHDB

# 特定パターンのみクリア
redis-cli --scan --pattern "agent:*" | xargs redis-cli DEL
```

---

### 8. エスカレーション手順

#### レベル1: セルフサービス
1. このトラブルシューティングガイドを確認
2. ログを確認
3. 設定を確認

#### レベル2: チームサポート
1. Slackの #ddd-mapper-support チャンネルに質問
2. 以下の情報を提供:
   - エラーメッセージ
   - 入力データ（センシティブ情報を除く）
   - 実行環境
   - 再現手順

#### レベル3: エンジニアリング
1. GitHubにIssueを作成
2. デバッグログを添付
3. 最小限の再現コードを提供

---

### 9. 予防保守

#### 定期的な確認事項

**日次**:
- エラー率の確認
- ディスク使用量の確認
- API利用量の確認

**週次**:
- パフォーマンスメトリクスのレビュー
- ログローテーションの確認
- バックアップの検証

**月次**:
- 依存関係の更新
- セキュリティパッチの適用
- キャパシティプランニング

#### モニタリングアラート設定

```yaml
alerts:
  - name: "High Error Rate"
    condition: "error_rate > 0.05"
    action: "email,slack"

  - name: "Low Cache Hit Rate"
    condition: "cache_hit_rate < 0.5"
    action: "slack"

  - name: "High Memory Usage"
    condition: "memory_usage > 90%"
    action: "pagerduty"
```

---

### 10. FAQ

**Q: 処理が完了しない**
A: タイムアウト設定を確認し、必要に応じて延長してください。大量データの場合は分割処理を検討してください。

**Q: 同じ入力で異なる結果が返る**
A: temperature設定が0になっているか確認してください。キャッシュが有効な場合は、キャッシュをクリアしてください。

**Q: Coreドメインが検出されない**
A: 証拠データが十分か確認してください。スコアリング基準を理解し、必要に応じてカスタム設定を調整してください。

**Q: メモリ不足エラーが発生**
A: モジュール数を減らすか、バッチサイズを小さくしてください。メモリ制限を増やすことも検討してください。

**Q: APIキーが無効と表示される**
A: APIキーの有効期限と権限を確認してください。環境変数が正しく設定されているか確認してください。