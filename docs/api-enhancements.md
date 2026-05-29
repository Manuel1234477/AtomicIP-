# API Enhancements: High Availability & Resilience

This document describes the four API enhancements implemented to improve reliability, observability, and performance under high load.

## Overview

The API server now includes:
1. **Fallback Endpoints** (#537) - High availability through RPC endpoint failover
2. **Request Tracing** (#538) - Distributed tracing for debugging and monitoring
3. **Error Recovery** (#539) - Automatic retry strategies with circuit breaker pattern
4. **Request Queuing** (#540) - Load management during high traffic periods

## 1. Fallback Endpoints (#537)

### Purpose
Support fallback RPC endpoints for high availability. If the primary endpoint fails, requests automatically route to fallback endpoints.

### Features
- **Primary + Fallback Configuration**: Define a primary endpoint and multiple fallback endpoints
- **Health Tracking**: Each endpoint tracks consecutive failures
- **Automatic Failover**: After 3 consecutive failures, an endpoint is marked unhealthy
- **Endpoint Recovery**: Healthy endpoints can be restored after recovery
- **Health Status API**: Query the health status of all endpoints

### Usage

```rust
use api_server::fallback::{FallbackConfig, FallbackManager};
use std::time::Duration;

let config = FallbackConfig {
    primary_endpoint: "https://soroban-testnet.stellar.org".to_string(),
    fallback_endpoints: vec![
        "https://soroban-testnet-backup1.stellar.org".to_string(),
        "https://soroban-testnet-backup2.stellar.org".to_string(),
    ],
    health_check_interval: Duration::from_secs(30),
    timeout: Duration::from_secs(5),
};

let manager = FallbackManager::new(config);

// Get the active endpoint
let endpoint = manager.get_active_endpoint();

// Mark endpoint as failed
manager.mark_failed(&endpoint);

// Mark endpoint as healthy
manager.mark_healthy(&endpoint);

// Get health status
let status = manager.get_health_status();
```

### Configuration

Add to `.env`:
```env
PRIMARY_RPC_ENDPOINT=https://soroban-testnet.stellar.org
FALLBACK_RPC_ENDPOINTS=https://soroban-testnet-backup1.stellar.org,https://soroban-testnet-backup2.stellar.org
HEALTH_CHECK_INTERVAL_SECS=30
RPC_TIMEOUT_SECS=5
```

## 2. Request Tracing (#538)

### Purpose
Implement distributed tracing for request debugging and monitoring. Trace requests across service boundaries.

### Features
- **Trace ID Generation**: Automatic UUID generation for each request
- **Span ID Tracking**: Unique span IDs for request segments
- **Parent-Child Relationships**: Support for distributed tracing hierarchies
- **W3C Traceparent Standard**: Compatible with standard tracing formats
- **Header Propagation**: Automatic trace context propagation in responses

### Usage

```rust
use api_server::distributed_tracing::{
    DistributedTraceContext, 
    distributed_tracing_middleware,
    get_trace_context,
};

// Middleware automatically extracts/generates trace context
// Access trace context in handlers:
let trace_context = get_trace_context(req.headers());
tracing::info!(
    trace_id = %trace_context.trace_id,
    span_id = %trace_context.span_id,
    "Processing request"
);
```

### Headers

**Request Headers:**
- `X-Trace-ID`: Trace ID (UUID format)
- `X-Span-ID`: Parent span ID (UUID format)

**Response Headers:**
- `X-Trace-ID`: Trace ID (echoed from request or generated)
- `X-Span-ID`: New span ID for this request

### Example

```bash
curl -H "X-Trace-ID: 550e8400-e29b-41d4-a716-446655440000" \
     -H "X-Span-ID: 660e8400-e29b-41d4-a716-446655440001" \
     http://localhost:8080/v1/ip/commit
```

## 3. Error Recovery (#539)

### Purpose
Implement automatic error recovery strategies including retry logic and circuit breaker pattern.

### Features
- **Exponential Backoff**: Configurable retry delays with exponential growth
- **Retryable Error Classification**: Automatic detection of retryable errors (5xx, timeouts)
- **Circuit Breaker Pattern**: Prevent cascading failures
- **Configurable Thresholds**: Customize failure thresholds and recovery behavior
- **Recovery Strategies**: Multiple strategies (Retry, CircuitBreaker, Fallback, Fail)

### Usage

```rust
use api_server::error_recovery::{
    RetryConfig, 
    CircuitBreaker,
    CircuitBreakerState,
    is_retryable_error,
    calculate_backoff,
};
use std::time::Duration;

// Configure retry behavior
let config = RetryConfig {
    max_retries: 3,
    initial_backoff: Duration::from_millis(100),
    max_backoff: Duration::from_secs(10),
    backoff_multiplier: 2.0,
};

// Calculate backoff for attempt 2
let backoff = calculate_backoff(2, &config);

// Circuit breaker
let mut cb = CircuitBreaker::new(
    3,  // failure threshold
    2,  // success threshold for recovery
);

cb.record_failure();
cb.record_failure();
cb.record_failure();
assert_eq!(cb.get_state(), CircuitBreakerState::Open);

// Attempt recovery
if cb.can_attempt() {
    // Try request
    cb.record_success();
}
```

### Retryable Errors

The following HTTP status codes trigger automatic retry:
- `408 Request Timeout`
- `429 Too Many Requests`
- `502 Bad Gateway`
- `503 Service Unavailable`
- `504 Gateway Timeout`

### Circuit Breaker States

1. **Closed**: Normal operation, requests pass through
2. **Open**: Too many failures, requests rejected immediately
3. **Half-Open**: Testing if service recovered, limited requests allowed

## 4. Request Queuing (#540)

### Purpose
Queue requests during high load to prevent server overload and ensure fair request handling.

### Features
- **Configurable Queue Size**: Limit total queued requests
- **Concurrency Control**: Semaphore-based concurrent request limiting
- **Request Timeout**: Configurable timeout for queued requests
- **Queue Statistics**: Monitor queue depth and wait times
- **Automatic Cleanup**: Guard pattern ensures queue cleanup

### Usage

```rust
use api_server::request_queue::{RequestQueue, QueueConfig};
use std::time::Duration;

let config = QueueConfig {
    max_queue_size: 1000,
    max_concurrent_requests: 100,
    request_timeout: Duration::from_secs(30),
};

let queue = RequestQueue::new(config);

// Acquire queue slot
match queue.acquire("request-123".to_string()).await {
    Ok(_guard) => {
        // Process request
        // Guard automatically removes from queue when dropped
    }
    Err(status) => {
        // Queue full or timeout
    }
}

// Get queue statistics
let stats = queue.get_stats();
println!("Queue size: {}/{}", stats.queue_size, stats.max_queue_size);
println!("Avg wait time: {:?}", stats.avg_wait_time);
```

### Configuration

Add to `.env`:
```env
MAX_QUEUE_SIZE=1000
MAX_CONCURRENT_REQUESTS=100
REQUEST_TIMEOUT_SECS=30
```

### Error Responses

**Queue Full:**
```json
{
  "error": "Service Unavailable",
  "status": 503
}
```

**Request Timeout:**
```json
{
  "error": "Request Timeout",
  "status": 408
}
```

## Integration

### Middleware Stack

The middleware stack should be ordered as:

```rust
.layer(middleware::from_fn(distributed_tracing_middleware))
.layer(middleware::from_fn(error_recovery_middleware))
.layer(middleware::from_fn(request_queue_middleware))
.layer(middleware::from_fn(compression_middleware))
```

### Environment Variables

```env
# Fallback Endpoints
PRIMARY_RPC_ENDPOINT=https://soroban-testnet.stellar.org
FALLBACK_RPC_ENDPOINTS=https://backup1.stellar.org,https://backup2.stellar.org
HEALTH_CHECK_INTERVAL_SECS=30
RPC_TIMEOUT_SECS=5

# Error Recovery
MAX_RETRIES=3
INITIAL_BACKOFF_MS=100
MAX_BACKOFF_SECS=10
BACKOFF_MULTIPLIER=2.0

# Request Queuing
MAX_QUEUE_SIZE=1000
MAX_CONCURRENT_REQUESTS=100
REQUEST_TIMEOUT_SECS=30
```

## Monitoring

### Metrics

Each module exposes metrics:

**Fallback Endpoints:**
- `api_fallback_endpoint_health` - Health status of each endpoint
- `api_fallback_failover_count` - Number of failovers

**Request Tracing:**
- `api_trace_requests_total` - Total traced requests
- `api_trace_duration_seconds` - Request duration histogram

**Error Recovery:**
- `api_retry_attempts_total` - Total retry attempts
- `api_circuit_breaker_state` - Current circuit breaker state
- `api_circuit_breaker_transitions` - State transitions

**Request Queuing:**
- `api_queue_size` - Current queue size
- `api_queue_wait_time_seconds` - Average wait time
- `api_queue_full_errors` - Queue full errors

### Logging

All modules use structured logging with trace context:

```json
{
  "timestamp": "2026-05-29T10:42:09Z",
  "level": "INFO",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "span_id": "660e8400-e29b-41d4-a716-446655440001",
  "message": "Request completed",
  "duration_ms": 125
}
```

## Testing

Each module includes comprehensive unit tests:

```bash
# Run all tests
cargo test -p api-server

# Run specific module tests
cargo test -p api-server fallback::tests
cargo test -p api-server distributed_tracing::tests
cargo test -p api-server error_recovery::tests
cargo test -p api-server request_queue::tests
```

## Performance Considerations

### Fallback Endpoints
- Health checks run asynchronously
- Minimal overhead for endpoint selection
- Recommended: 2-3 fallback endpoints

### Request Tracing
- UUID generation per request (~1-2 µs)
- Header parsing and propagation (~100 ns)
- Minimal performance impact

### Error Recovery
- Exponential backoff prevents thundering herd
- Circuit breaker prevents cascading failures
- Recommended: 3 retries with 2x multiplier

### Request Queuing
- Semaphore-based concurrency control
- O(1) queue operations
- Recommended: Queue size = 10x concurrent requests

## Troubleshooting

### All Endpoints Unhealthy
- Check network connectivity
- Verify endpoint URLs in configuration
- Check RPC endpoint status pages

### High Queue Wait Times
- Increase `MAX_CONCURRENT_REQUESTS`
- Optimize handler performance
- Consider horizontal scaling

### Circuit Breaker Stuck Open
- Check downstream service health
- Verify network connectivity
- Check error logs for root cause

## References

- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
- [Semaphore Pattern](https://en.wikipedia.org/wiki/Semaphore_(programming))
