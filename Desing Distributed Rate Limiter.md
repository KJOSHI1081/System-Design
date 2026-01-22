# Distributed Rate Limiter Design

## Engineering Justification (The "Staff Interview" Talk Track)

If an interviewer at Meta asks why you built it this way, highlight these Architectural Decisions:

### Atomic State Management

By using Lua, we avoid the "Phantom Read" problem where two nodes see 1 token remaining and both try to grab it. The logic is encapsulated within the Redis engine.

### Network Efficiency

By implementing `batch_size`, we reduced the network I/O and CPU overhead on the Redis cluster by 99% (assuming a batch of 100). This is how you scale to billions of events.

### Thread Safety

The combination of `_registry_lock` and per-ticker `_local_locks` ensures that we don't have race conditions within a single multi-threaded Python process, while also avoiding a "Global Interpreter Lock" bottleneck for different tickers.

---

## Implementation

### Python Code

```python
import time
import threading
import redis
from typing import Dict, Optional

class FinalDistributedRateLimiter:
    """
    A high-performance, distributed rate limiter using the Token Bucket algorithm.
    
    Features: 
    - Atomic Lua scripting for thread/process safety.
    - Local Batch Leasing to reduce network latency.
    - Sharded locking to minimize local thread contention.
    """

    # The Lua script is stored as a class constant to be loaded once.
    LUA_TOKEN_BUCKET = """
    local key = KEYS[1]
    local max_capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local tokens_to_consume = tonumber(ARGV[3])
    local now = tonumber(ARGV[4])

    -- Get current state
    local state = redis.call('HMGET', key, 'tokens', 'last_update')
    local curr_tokens = tonumber(state[1]) or max_capacity
    local last_update = tonumber(state[2]) or now

    -- Calculate refill
    local time_passed = math.max(0, now - last_update)
    local refill = time_passed * refill_rate
    local new_tokens = math.min(max_capacity, curr_tokens + refill)

    -- Check if enough tokens exist for the batch
    if new_tokens >= tokens_to_consume then
        new_tokens = new_tokens - tokens_to_consume
        redis.call('HMSET', key, 'tokens', new_tokens, 'last_update', now)
        return 1 -- Success
    else
        return 0 -- Refused
    end
    """

    def __init__(self, redis_client: redis.Redis, batch_size: int = 100):
        """
        Initialize the rate limiter.
        
        Args:
            redis_client: Redis connection instance
            batch_size: Number of tokens to lease per Redis call
        """
        self.redis = redis_client
        self.batch_size = batch_size
        
        # Local state for the "Multiplier Effect" (Leasing)
        self._local_counts: Dict[str, int] = {}
        self._local_locks: Dict[str, threading.Lock] = {}
        self._registry_lock = threading.Lock()
        
        # Pre-register script for performance (EVALSHA)
        self._lua_sha = self.redis.script_load(self.LUA_TOKEN_BUCKET)

    def _get_lock(self, ticker: str) -> threading.Lock:
        """Sharded locking to ensure high-concurrency safety per ticker."""
        if ticker not in self._local_locks:
            with self._registry_lock:
                if ticker not in self._local_locks:
                    self._local_locks[ticker] = threading.Lock()
                    self._local_counts[ticker] = 0
        return self._local_locks[ticker]

    def allow_request(self, ticker: str, limit: int = 1000, refill_rate: int = 100) -> bool:
        """
        Main entry point. Checks local lease first, then goes to Redis.
        limit: Max tokens the global bucket can hold.
        refill_rate: Tokens added per second globally.
        """
        lock = self._get_lock(ticker)
        
        with lock:
            # Step 1: Check Local Lease (Nanoseconds latency)
            if self._local_counts[ticker] > 0:
                self._local_counts[ticker] -= 1
                return True

            # Step 2: Local lease empty, fetch a Batch from Redis (2ms latency)
            # We use time.time() here; in a Staff system, consider Redis TIME command for sync.
            now = time.time()
            
            try:
                # Atomically request a 'batch_size' of tokens
                success = self.redis.evalsha(
                    self._lua_sha, 1, f"rl:{ticker}", 
                    limit, refill_rate, self.batch_size, now
                )
                
                if success:
                    # We consumed 'batch_size' tokens, use 1 now, store the rest
                    self._local_counts[ticker] = self.batch_size - 1
                    return True
            except redis.exceptions.NoScriptError:
                # Fallback if script is flushed from Redis cache
                self._lua_sha = self.redis.script_load(self.LUA_TOKEN_BUCKET)
                return self.allow_request(ticker, limit, refill_rate)
            
            return False
```

---

## Example Usage: Reuters Financial Data Feed

```python
if __name__ == "__main__":
    # Setup Redis Connection
    r_client = redis.Redis(host='localhost', port=6379, db=0)
    
    # Initialize the Staff-level Limiter
    # 100 tokens per batch means we only talk to Redis once every 100 requests.
    limiter = FinalDistributedRateLimiter(r_client, batch_size=100)

    # Simulation
    ticker = "AAPL"
    for i in range(105):
        if limiter.allow_request(ticker, limit=500, refill_rate=50):
            print(f"Request {i+1}: Allowed")
        else:
            print(f"Request {i+1}: Rate Limited")
```

---

## Key Design Principles

### Token Bucket Algorithm

The token bucket algorithm is a widely-used rate limiting technique:
- Tokens accumulate at a fixed rate (refill_rate)
- Each request consumes one token
- If no tokens remain, the request is rejected
- The bucket has a maximum capacity (limit)

### Batch Leasing Optimization

Instead of fetching one token at a time from Redis, we fetch batches:
- Reduces Redis network round trips
- 99% reduction in overhead for batch_size=100
- Maintains accuracy while improving throughput

### Thread Safety Model

The implementation ensures thread-safety through:
- Per-ticker locks to avoid contention across different rate limits
- Registry lock to safely initialize new ticker entries
- Lua scripting for atomic operations at the Redis level