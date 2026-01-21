# Migration Guide: Between solver-multiRPC and eRPC

## Overview

This guide provides step-by-step instructions for migrating between the two solutions, depending on your needs.

---

## Part 1: Migrating from solver-multiRPC to eRPC

### Why Migrate?

Consider migrating when:
- Request volume exceeds library capabilities
- Need caching to reduce costs
- Require enterprise reliability features
- Moving to multi-language architecture
- Need better observability

### Step 1: Deploy eRPC

#### Option A: Quick Start with npx

```bash
npx start-erpc
```

#### Option B: Docker Deployment

```bash
# Create configuration file
cat > erpc.yaml << 'EOF'
logLevel: info
server:
  httpPort: 4000

projects:
  - id: main
    networks:
      - architecture: evm
        evm:
          chainId: 1
    upstreams:
      - id: primary
        endpoint: https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}
      - id: backup
        endpoint: https://mainnet.infura.io/v3/${INFURA_KEY}
EOF

# Run eRPC
docker run -p 4000:4000 \
  -v $(pwd)/erpc.yaml:/root/erpc.yaml \
  -e ALCHEMY_KEY=${ALCHEMY_KEY} \
  -e INFURA_KEY=${INFURA_KEY} \
  ghcr.io/erpc/erpc
```

### Step 2: Update Python Application

#### Before (solver-multiRPC)

```python
from multirpc import AsyncMultiRpc
from multirpc.utils import NestedDict

class MyDApp:
    def __init__(self):
        rpcs = NestedDict({
            "view": {
                1: ['https://eth-mainnet.g.alchemy.com/v2/KEY'],
                2: ['https://mainnet.infura.io/v3/KEY'],
            },
            "transaction": {
                1: ['https://eth-mainnet.g.alchemy.com/v2/KEY'],
            }
        })
        
        self.multi_rpc = AsyncMultiRpc(
            rpc_urls=rpcs,
            contract_address='0x...',
            contract_abi=ABI,
            enable_gas_estimation=True,
        )
        self.multi_rpc.set_account(ADDRESS, PRIVATE_KEY)

    async def get_balance(self, address):
        return await self.multi_rpc.functions.balanceOf(address).call()

    async def transfer(self, to, amount):
        return await self.multi_rpc.functions.transfer(to, amount).call()
```

#### After (Using eRPC with Python)

```python
import aiohttp
from eth_account import Account
from web3 import Web3

class MyDApp:
    def __init__(self):
        # eRPC endpoint
        self.erpc_url = "http://localhost:4000/main/evm/1"
        self.session = aiohttp.ClientSession()
        
        # For transaction signing
        self.account = Account.from_key(PRIVATE_KEY)
        self.w3 = Web3()  # Just for encoding
        self.contract = self.w3.eth.contract(
            address='0x...',
            abi=ABI
        )
    
    async def _rpc_call(self, method, params):
        """Generic RPC call through eRPC"""
        async with self.session.post(self.erpc_url, json={
            "jsonrpc": "2.0",
            "method": method,
            "params": params,
            "id": 1
        }) as response:
            result = await response.json()
            if "error" in result:
                raise Exception(result["error"])
            return result["result"]
    
    async def get_balance(self, address):
        # Encode the call
        call_data = self.contract.encodeABI(
            fn_name='balanceOf',
            args=[address]
        )
        
        # Call via eRPC (cached!)
        result = await self._rpc_call("eth_call", [{
            "to": self.contract.address,
            "data": call_data
        }, "latest"])
        
        # Decode result
        return self.w3.codec.decode(['uint256'], bytes.fromhex(result[2:]))[0]
    
    async def transfer(self, to, amount):
        # Build transaction
        nonce = int(await self._rpc_call(
            "eth_getTransactionCount",
            [self.account.address, "latest"]
        ), 16)
        
        gas_price = int(await self._rpc_call("eth_gasPrice", []), 16)
        
        tx = self.contract.functions.transfer(to, amount).build_transaction({
            'chainId': 1,
            'gas': 100000,
            'gasPrice': gas_price,
            'nonce': nonce,
        })
        
        # Sign locally
        signed = self.account.sign_transaction(tx)
        
        # Broadcast via eRPC
        tx_hash = await self._rpc_call(
            "eth_sendRawTransaction",
            [signed.rawTransaction.hex()]
        )
        
        return tx_hash
```

### Step 3: Create Helper Library

For easier migration, create a wrapper that mimics solver-multiRPC API:

```python
# erpc_wrapper.py
import aiohttp
from eth_account import Account
from web3 import Web3

class ERPCMultiRpc:
    """
    Drop-in replacement for solver-multiRPC that uses eRPC backend.
    """
    
    def __init__(
        self,
        erpc_url: str,
        contract_address: str,
        contract_abi: list,
    ):
        self.erpc_url = erpc_url
        self.session = None
        self.w3 = Web3()
        self.contract_address = Web3.to_checksum_address(contract_address)
        self.contract = self.w3.eth.contract(
            address=self.contract_address,
            abi=contract_abi
        )
        self.account = None
        self.private_key = None
        self.functions = ContractFunctions(self)
    
    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        return self
    
    async def __aexit__(self, *args):
        await self.session.close()
    
    def set_account(self, address: str, private_key: str):
        self.account = Account.from_key(private_key)
        self.private_key = private_key
    
    async def rpc_call(self, method: str, params: list):
        async with self.session.post(self.erpc_url, json={
            "jsonrpc": "2.0",
            "method": method,
            "params": params,
            "id": 1
        }) as response:
            result = await response.json()
            if "error" in result:
                raise Exception(f"RPC Error: {result['error']}")
            return result["result"]

class ContractFunctions:
    def __init__(self, multi_rpc: ERPCMultiRpc):
        self.multi_rpc = multi_rpc
    
    def __getattr__(self, name: str):
        return ContractFunction(name, self.multi_rpc)

class ContractFunction:
    def __init__(self, name: str, multi_rpc: ERPCMultiRpc):
        self.name = name
        self.multi_rpc = multi_rpc
        self.args = ()
        self.kwargs = {}
    
    def __call__(self, *args, **kwargs):
        func = ContractFunction(self.name, self.multi_rpc)
        func.args = args
        func.kwargs = kwargs
        return func
    
    async def call(self, block_identifier='latest'):
        """Execute view function"""
        contract_func = getattr(
            self.multi_rpc.contract.functions,
            self.name
        )(*self.args)
        
        call_data = contract_func._encode_transaction_data()
        
        result = await self.multi_rpc.rpc_call("eth_call", [{
            "to": self.multi_rpc.contract_address,
            "data": call_data
        }, block_identifier])
        
        # Decode and return
        return contract_func.decode_output(bytes.fromhex(result[2:]))

# Usage - almost identical to solver-multiRPC
async def main():
    async with ERPCMultiRpc(
        erpc_url="http://localhost:4000/main/evm/1",
        contract_address='0x...',
        contract_abi=ABI,
    ) as multi_rpc:
        multi_rpc.set_account(ADDRESS, PRIVATE_KEY)
        
        # Same API as before!
        balance = await multi_rpc.functions.balanceOf(ADDRESS).call()
        print(f"Balance: {balance}")
```

### Step 4: Configure eRPC for Production

```yaml
# production-erpc.yaml
logLevel: warn

server:
  httpPort: 4000
  maxTimeout: 60s
  enableGzip: true

database:
  evmJsonRpcCache:
    connectors:
      - id: redis
        driver: redis
        redis:
          addr: ${REDIS_URL}
          connPoolSize: 100
    policies:
      - connector: redis
        finality: finalized
        ttl: 168h
      - connector: redis
        finality: unknown
        ttl: 12s

metrics:
  enabled: true
  port: 9090

projects:
  - id: main
    auth:
      strategies:
        - type: secret
          secret:
            value: ${API_SECRET}
    networks:
      - architecture: evm
        evm:
          chainId: 1
        failsafe:
          - retry:
              maxAttempts: 3
              delay: 200ms
          - circuitBreaker:
              failureThresholdCount: 5
              halfOpenAfter: 30s
          - hedge:
              delay: 500ms
              maxCount: 2
    upstreams:
      - id: alchemy
        endpoint: https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}
        rateLimitBudget: alchemy-budget
      - id: infura
        endpoint: https://mainnet.infura.io/v3/${INFURA_KEY}
        rateLimitBudget: infura-budget

rateLimiters:
  budgets:
    - id: alchemy-budget
      rules:
        - method: "*"
          maxCount: 300
          period: second
    - id: infura-budget
      rules:
        - method: "*"
          maxCount: 100
          period: second
```

---

## Part 2: Migrating from eRPC to solver-multiRPC

### Why Migrate?

Consider migrating when:
- Simplifying Python-only application
- Need direct transaction management
- Reducing infrastructure complexity
- Building DeFi bots with tight integration needs

### Step 1: Install solver-multiRPC

```bash
pip install solver-multiRPC
```

### Step 2: Convert Configuration

#### Before (eRPC erpc.yaml)

```yaml
projects:
  - id: main
    upstreams:
      - id: alchemy
        endpoint: https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}
      - id: infura
        endpoint: https://mainnet.infura.io/v3/${INFURA_KEY}
      - id: ankr
        endpoint: https://rpc.ankr.com/eth
```

#### After (solver-multiRPC Python)

```python
from multirpc.utils import NestedDict
import os

rpcs = NestedDict({
    "view": {
        1: [
            f"https://eth-mainnet.g.alchemy.com/v2/{os.environ['ALCHEMY_KEY']}",
            f"https://mainnet.infura.io/v3/{os.environ['INFURA_KEY']}",
        ],
        2: [
            "https://rpc.ankr.com/eth",
        ],
    },
    "transaction": {
        1: [
            f"https://eth-mainnet.g.alchemy.com/v2/{os.environ['ALCHEMY_KEY']}",
        ],
        2: [
            f"https://mainnet.infura.io/v3/{os.environ['INFURA_KEY']}",
        ],
    }
})
```

### Step 3: Update Application Code

#### Before (HTTP calls to eRPC)

```python
async def get_data():
    async with aiohttp.ClientSession() as session:
        response = await session.post(
            "http://localhost:4000/main/evm/1",
            json={
                "jsonrpc": "2.0",
                "method": "eth_call",
                "params": [{"to": "0x...", "data": "0x..."}, "latest"],
                "id": 1
            }
        )
        return await response.json()
```

#### After (solver-multiRPC)

```python
from multirpc import AsyncMultiRpc

multi_rpc = AsyncMultiRpc(
    rpc_urls=rpcs,
    contract_address='0x...',
    contract_abi=ABI,
)

async def get_data():
    # Direct contract method call
    return await multi_rpc.functions.getData().call()
```

### Step 4: Handle Loss of Features

When migrating from eRPC, you lose certain features. Here's how to compensate:

#### Loss: Caching

```python
# Implement simple caching manually
from functools import lru_cache
import asyncio

class CachedMultiRpc:
    def __init__(self, multi_rpc, cache_ttl=60):
        self.multi_rpc = multi_rpc
        self.cache = {}
        self.cache_ttl = cache_ttl
    
    async def cached_call(self, func_name, *args, cache_key=None):
        key = cache_key or f"{func_name}:{args}"
        
        # Check cache
        if key in self.cache:
            value, timestamp = self.cache[key]
            if time.time() - timestamp < self.cache_ttl:
                return value
        
        # Call and cache
        func = getattr(self.multi_rpc.functions, func_name)
        result = await func(*args).call()
        self.cache[key] = (result, time.time())
        return result
```

#### Loss: Metrics

```python
# Add basic metrics
import time
from collections import defaultdict

class MetricsWrapper:
    def __init__(self):
        self.request_count = defaultdict(int)
        self.error_count = defaultdict(int)
        self.latency_sum = defaultdict(float)
    
    async def call_with_metrics(self, func_name, func):
        start = time.time()
        try:
            result = await func
            self.request_count[func_name] += 1
            self.latency_sum[func_name] += time.time() - start
            return result
        except Exception as e:
            self.error_count[func_name] += 1
            raise
    
    def get_stats(self, func_name):
        count = self.request_count[func_name]
        return {
            "requests": count,
            "errors": self.error_count[func_name],
            "avg_latency": self.latency_sum[func_name] / count if count else 0
        }
```

#### Loss: Circuit Breaker

```python
# Simple circuit breaker implementation
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.state = "closed"
        self.last_failure_time = None
    
    def call(self, func):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "half-open"
            else:
                raise Exception("Circuit breaker is open")
        
        try:
            result = func()
            if self.state == "half-open":
                self.state = "closed"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            raise
```

---

## Part 3: Running Both Solutions

For a gradual migration, you can run both solutions simultaneously.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Your Application                       │
│  ┌─────────────────────┐  ┌─────────────────────────┐  │
│  │  solver-multiRPC    │  │     eRPC Client         │  │
│  │  (Transactions)     │  │     (Cached Reads)      │  │
│  └──────────┬──────────┘  └───────────┬─────────────┘  │
└─────────────┼─────────────────────────┼─────────────────┘
              │                         │
              │                    ┌────▼────┐
              │                    │  eRPC   │
              │                    │  Proxy  │
              │                    └────┬────┘
              │                         │
              └─────────────────────────┼─────────────────┘
                                        │
                               ┌────────▼────────┐
                               │  RPC Providers  │
                               └─────────────────┘
```

### Implementation

```python
class HybridRpcClient:
    """Use both solutions optimally"""
    
    def __init__(
        self,
        rpcs: NestedDict,
        erpc_url: str,
        contract_address: str,
        contract_abi: list,
    ):
        # solver-multiRPC for transactions
        self.tx_client = AsyncMultiRpc(
            rpc_urls=rpcs,
            contract_address=contract_address,
            contract_abi=contract_abi,
            enable_gas_estimation=True,
        )
        
        # eRPC for cached reads
        self.erpc_url = erpc_url
        self.http_session = None
        
        # Contract for encoding
        self.w3 = Web3()
        self.contract = self.w3.eth.contract(
            address=contract_address,
            abi=contract_abi
        )
    
    async def setup(self):
        self.http_session = aiohttp.ClientSession()
    
    async def close(self):
        await self.http_session.close()
    
    def set_account(self, address, private_key):
        self.tx_client.set_account(address, private_key)
    
    async def view(self, func_name, *args, block_identifier='latest'):
        """
        View calls go through eRPC for caching benefit.
        """
        func = getattr(self.contract.functions, func_name)(*args)
        call_data = func._encode_transaction_data()
        
        async with self.http_session.post(self.erpc_url, json={
            "jsonrpc": "2.0",
            "method": "eth_call",
            "params": [{
                "to": self.contract.address,
                "data": call_data
            }, block_identifier],
            "id": 1
        }) as response:
            result = await response.json()
            if "error" in result:
                raise Exception(result["error"])
            return func.decode_output(bytes.fromhex(result["result"][2:]))
    
    async def transact(self, func_name, *args, **kwargs):
        """
        Transactions go through solver-multiRPC for signing.
        """
        func = getattr(self.tx_client.functions, func_name)
        return await func(*args).call(**kwargs)

# Usage
async def main():
    client = HybridRpcClient(
        rpcs=rpcs,
        erpc_url="http://localhost:4000/main/evm/1",
        contract_address=CONTRACT,
        contract_abi=ABI,
    )
    await client.setup()
    client.set_account(ADDRESS, PRIVATE_KEY)
    
    try:
        # Cached read via eRPC
        balance = await client.view('balanceOf', ADDRESS)
        
        # Transaction via solver-multiRPC
        tx = await client.transact('transfer', TO_ADDRESS, AMOUNT)
        
    finally:
        await client.close()
```

---

## Summary

| Migration Path | Effort | Benefit |
|----------------|--------|---------|
| solver-multiRPC → eRPC | Medium | Caching, reliability, monitoring |
| eRPC → solver-multiRPC | Low | Simplicity, Python-native transactions |
| Hybrid approach | Medium | Best of both worlds |

Choose your migration path based on your specific requirements and constraints.
