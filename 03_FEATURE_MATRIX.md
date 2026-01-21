# Feature Matrix: solver-multiRPC vs eRPC

## Complete Feature Comparison

### Legend
- ✅ Full support
- ⚡ Partial/Basic support
- ❌ Not supported

---

## 1. Core RPC Features

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Multiple RPC Endpoints** | ✅ | ✅ | Both support multiple providers |
| **Automatic Failover** | ✅ | ✅ | eRPC more sophisticated |
| **Load Balancing** | ⚡ Priority-based | ✅ Score-based | eRPC uses health scores |
| **RPC Health Tracking** | ⚡ Connection check | ✅ Full metrics | eRPC tracks latency, errors, etc |
| **JSON-RPC Support** | ✅ | ✅ | Standard JSON-RPC 2.0 |
| **Batch Requests** | ✅ (via multicall) | ✅ Smart batching | eRPC aggregates automatically |
| **WebSocket Support** | ✅ | ⚡ Limited | solver-multiRPC better WS support |

---

## 2. Reliability & Fault Tolerance

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Retry Logic** | ✅ Basic | ✅ Advanced | eRPC: exponential backoff, jitter |
| **Circuit Breaker** | ❌ | ✅ | eRPC prevents cascading failures |
| **Timeout Handling** | ⚡ Fixed | ✅ Configurable per-method | |
| **Hedged Requests** | ❌ | ✅ | eRPC sends parallel speculative requests |
| **Consensus Policy** | ⚡ ViewPolicy | ✅ Full consensus | eRPC compares multiple upstream results |
| **Re-org Awareness** | ❌ | ✅ | eRPC invalidates cache on reorgs |
| **Data Integrity Validation** | ❌ | ✅ | eRPC validates response correctness |

### Retry Configuration Comparison

**solver-multiRPC:**
```python
# Basic retry through bracket iteration
# Automatically tries next priority bracket on failure
for p, c in zip(providers['transaction'].values(), contracts['transaction'].values()):
    try:
        return await self.__call_tx(...)
    except (ConnectionError, ReadTimeout, ...):
        pass  # Try next bracket
```

**eRPC:**
```yaml
failsafe:
  - retry:
      maxAttempts: 5
      delay: 100ms
      backoffMaxDelay: 2s
      backoffFactor: 2.0
      jitter: 50ms
      emptyResultConfidence: finalized
```

---

## 3. Caching

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Response Caching** | ❌ | ✅ | eRPC has multi-tier caching |
| **Memory Cache** | ❌ | ✅ | Ristretto-based LRU |
| **Redis Cache** | ❌ | ✅ | Distributed caching |
| **PostgreSQL Cache** | ❌ | ✅ | Persistent storage |
| **DynamoDB Cache** | ❌ | ✅ | AWS integration |
| **Cache TTL Policies** | ❌ | ✅ | Per-method TTL configuration |
| **Finality-aware Caching** | ❌ | ✅ | Different TTL for finalized vs latest |
| **Cache Compression** | ❌ | ✅ | zstd compression |

### eRPC Cache Policy Example

```yaml
database:
  evmJsonRpcCache:
    connectors:
      - id: memory
        driver: memory
        memory:
          maxItems: 100000
          maxTotalSize: 1GB
      - id: redis
        driver: redis
        redis:
          addr: localhost:6379
    policies:
      # Finalized data - long TTL
      - connector: redis
        finality: finalized
        ttl: 168h  # 1 week
      
      # Latest/unfinalized - short TTL
      - connector: memory
        finality: unknown
        ttl: 5s
      
      # Specific method override
      - connector: redis
        method: eth_getBlockByNumber
        finality: finalized
        ttl: 720h  # 30 days
```

---

## 4. Rate Limiting

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Per-Upstream Limits** | ❌ | ✅ | |
| **Per-Project Limits** | ❌ | ✅ | Multi-tenant support |
| **Per-User Limits** | ❌ | ✅ | Based on auth |
| **Per-Method Limits** | ❌ | ✅ | |
| **Auto-tuning** | ❌ | ✅ | Adjusts based on 429 errors |
| **Distributed Limits** | ❌ | ✅ | Via Redis |

### eRPC Rate Limiter Configuration

```yaml
rateLimiters:
  budgets:
    - id: alchemy-budget
      rules:
        - method: "*"
          maxCount: 100
          period: second
        - method: eth_getLogs
          maxCount: 10
          period: second
        - method: eth_call
          maxCount: 200
          period: second

upstreams:
  - id: alchemy
    rateLimitBudget: alchemy-budget
    rateLimitAutoTune:
      enabled: true
      adjustmentPeriod: 1m
      errorRateThreshold: 0.1
      increaseFactor: 1.1
      decreaseFactor: 0.9
```

---

## 5. EVM-Specific Features

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Chain ID Detection** | ✅ | ✅ | |
| **Block Number Tracking** | ⚡ Per-call | ✅ Continuous polling | |
| **Finality Detection** | ❌ | ✅ | Latest vs finalized blocks |
| **eth_getLogs Splitting** | ❌ | ✅ | Auto-splits large ranges |
| **Block Availability Tracking** | ❌ | ✅ | Per-upstream availability bounds |
| **Archive Node Detection** | ❌ | ✅ | Identifies full vs archive nodes |
| **Syncing State Detection** | ❌ | ✅ | Skips syncing nodes |
| **Proof of Authority Support** | ✅ | ✅ | PoA middleware |

---

## 6. Transaction Features

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Transaction Building** | ✅ | ❌ | solver-multiRPC builds txs |
| **Transaction Signing** | ✅ | ❌ | solver-multiRPC signs txs |
| **Gas Estimation** | ✅ | ❌ | Multiple strategies |
| **Gas Price APIs** | ✅ | ❌ | External gas API integration |
| **Transaction Broadcasting** | ✅ | ✅ (pass-through) | |
| **Receipt Waiting** | ✅ | ✅ (pass-through) | |
| **Transaction Tracing** | ✅ | ❌ | TxTrace class |
| **Idempotent Broadcasting** | ❌ | ✅ | Handles "already known" errors |

### solver-multiRPC Gas Estimation

```python
class GasEstimation:
    """
    Multiple strategies:
    - GAS_API_PROVIDER: External gas station APIs
    - RPC: Query gas price from RPC
    - FIXED: Use fixed values per chain
    - CUSTOM: User-defined logic
    """
    
    async def get_gas_price(self, gas_upper_bound, priority, method):
        # Try methods in order until one succeeds
        for method_key in self.method_sorted_priority:
            try:
                return await self.gas_estimation_method[method_key](priority, gas_upper_bound)
            except (FailedToGetGasPrice, OutOfRangeTransactionFee):
                continue
```

---

## 7. Authentication & Security

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **API Key Auth** | ❌ | ✅ | Secret strategy |
| **JWT Auth** | ❌ | ✅ | Full JWT support |
| **SIWE Auth** | ❌ | ✅ | Sign-In with Ethereum |
| **Database Auth** | ❌ | ✅ | Lookup from DB |
| **Network/IP Auth** | ❌ | ✅ | IP allowlisting |
| **TLS/HTTPS** | ❌ | ✅ | |
| **CORS Support** | ❌ | ✅ | |
| **Private Key Handling** | ✅ | ❌ | solver-multiRPC manages keys |

### eRPC Authentication Configuration

```yaml
projects:
  - id: main
    auth:
      strategies:
        - type: secret
          secret:
            value: ${API_SECRET}
        - type: jwt
          jwt:
            allowedIssuers: ["https://auth.example.com"]
            verificationKeys:
              RS256: |
                -----BEGIN PUBLIC KEY-----
                ...
                -----END PUBLIC KEY-----
        - type: siwe
          siwe:
            allowedDomains: ["app.example.com"]
```

---

## 8. Monitoring & Observability

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Logging** | ✅ Basic | ✅ Structured (zerolog) | |
| **Prometheus Metrics** | ❌ | ✅ | Full metrics suite |
| **Grafana Dashboards** | ❌ | ✅ | Pre-built dashboards |
| **OpenTelemetry Tracing** | ❌ | ✅ | Distributed tracing |
| **Health Endpoints** | ❌ | ✅ | Kubernetes-ready |
| **Admin API** | ❌ | ✅ | Runtime configuration |
| **Error Tracking** | ⚡ Logging | ✅ Structured errors | |

### eRPC Metrics Examples

```prometheus
# Request metrics
erpc_upstream_request_total{project="main", upstream="alchemy", method="eth_call"}
erpc_upstream_request_duration_seconds{project="main", upstream="alchemy"}
erpc_upstream_error_total{project="main", upstream="alchemy", error_type="timeout"}

# Cache metrics  
erpc_cache_hit_total{project="main", connector="redis"}
erpc_cache_miss_total{project="main", connector="redis"}

# Rate limit metrics
erpc_rate_limit_exceeded_total{project="main", budget="alchemy-budget"}
```

---

## 9. Configuration & Deployment

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Python API** | ✅ | ❌ | |
| **YAML Config** | ❌ | ✅ | |
| **TypeScript Config** | ❌ | ✅ | |
| **Environment Variables** | ⚡ Manual | ✅ Native support | |
| **Docker Image** | ❌ | ✅ | Official images |
| **Kubernetes Manifests** | ❌ | ✅ | |
| **Railway Deploy** | ❌ | ✅ | One-click deploy |
| **npx Quick Start** | ❌ | ✅ | `npx start-erpc` |

---

## 10. Multi-Chain Support

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Single Chain Instance** | ✅ | ✅ | |
| **Multiple Chains** | ⚡ Multiple instances | ✅ Single instance | eRPC handles all chains |
| **Chain Discovery** | ❌ | ✅ | Built-in chain registry |
| **Public RPC Directory** | ❌ | ✅ | 4000+ public endpoints |
| **Chain Aliasing** | ❌ | ✅ | Custom network names |

---

## 11. Advanced Features

| Feature | solver-multiRPC | eRPC | Notes |
|---------|-----------------|------|-------|
| **Multicall Integration** | ✅ | ⚡ Batch support | solver-multiRPC native multicall |
| **Selection Policies** | ❌ | ✅ | JavaScript DSL |
| **Shadow Mode** | ❌ | ✅ | Compare upstreams without serving |
| **Consensus Misbehavior Export** | ❌ | ✅ | Export to S3/file |
| **Block Heatmap** | ❌ | ✅ | Visualize block distribution |
| **Method Routing** | ⚡ View/Tx split | ✅ Per-method rules | |
| **Response Interpolation** | ❌ | ✅ | Fill missing block data |

### eRPC Selection Policy Example

```yaml
selectionPolicy:
  evalInterval: 10s
  evalFunction: |
    (upstreams) => {
      // Custom upstream selection logic
      return upstreams
        .filter(u => u.metrics.errorRate < 0.1)
        .sort((a, b) => a.metrics.latency - b.metrics.latency)
        .slice(0, 3);
    }
```

---

## Feature Summary Score

| Category | solver-multiRPC | eRPC |
|----------|-----------------|------|
| Core RPC | 7/10 | 10/10 |
| Reliability | 4/10 | 10/10 |
| Caching | 0/10 | 10/10 |
| Rate Limiting | 0/10 | 10/10 |
| EVM Features | 4/10 | 9/10 |
| Transactions | 9/10 | 3/10 |
| Auth & Security | 1/10 | 10/10 |
| Monitoring | 2/10 | 10/10 |
| Configuration | 6/10 | 9/10 |
| Multi-Chain | 4/10 | 10/10 |
| **Overall** | **37/100** | **91/100** |

> Note: Scores are relative to infrastructure/proxy use cases. For Python-native transaction-focused applications, solver-multiRPC scores higher in relevant categories.
