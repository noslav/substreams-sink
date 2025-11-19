# Async Processing Feature

## Overview

The async processing feature prevents slow handler operations from blocking the gRPC receive loop, allowing the sink to receive blocks from the upstream substreams node without delays.

## Problem

Previously, the `HandleBlockScopedData` handler was called synchronously within the receive loop. If your handler performs slow operations (database writes, network calls, etc.), it blocks the next `stream.Recv()` call, causing delays in receiving new blocks from the upstream node.

## Solution

With async processing enabled, blocks are queued in a buffered channel and processed by a separate goroutine. This allows the receive loop to continue fetching blocks while your handler processes them concurrently.

## Usage

### Basic Example

```go
sinker, err := sink.New(
    sink.SubstreamsModeProduction,
    false,
    pkg,
    outputModule,
    hash,
    clientConfig,
    logger,
    tracer,
    backoff.NewExponentialBackOff(),
    sink.WithAsyncProcessing(50), // Enable async processing with buffer size of 50
)
```

### Channel Size Recommendations

- **Small (10-20)**: For handlers with low latency (< 10ms)
- **Medium (50-100)**: For handlers with moderate latency (10-100ms)
- **Large (100+)**: For handlers with high latency (> 100ms)

**Note**: Larger buffers use more memory. Choose based on your handler's latency and memory constraints.

## How It Works

1. The receive loop fetches blocks from the upstream node via `stream.Recv()`
2. Blocks are sent to a buffered channel for processing
3. A separate goroutine processes blocks from the channel by calling your handler
4. The receive loop waits for each block's processing to complete before continuing
5. Errors from the handler are propagated back to the main loop

## Benefits

- **Lower latency**: Receive loop isn't blocked by slow handler operations
- **Better throughput**: Can receive and process blocks in parallel
- **Maintains ordering**: Blocks are still processed in order
- **Error handling**: Errors from async processing are properly propagated

## Compatibility

- Works with or without buffer (`WithBlockDataBuffer`)
- Works with `WithFinalBlocksOnly`
- Compatible with all existing handler implementations
- No changes needed to your handler code

## Performance Considerations

1. **Backpressure**: If your handler is slower than the receive rate, the channel will fill up, and the receive loop will block when the channel is full
2. **Memory usage**: Each queued block consumes memory proportional to its size
3. **CPU usage**: Async processing adds minimal CPU overhead for goroutine scheduling
4. **Ordering**: Blocks are always processed in order (FIFO)

## Example: Before and After

### Before (Synchronous)
```
[Receive Block 1] → [Process Block 1] → [Receive Block 2] → [Process Block 2]
     10ms                100ms               10ms               100ms
Total: 220ms for 2 blocks
```

### After (Asynchronous)
```
[Receive Block 1] → [Receive Block 2] → [Receive Block 3]
     10ms     ↓         10ms      ↓          10ms
          [Process 1]         [Process 2]
            100ms               100ms
Total: 130ms for 3 blocks
```

## Migration Guide

### Step 1: Add the Option
```go
// Old code
sinker, err := sink.New(
    mode, noopMode, pkg, outputModule, hash,
    clientConfig, logger, tracer, backoff,
)

// New code
sinker, err := sink.New(
    mode, noopMode, pkg, outputModule, hash,
    clientConfig, logger, tracer, backoff,
    sink.WithAsyncProcessing(50), // Add this line
)
```

### Step 2: Test and Tune
1. Start with a channel size of 50
2. Monitor your logs for "async_processing=true"
3. Adjust the channel size based on your handler's latency
4. Monitor memory usage and adjust if needed

### Step 3: Deploy
No other changes are needed. Your handler code remains the same.

