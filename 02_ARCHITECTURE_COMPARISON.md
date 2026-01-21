# Architecture Comparison: solver-multiRPC vs eRPC

## Overview

This document provides an in-depth comparison of the architectural designs between `solver-multiRPC` and `eRPC`.

---

## 1. High-Level Architecture

### solver-multiRPC Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Python Application                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │              solver-multiRPC Library             │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────┐  │   │
│  │  │ AsyncMultiRpc│  │   MultiRpc  │  │   Gas   │  │   │
│  │  │  (Async)     │  │   (Sync)    │  │ Estimator│  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └────┬────┘  │   │
│  │         │                 │               │       │   │
│  │  ┌──────▼─────────────────▼───────────────▼────┐ │   │
│  │  │           BaseMultiRpc (ABC)                 │ │   │
│  │  │  - RPC bracket management                    │ │   │
│  │  │  - Contract interaction                      │ │   │
│  │  │  - Transaction building/signing              │ │   │
│  │  └──────────────────────┬──────────────────────┘ │   │
│  └─────────────────────────┼────────────────────────┘   │
│                            │                             │
└────────────────────────────┼─────────────────────────────┘
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        ▼                        │
    │  ┌──────────┐   ┌──────────┐   ┌──────────┐   │
    │  │  RPC 1   │   │  RPC 2   │   │  RPC 3   │   │
    │  │ Priority 1│   │ Priority 2│   │ Priority 3│   │
    │  └──────────┘   └──────────┘   └──────────┘   │
    │              RPC Endpoints (Brackets)          │
    └────────────────────────────────────────────────┘
```

**Key Characteristics:**
- **Embedded Library**: Runs within your Python application
- **Direct Integration**: Uses Web3.py directly
- **Bracket System**: RPCs organized by priority tiers
- **Synchronous & Asynchronous**: Both patterns supported
- **Single Chain Focus**: One contract/chain per instance

### eRPC Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        eRPC Server Process                           │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      HTTP/gRPC Server                          │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐  │  │
│  │  │   Auth Layer    │  │  Rate Limiters  │  │   Metrics     │  │  │
│  │  │ (JWT/SIWE/etc)  │  │  (Per-project)  │  │  (Prometheus) │  │  │
│  │  └────────┬────────┘  └────────┬────────┘  └───────────────┘  │  │
│  └───────────┼────────────────────┼───────────────────────────────┘  │
│              │                    │                                   │
│  ┌───────────▼────────────────────▼───────────────────────────────┐  │
│  │                    Projects Registry                            │  │
│  │  ┌───────────────────────────────────────────────────────────┐ │  │
│  │  │                    Project A                               │ │  │
│  │  │  ┌─────────────────────────────────────────────────────┐  │ │  │
│  │  │  │              Networks Registry                       │  │ │  │
│  │  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │  │ │  │
│  │  │  │  │ Network:EVM:1│  │Network:EVM:42│  │Network:EVM:N│ │  │ │  │
│  │  │  │  │  (Ethereum)  │  │  (Arbitrum)  │  │  (Other)   │ │  │ │  │
│  │  │  │  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │  │ │  │
│  │  │  └─────────┼─────────────────┼────────────────┼─────────┘  │ │  │
│  │  └────────────┼─────────────────┼────────────────┼────────────┘ │  │
│  └───────────────┼─────────────────┼────────────────┼──────────────┘  │
│                  │                 │                │                  │
│  ┌───────────────▼─────────────────▼────────────────▼──────────────┐  │
│  │                   Upstreams Registry                             │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐│    │  │
│  │  │  │Upstream 1│  │Upstream 2│  │Upstream 3│  │Upstream N││    │  │
│  │  │  │(Alchemy) │  │(Infura)  │  │(Ankr)    │  │(Custom)  ││    │  │
│  │  │  │          │  │          │  │          │  │          ││    │  │
│  │  │  │•Failsafe │  │•Failsafe │  │•Failsafe │  │•Failsafe ││    │  │
│  │  │  │•RateLim  │  │•RateLim  │  │•RateLim  │  │•RateLim  ││    │  │
│  │  │  │•Metrics  │  │•Metrics  │  │•Metrics  │  │•Metrics  ││    │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘│    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         Data Layer                                │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  │  │
│  │  │   Memory   │  │   Redis    │  │ PostgreSQL │  │  DynamoDB  │  │  │
│  │  │   Cache    │  │   Cache    │  │   Cache    │  │   Cache    │  │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
         ┌──────────────────────────────────────────────┐
         │              Upstream Providers               │
         │  Alchemy • Infura • Ankr • Public RPCs       │
         │  QuickNode • Custom endpoints • etc.         │
         └──────────────────────────────────────────────┘
```

**Key Characteristics:**
- **Standalone Proxy**: Runs as a separate service
- **Language Agnostic**: Any client can use HTTP JSON-RPC
- **Multi-Project**: Supports multiple projects/tenants
- **Multi-Network**: Handles any EVM chain
- **Layered Caching**: Multiple storage backends
- **Full Observability**: Metrics, tracing, logging

---

## 2. Component Comparison

### 2.1 RPC Management

#### solver-multiRPC: Bracket System

```python
# Bracket-based RPC organization
rpcs = NestedDict({
    "view": {
        1: ['https://rpc1.io', 'https://rpc2.io'],  # Priority 1
        2: ['https://rpc3.io'],                      # Priority 2
        3: ['https://rpc4.io'],                      # Priority 3 (fallback)
    },
    "transaction": {
        1: ['https://rpc1.io'],
        2: ['https://rpc2.io', 'https://rpc3.io'],
    }
})
```

**Characteristics:**
- Separates view and transaction RPCs
- Priority-based tiers (brackets)
- Manual RPC configuration
- Simple fallback logic

#### eRPC: Upstream Registry System

```yaml
# Upstream configuration
upstreams:
  - id: alchemy-mainnet
    endpoint: https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}
    rateLimitBudget: alchemy-budget
    evm:
      chainId: 1
    failsafe:
      - timeout:
          duration: 15s
      - retry:
          maxAttempts: 3
          delay: 500ms
      - circuitBreaker:
          failureThresholdCount: 5
          halfOpenAfter: 30s
    routing:
      scoreMultipliers:
        - network: "*"
          method: "*"
          errorRate: 1.5
          respLatency: 1.2
```

**Characteristics:**
- Dynamic upstream scoring and selection
- Per-upstream failsafe policies
- Automatic method routing
- Health tracking per upstream
- Rate limit auto-tuning

### 2.2 Error Handling & Reliability

#### solver-multiRPC Approach

```python
# Custom exceptions hierarchy
class Web3InterfaceException(Exception): ...
class FailedOnAllRPCs(Web3InterfaceException): ...
class TransactionFailedStatus(Web3InterfaceException): ...
class OutOfRangeTransactionFee(Web3InterfaceException): ...

# Error handling in execution
async def __execute_batch_tasks(execution_list, exception_handler, final_exception):
    """Execute tasks and return first success, handle exceptions"""
    for task in tasks:
        try:
            result = await task
            if cancel_event.is_set():
                break
        except tuple(exception_handler) as e:
            exception = e
            continue
```

#### eRPC Approach

```go
// Failsafe policies with circuit breaker
type FailsafeConfig struct {
    Retry          *RetryPolicyConfig
    CircuitBreaker *CircuitBreakerPolicyConfig
    Timeout        *TimeoutPolicyConfig
    Hedge          *HedgePolicyConfig
    Consensus      *ConsensusPolicyConfig
}

// Sophisticated error classification
func ClassifySeverity(err error) ErrorSeverity {
    // Classifies errors into: Critical, High, Medium, Low, Info
}

// Circuit breaker integration
type CircuitBreakerPolicyConfig struct {
    FailureThresholdCount    uint
    FailureThresholdCapacity uint
    HalfOpenAfter            Duration
    SuccessThresholdCount    uint
}
```

### 2.3 Request Flow Comparison

#### solver-multiRPC Request Flow

```
1. User calls contract function via MultiRpc
2. Function determines type (view vs transaction)
3. For view: Query all RPCs, select based on ViewPolicy
   - MostUpdated: Wait for all, return highest block result
   - FirstSuccess: Return first successful response
4. For transaction:
   a. Get nonce from RPCs
   b. Build transaction with gas params
   c. Sign transaction locally
   d. Broadcast to all transaction RPCs
   e. Wait for receipt from first responding RPC
5. Return result to user
```

#### eRPC Request Flow

```
1. HTTP request arrives at eRPC server
2. Authentication check (if configured)
3. Rate limiting check (project + network level)
4. Cache lookup (if enabled)
   - Check memory cache
   - Check Redis/PostgreSQL/DynamoDB
5. If cache miss:
   a. Select upstreams based on:
      - Health scores
      - Selection policy evaluation
      - Block availability
   b. Execute with failsafe:
      - Timeout policy
      - Retry policy
      - Hedge policy (speculative parallel requests)
      - Circuit breaker
      - Consensus policy (if enabled)
   c. Validate response integrity
   d. Cache result (with reorg awareness)
6. Return response to client
7. Record metrics
```

---

## 3. Data Flow Architecture

### solver-multiRPC Data Flow

```
┌─────────────┐     ┌────────────────┐     ┌─────────────┐
│   Python    │────▶│ solver-multiRPC│────▶│   RPC       │
│    App      │◀────│   Library      │◀────│  Endpoints  │
└─────────────┘     └────────────────┘     └─────────────┘
                           │
                    No caching layer
                    No metrics storage
                    Direct pass-through
```

### eRPC Data Flow

```
┌─────────────┐     ┌────────────────┐     ┌─────────────┐
│   Any       │────▶│     eRPC       │────▶│   RPC       │
│   Client    │◀────│    Server      │◀────│  Upstreams  │
└─────────────┘     └───────┬────────┘     └─────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Memory   │ │  Redis   │ │PostgreSQL│
        │ Cache    │ │  Cache   │ │  Cache   │
        └──────────┘ └──────────┘ └──────────┘
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                    ┌──────────────┐
                    │  Prometheus  │
                    │   Metrics    │
                    └──────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │   Grafana    │
                    │  Dashboards  │
                    └──────────────┘
```

---

## 4. Configuration Architecture

### solver-multiRPC Configuration

Configuration is done programmatically in Python:

```python
# In-code configuration
multi_rpc = AsyncMultiRpc(
    rpc_urls=rpcs,
    contract_address='0x...',
    contract_abi=abi,
    view_policy=ViewPolicy.MostUpdated,
    gas_estimation=gas_estimation,
    gas_limit=1_000_000,
    gas_upper_bound=26_000,
    enable_gas_estimation=True,
    is_proof_authority=False,
)
```

### eRPC Configuration

Supports YAML and TypeScript configuration files:

```yaml
# erpc.yaml
logLevel: info
server:
  httpPort: 4000
  maxTimeout: 60s

database:
  evmJsonRpcCache:
    connectors:
      - id: memory-cache
        driver: memory
        memory:
          maxItems: 100000
          maxTotalSize: 1GB
    policies:
      - connector: memory-cache
        network: "*"
        method: "*"
        finality: finalized
        ttl: 24h

projects:
  - id: main
    networks:
      - architecture: evm
        evm:
          chainId: 1
        failsafe:
          - timeout:
              duration: 30s
            retry:
              maxAttempts: 3
    upstreams:
      - id: alchemy
        endpoint: https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}
```

---

## 5. Deployment Architecture

### solver-multiRPC Deployment

```
┌────────────────────────────────────────────────────┐
│              Application Deployment                 │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │           Python Application                  │ │
│  │  ┌─────────────────────────────────────────┐ │ │
│  │  │  requirements.txt:                      │ │ │
│  │  │    solver-multiRPC>=3.0.0               │ │ │
│  │  │    web3>=6.0.0                          │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

**Deployment Options:**
- pip install into Python environment
- Part of application container
- No separate infrastructure

### eRPC Deployment

```
┌────────────────────────────────────────────────────────────┐
│                   eRPC Infrastructure                       │
│                                                            │
│  ┌──────────────────┐    ┌──────────────────┐             │
│  │   Load Balancer  │    │    Kubernetes    │             │
│  │                  │    │    Deployment    │             │
│  └────────┬─────────┘    └────────┬─────────┘             │
│           │                       │                        │
│  ┌────────▼───────────────────────▼────────┐              │
│  │              eRPC Cluster                │              │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │              │
│  │  │ eRPC Pod │  │ eRPC Pod │  │eRPC Pod│ │              │
│  │  └──────────┘  └──────────┘  └────────┘ │              │
│  └──────────────────┬───────────────────────┘              │
│                     │                                      │
│  ┌──────────────────▼───────────────────────┐             │
│  │            Shared Services                │             │
│  │  ┌────────┐  ┌────────┐  ┌────────────┐  │             │
│  │  │ Redis  │  │Postgres│  │ Prometheus │  │             │
│  │  │Cluster │  │   DB   │  │  + Grafana │  │             │
│  │  └────────┘  └────────┘  └────────────┘  │             │
│  └───────────────────────────────────────────┘             │
└────────────────────────────────────────────────────────────┘
```

**Deployment Options:**
- `npx start-erpc` (quick start)
- Docker: `docker run ghcr.io/erpc/erpc`
- Kubernetes with Helm charts
- Railway one-click deploy

---

## 6. Scalability Architecture

### solver-multiRPC Scalability

```
Horizontal scaling = Scale the application
                     ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   App Pod 1  │  │   App Pod 2  │  │   App Pod 3  │
│   + library  │  │   + library  │  │   + library  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │    RPC Endpoints     │
              │  (shared pressure)   │
              └──────────────────────┘

Limitations:
- Each pod manages its own RPC connections
- No shared caching between instances
- No coordinated rate limiting
- RPC providers see N x requests
```

### eRPC Scalability

```
┌──────────────────────────────────────────────────────────┐
│                    Client Applications                    │
│   App 1    App 2    App 3    App 4    App 5    App N     │
└─────┬────────┬────────┬────────┬────────┬────────┬───────┘
      └────────┴────────┴────────┴────────┴────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────┐
│                  eRPC Cluster (Scalable)                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ eRPC 1  │  │ eRPC 2  │  │ eRPC 3  │  │ eRPC N  │     │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘     │
│       └────────────┴────────────┴────────────┘           │
│                          │                               │
│  ┌───────────────────────▼───────────────────────┐      │
│  │         Shared State (Redis/PostgreSQL)        │      │
│  │  • Distributed caching                         │      │
│  │  • Coordinated rate limiting                   │      │
│  │  • Shared health metrics                       │      │
│  └────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────┐
│                    RPC Providers                          │
│   (Significantly reduced request volume due to caching)   │
└──────────────────────────────────────────────────────────┘

Benefits:
- Shared caching across all instances
- Coordinated rate limiting
- RPC providers see optimized traffic
- True horizontal scalability
```

---

## Summary

| Architecture Aspect | solver-multiRPC | eRPC |
|---------------------|-----------------|------|
| **Deployment Model** | Embedded library | Standalone service |
| **Language Binding** | Python only | Language agnostic |
| **State Management** | Per-instance | Distributed/shared |
| **Caching** | None | Multi-tier |
| **Scalability** | Limited by application | Horizontally scalable |
| **Configuration** | Code-based | File-based (YAML/TS) |
| **Observability** | Basic logging | Full stack |
| **Complexity** | Simple | Enterprise-grade |
