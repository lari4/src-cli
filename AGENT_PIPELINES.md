# Agent Pipelines Documentation

## Overview

This document describes all agent workflows and pipelines that existed in the src-cli application for AI benchmarking.

**IMPORTANT NOTE:** All pipelines documented here were **removed from the codebase** in commit `ce79fc9` on June 19, 2025. They are documented here for historical reference.

### Historical Context

The pipelines were part of the **Cody Gateway Benchmarking Suite**, designed to test and measure performance of Sourcegraph's AI gateway infrastructure. The suite supported multiple communication protocols (HTTP, WebSocket) and multiple AI providers (Anthropic, Fireworks).

---

## Table of Contents

1. [HTTP Streaming Completion Pipeline](#http-streaming-completion-pipeline)
   - [Anthropic/Claude Pipeline](#anthropicclaude-http-streaming-pipeline)
   - [Fireworks/StarCoder Pipeline](#fireworksstarcoder-http-streaming-pipeline)
2. [WebSocket Benchmark Pipeline](#websocket-benchmark-pipeline)
3. [HTTP Test Endpoint Pipeline](#http-test-endpoint-pipeline)
4. [Complete Benchmark Orchestration](#complete-benchmark-orchestration)

---

## Pipeline Categories

The benchmarking suite had **4 main pipeline types**:

1. **HTTP Streaming Completion** - Code completion with Server-Sent Events (SSE)
2. **WebSocket Ping/Pong** - Latency testing via WebSocket protocol
3. **HTTP Test Endpoint** - Basic HTTP connectivity testing
4. **Orchestration** - Coordinating multiple benchmarks with statistical analysis

---

## HTTP Streaming Completion Pipeline

### Anthropic/Claude HTTP Streaming Pipeline

**Purpose:** Benchmark code completion performance using Claude models through Cody Gateway
**Protocol:** HTTP/HTTPS with Server-Sent Events (SSE)
**Source:** `cmd/src/gateway_benchmark_stream.go` (removed)
**Entry Point:** `buildGatewayHttpEndpoint()` with provider="anthropic"

#### Pipeline Overview

This pipeline sends code completion requests to Cody Gateway and measures response times for Claude-based completions.

#### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Anthropic/Claude HTTP Streaming Pipeline             │
└─────────────────────────────────────────────────────────────────────────┘

User Input
   │
   ├─► Provider Selection: "anthropic"
   ├─► Max Tokens: 256 (configurable)
   ├─► Stream Mode: true/false
   ├─► Request Count: 1000 (configurable)
   └─► Authentication: Bearer token
       │
       ▼
┌──────────────────────────────────────┐
│  buildGatewayHttpEndpoint()          │
│  - Constructs endpoint URL           │
│  - Builds JSON request body          │
│  - Sets authentication headers       │
└──────────────────────────────────────┘
       │
       ▼
   Endpoint Configuration:
   ├─► URL: /v1/completions/anthropic-messages
   ├─► Model: claude-3-haiku-20240307
   └─► Prompt: Python bubble_sort starter
       │
       ▼
┌──────────────────────────────────────┐
│  benchmarkCodeCompletions()          │
│  - Loops N times (request count)     │
│  - Calls benchmarkCodeCompletion()   │
│  - Collects timing data              │
└──────────────────────────────────────┘
       │
       ▼ (for each request)
┌──────────────────────────────────────┐
│  benchmarkCodeCompletion()           │
│  Step 1: Start timer                 │
│  Step 2: Create HTTP POST request    │
│  Step 3: Set headers                 │
│         - Content-Type: json         │
│         - Authorization: Bearer      │
│         - X-Sourcegraph-Should-Trace │
│         - X-Sourcegraph-Feature      │
└──────────────────────────────────────┘
       │
       ▼
   HTTP Request Body:
   {
     "model": "claude-3-haiku-20240307",
     "messages": [
       {"role": "user", "content": "def bubble_sort(arr):"},
       {"role": "assistant", "content": "Here is a bubble sort:"}
     ],
     "max_tokens": 256,
     "temperature": 0.0,
     "stream": true/false
   }
       │
       ▼
┌──────────────────────────────────────┐
│  HTTP Client Execution               │
│  - POST to Cody Gateway              │
│  - Wait for response                 │
└──────────────────────────────────────┘
       │
       ▼
  Cody Gateway
  /v1/completions/anthropic-messages
       │
       ▼
  Anthropic API (Claude)
       │
       ▼
┌──────────────────────────────────────┐
│  Response Processing                 │
│  - Read response body (io.ReadAll)   │
│  - Check HTTP status code            │
│  - Extract X-Trace header            │
│  - Stop timer                        │
└──────────────────────────────────────┘
       │
       ▼
   Response Data:
   ├─► HTTP Status: 200 OK (success)
   ├─► Trace ID: X-Trace header value
   ├─► Completion: Generated Python code
   └─► Duration: time.Since(start)
       │
       ▼
┌──────────────────────────────────────┐
│  requestResult{}                     │
│  - duration: time.Duration           │
│  - traceID: string                   │
└──────────────────────────────────────┘
       │
       ▼
   Collect results from all requests
       │
       ▼
┌──────────────────────────────────────┐
│  calculateStats()                    │
│  - Sort durations                    │
│  - Calculate percentiles:            │
│    • p5  (5th percentile)            │
│    • p75 (75th percentile)           │
│    • p80 (80th percentile)           │
│    • p95 (95th percentile)           │
│  - Calculate median                  │
│  - Calculate average                 │
│  - Sum total duration                │
└──────────────────────────────────────┘
       │
       ▼
   Statistical Results:
   {
     "avg": time.Duration,
     "median": time.Duration,
     "p5": time.Duration,
     "p75": time.Duration,
     "p80": time.Duration,
     "p95": time.Duration,
     "total": time.Duration,
     "successful": int
   }
       │
       ▼
┌──────────────────────────────────────┐
│  Output Results                      │
│  - printResults() → Console table    │
│  - writeResultsToCSV() → CSV file    │
│  - writeRequestResultsToCSV()        │
│    → Request-level CSV with traces   │
└──────────────────────────────────────┘
       │
       ▼
   Final Output
```

#### Key Functions and Their Roles

| Function | Input | Output | Purpose |
|----------|-------|--------|---------|
| `buildGatewayHttpEndpoint()` | provider, tokens, stream | `httpEndpoint{}` | Constructs endpoint configuration |
| `benchmarkCodeCompletions()` | endpoint, count | `endpointResult`, `[]requestResult` | Orchestrates multiple requests |
| `benchmarkCodeCompletion()` | endpoint | `requestResult` | Executes single request with timing |
| `calculateStats()` | `[]time.Duration` | `Stats{}` | Computes statistical metrics |
| `toEndpointResult()` | name, stats, count | `endpointResult{}` | Formats results for output |
| `printResults()` | `[]endpointResult` | - | Displays console table |
| `writeResultsToCSV()` | filename, results | - | Exports aggregate CSV |
| `writeRequestResultsToCSV()` | filename, results | - | Exports detailed CSV |

#### Data Structures

```go
// Endpoint configuration
type httpEndpoint struct {
    url        string  // "https://gateway/v1/completions/anthropic-messages"
    authHeader string  // "Bearer <token>"
    body       string  // JSON request payload
}

// Single request result
type requestResult struct {
    duration time.Duration  // How long the request took
    traceID  string        // X-Trace header for debugging
}

// Aggregated statistics
type Stats struct {
    Avg    time.Duration
    P5     time.Duration
    P75    time.Duration
    P80    time.Duration
    P95    time.Duration
    Median time.Duration
    Total  time.Duration
}

// Final endpoint results
type endpointResult struct {
    name       string         // "gateway" or "sourcegraph"
    avg        time.Duration
    median     time.Duration
    p5         time.Duration
    p75        time.Duration
    p80        time.Duration
    p95        time.Duration
    successful int           // Number of successful requests
    total      time.Duration
}
```

#### Error Handling

The pipeline handles errors at multiple stages:

1. **Request Creation Errors**
   - Failed to create HTTP request
   - Returns `requestResult{0, ""}` (zero duration)

2. **Network Errors**
   - Connection failures
   - Timeout errors
   - Returns `requestResult{0, ""}` (zero duration)

3. **HTTP Status Errors**
   - Non-200 status codes
   - Logs error with response body
   - Returns `requestResult{0, ""}` (zero duration)

4. **Response Reading Errors**
   - Failed to read response body
   - Returns `requestResult{0, ""}` (zero duration)

Failed requests (duration = 0) are filtered out before calculating statistics.

#### Configuration Options

Users could configure:

```bash
--requests <N>          # Number of requests per endpoint (default: 1000)
--max-tokens <N>        # Max tokens to generate (default: 256)
--provider anthropic    # Select Anthropic/Claude provider
--stream               # Enable streaming mode (default: false)
--gateway <URL>        # Cody Gateway endpoint
--sgd <token>          # Sourcegraph Dotcom user key
```

#### Example Usage

```bash
# Test Claude completions with streaming
src gateway benchmark-stream \
  --provider anthropic \
  --stream \
  --requests 100 \
  --max-tokens 256 \
  --gateway https://cody-gateway.sourcegraph.com \
  --sgd <your-token>

# Export results to CSV
src gateway benchmark-stream \
  --provider anthropic \
  --requests 500 \
  --csv aggregate_results.csv \
  --request-csv detailed_results.csv \
  --sgd <your-token>
```

#### Performance Metrics Collected

For each benchmark run:
- **Request-level metrics:** Individual duration + trace ID for each request
- **Aggregate metrics:** p5, p75, p80, p95, median, average, total time
- **Success rate:** Percentage of successful completions
- **CSV exports:** Both aggregate and request-level data

---

