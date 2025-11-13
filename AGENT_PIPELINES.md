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

### Fireworks/StarCoder HTTP Streaming Pipeline

**Purpose:** Benchmark Fill-In-the-Middle (FIM) code completion using StarCoder
**Protocol:** HTTP/HTTPS with Server-Sent Events (SSE)
**Source:** `cmd/src/gateway_benchmark_stream.go` (removed)
**Entry Point:** `buildGatewayHttpEndpoint()` with provider="fireworks"

#### Pipeline Overview

This pipeline is structurally identical to the Anthropic pipeline but uses StarCoder's FIM (Fill-In-the-Middle) format for code completion. The key differences are in the prompt structure and model parameters.

#### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  Fireworks/StarCoder HTTP Streaming Pipeline            │
└─────────────────────────────────────────────────────────────────────────┘

User Input
   │
   ├─► Provider Selection: "fireworks"
   ├─► Max Tokens: 256 (configurable)
   ├─► Stream Mode: true/false
   ├─► Request Count: 1000 (configurable)
   └─► Authentication: Bearer token
       │
       ▼
┌──────────────────────────────────────┐
│  buildGatewayHttpEndpoint()          │
│  - Constructs endpoint URL           │
│  - Builds JSON with FIM tokens       │
│  - Sets authentication headers       │
└──────────────────────────────────────┘
       │
       ▼
   Endpoint Configuration:
   ├─► URL: /v1/completions/fireworks
   ├─► Model: starcoder
   └─► Prompt: TypeScript FIM with special tokens
       │
       ▼
┌──────────────────────────────────────┐
│  benchmarkCodeCompletions()          │
│  (Same as Anthropic pipeline)        │
│  - Loops N times                     │
│  - Collects timing data              │
└──────────────────────────────────────┘
       │
       ▼
   HTTP Request Body:
   {
     "model": "starcoder",
     "prompt": "#hello.ts<｜fim▁begin｜>const sayHello = () => <｜fim▁hole｜><｜fim▁end｜>",
     "max_tokens": 256,
     "stop": ["\n\n", "\n\r\n", "<｜fim▁begin｜>", ...],
     "temperature": 0.2,
     "topK": 0,
     "topP": 0,
     "stream": true/false
   }
       │
       ▼
  Cody Gateway
  /v1/completions/fireworks
       │
       ▼
  Fireworks API (StarCoder)
       │
       ▼
   Response: TypeScript code completion
       │
       ▼
   (Same processing as Anthropic pipeline)
   │
   ├─► Statistical analysis
   ├─► CSV export
   └─► Console output
```

#### Key Differences from Anthropic Pipeline

| Aspect | Anthropic/Claude | Fireworks/StarCoder |
|--------|------------------|---------------------|
| **Endpoint** | `/v1/completions/anthropic-messages` | `/v1/completions/fireworks` |
| **Model** | `claude-3-haiku-20240307` | `starcoder` |
| **Prompt Format** | Messages array (role/content) | Single prompt string with FIM tokens |
| **Language** | Python | TypeScript |
| **Temperature** | 0.0 (deterministic) | 0.2 (slightly creative) |
| **Stop Sequences** | None (implicit) | Explicit FIM tokens + newlines |
| **Sampling** | Default | topK=0, topP=0 (disabled) |
| **Use Case** | General completion | Fill-in-the-middle completion |

#### FIM Token Processing

StarCoder's FIM tokens structure the prompt:

```
Prefix: "#hello.ts<｜fim▁begin｜>const sayHello = () => "
Hole:   "<｜fim▁hole｜>"
Suffix: "<｜fim▁end｜>"
```

The model understands:
1. **Begin token** - Start of file context
2. **Hole token** - Where to insert completion
3. **End token** - End of file context

This allows the model to see code before AND after the cursor, producing more contextually accurate completions.

#### Example Request/Response Flow

**Request:**
```json
{
    "model": "starcoder",
    "prompt": "#hello.ts<｜fim▁begin｜>const sayHello = () => <｜fim▁hole｜><｜fim▁end｜>",
    "max_tokens": 256,
    "stop": ["\n\n", "\n\r\n", "<｜fim▁begin｜>", "<｜fim▁hole｜>", "<｜fim▁end｜>, <|eos_token|>"],
    "temperature": 0.2,
    "stream": false
}
```

**Response:**
```json
{
    "choices": [{
        "text": "{\n    console.log(\"Hello, World!\");\n}"
    }],
    "usage": { "completion_tokens": 15 }
}
```

**Final Code:**
```typescript
const sayHello = () => {
    console.log("Hello, World!");
}
```

#### Configuration Options

```bash
--requests <N>          # Number of requests per endpoint (default: 1000)
--max-tokens <N>        # Max tokens to generate (default: 256)
--provider fireworks    # Select Fireworks/StarCoder provider
--stream               # Enable streaming mode (default: false)
--gateway <URL>        # Cody Gateway endpoint
--sgd <token>          # Sourcegraph Dotcom user key
```

#### Example Usage

```bash
# Test StarCoder FIM completions
src gateway benchmark-stream \
  --provider fireworks \
  --stream \
  --requests 100 \
  --max-tokens 256 \
  --gateway https://cody-gateway.sourcegraph.com \
  --sgd <your-token>

# Compare StarCoder vs Claude
src gateway benchmark-stream \
  --provider fireworks \
  --requests 500 \
  --csv starcoder_results.csv \
  --sgd <your-token>
```

#### Performance Considerations

StarCoder vs Claude performance characteristics:

**StarCoder Advantages:**
- Faster for simple code completions
- Better for middle-of-line insertions
- More efficient tokenization for code

**Claude Advantages:**
- Better contextual understanding
- More natural language comprehension
- Handles complex multi-line completions better

Both pipelines collect identical metrics for fair comparison.

---

## WebSocket Benchmark Pipeline

**Purpose:** Test latency and reliability of WebSocket connections to Cody Gateway
**Protocol:** WebSocket (ws:// or wss://)
**Source:** `cmd/src/gateway_benchmark.go` (removed)
**Entry Point:** `benchmarkEndpointWebSocket()`

#### Pipeline Overview

This pipeline measures round-trip time for WebSocket ping/pong messages, testing connectivity and baseline latency without AI model involvement.

#### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        WebSocket Ping/Pong Pipeline                     │
└─────────────────────────────────────────────────────────────────────────┘

User Input
   │
   ├─► Endpoint URL (Gateway or Sourcegraph)
   ├─► Authentication headers
   ├─► Request Count: 1000 (configurable)
   └─► Special headers (optional)
       │
       ▼
┌──────────────────────────────────────┐
│  webSocketClient{}                   │
│  - URL: ws://endpoint/v2/websocket   │
│  - Headers: Auth + Trace             │
│  - Connection: initially nil         │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  benchmarkEndpointWebSocket()        │
│  Step 1: Check if connected          │
│  Step 2: Connect if needed           │
└──────────────────────────────────────┘
       │
       ▼ (if not connected)
┌──────────────────────────────────────┐
│  webSocketClient.reconnect()         │
│  - Close existing connection         │
│  - Dial WebSocket with headers       │
│  - Store response headers            │
└──────────────────────────────────────┘
       │
       ▼
   WebSocket Connection Established
   (Persistent for multiple requests)
       │
       ▼
┌──────────────────────────────────────┐
│  Benchmark Loop (N iterations)       │
│                                      │
│  For each request:                   │
│  1. Start timer                      │
│  2. Send "ping" message              │
│  3. Read response message            │
│  4. Validate "pong" response         │
│  5. Stop timer                       │
│  6. Handle errors (reconnect)        │
└──────────────────────────────────────┘
       │
       ▼
   Message Flow:

   Client                    Server
     │                         │
     ├──── "ping" ───────────►│
     │    (TextMessage)        │
     │                         │
     │◄──── "pong" ────────────┤
     │    (TextMessage)        │
     │                         │
     └─► Duration recorded     │
       │
       ▼
┌──────────────────────────────────────┐
│  Error Handling                      │
│  - Write error → Reconnect           │
│  - Read error → Reconnect            │
│  - Wrong response → Reconnect        │
│  - Failed requests return duration=0 │
└──────────────────────────────────────┘
       │
       ▼
   Collect all durations
       │
       ▼
┌──────────────────────────────────────┐
│  calculateStats()                    │
│  (Same as HTTP pipelines)            │
│  - Percentiles, median, average      │
└──────────────────────────────────────┘
       │
       ▼
   Statistical Results + Output
```

#### Key Functions

| Function | Purpose |
|----------|---------|
| `webSocketClient.reconnect()` | Establishes new WebSocket connection |
| `benchmarkEndpointWebSocket()` | Sends ping, receives pong, measures latency |
| `conn.WriteMessage()` | Sends text message over WebSocket |
| `conn.ReadMessage()` | Reads response from WebSocket |

#### Data Structure

```go
type webSocketClient struct {
    conn        *websocket.Conn  // Active WebSocket connection
    URL         string            // ws://endpoint/path
    reqHeaders  http.Header       // Headers sent during handshake
    respHeaders http.Header       // Headers received from server
}
```

#### Message Protocol

Very simple ping/pong protocol:

```
Client sends:    "ping"   (TextMessage)
Server responds: "pong"   (TextMessage)
```

Any response other than "pong" is considered an error and triggers reconnection.

#### Connection Management

**Connection Lifecycle:**
1. **Initial Connection:** First request establishes WebSocket
2. **Persistent:** Connection reused for all subsequent requests
3. **Auto-Reconnect:** On any error, close and reconnect
4. **Cleanup:** Connection closed when benchmark completes

**Reconnection Triggers:**
- Write errors
- Read errors
- Invalid "pong" response
- Any WebSocket protocol errors

#### Configuration Options

```bash
--requests <N>              # Number of ping/pong cycles (default: 1000)
--gateway <URL>            # Cody Gateway WebSocket endpoint
--sourcegraph <URL>        # Sourcegraph WebSocket endpoint
--sgp <token>              # Sourcegraph personal access token
--use-special-header       # Add special test header
```

#### Example Usage

```bash
# Test WebSocket latency to Cody Gateway
src gateway benchmark \
  --gateway https://cody-gateway.sourcegraph.com \
  --requests 500 \
  --sgp <your-token>

# Test Sourcegraph instance
src gateway benchmark \
  --sourcegraph https://sourcegraph.com \
  --requests 1000 \
  --sgp <your-token>

# Compare both endpoints
src gateway benchmark \
  --gateway https://cody-gateway.sourcegraph.com \
  --sourcegraph https://sourcegraph.com \
  --csv comparison.csv \
  --sgp <your-token>
```

#### Endpoints Tested

| Name | URL Pattern | Purpose |
|------|-------------|---------|
| Gateway WebSocket | `ws(s)://gateway/v2/websocket` | Cody Gateway connectivity |
| Sourcegraph WebSocket | `ws(s)://instance/.api/gateway/websocket` | Sourcegraph instance connectivity |

#### Performance Metrics

WebSocket ping/pong typically shows:
- **Lower latency** than HTTP (persistent connection)
- **More consistent** response times (no connection overhead)
- **Faster p95/p99** percentiles (fewer outliers)

Use this to establish baseline connectivity performance before testing AI completions.

---

## HTTP Test Endpoint Pipeline

**Purpose:** Test basic HTTP connectivity without AI model processing
**Protocol:** HTTP/HTTPS
**Source:** `cmd/src/gateway_benchmark.go` (removed)
**Entry Point:** `benchmarkEndpointHTTP()`

#### Pipeline Overview

Simple HTTP GET/POST request to test endpoints, measuring pure network latency and gateway responsiveness without AI workloads.

#### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HTTP Test Endpoint Pipeline                        │
└─────────────────────────────────────────────────────────────────────────┘

User Input
   │
   ├─► Endpoint URL
   ├─► Authentication token
   └─► Request Count
       │
       ▼
┌──────────────────────────────────────┐
│  benchmarkEndpointHTTP()             │
│  - Start timer                       │
│  - Create HTTP POST request          │
│  - Set headers (Auth, Trace)         │
│  - Execute request                   │
│  - Read response body                │
│  - Stop timer                        │
└──────────────────────────────────────┘
       │
       ▼
   Request Headers:
   ├─► Authorization: token <sgp-token>
   ├─► X-Sourcegraph-Should-Trace: true
   └─► cody-core-gc-test: <value> (optional)
       │
       ▼
   HTTP POST → Test Endpoint
   (No AI processing, just connectivity check)
       │
       ▼
   Response (200 OK)
       │
       ▼
   requestResult{duration, traceID}
       │
       ▼
   Statistical Analysis
```

#### Endpoints Tested

| Name | URL Pattern |
|------|-------------|
| Gateway HTTP | `http(s)://gateway/v2/http` |
| Sourcegraph HTTP | `http(s)://instance/.api/gateway/http` |
| HTTP-then-WS | `http(s)://instance/.api/gateway/http-then-websocket` |

#### Purpose of Each Endpoint

1. **`/v2/http`** - Standard HTTP test endpoint
2. **`/.api/gateway/http`** - Sourcegraph HTTP gateway test
3. **`/.api/gateway/http-then-websocket`** - Tests HTTP→WebSocket upgrade capability

#### Use Case

This pipeline helps identify:
- Network latency issues
- Authentication problems
- Gateway availability
- Baseline performance before adding AI workload

Useful for isolating whether performance issues are network-related or AI-model-related.

---

## Complete Benchmark Orchestration

**Purpose:** Coordinate multiple pipelines to comprehensively test Cody Gateway
**Source:** `cmd/src/gateway_benchmark.go` and `cmd/src/gateway_benchmark_stream.go` (removed)
**Commands:** `src gateway benchmark` and `src gateway benchmark-stream`

### Overview

The benchmarking suite could run multiple pipelines concurrently or sequentially to compare performance across:
- Different endpoints (Gateway vs Sourcegraph)
- Different protocols (HTTP vs WebSocket)
- Different AI providers (Anthropic vs Fireworks)
- Different modes (Streaming vs Non-streaming)

### Complete Orchestration Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Complete Benchmark Orchestration                      │
└─────────────────────────────────────────────────────────────────────────┘

CLI Command: src gateway benchmark-stream
   │
   ├─► Parse flags
   │   ├─► --requests <N>
   │   ├─► --provider <anthropic|fireworks>
   │   ├─► --stream <true|false>
   │   ├─► --gateway <URL>
   │   ├─► --sourcegraph <URL>
   │   ├─► --sgd <dotcom-token>
   │   ├─► --sgp <instance-token>
   │   ├─► --csv <filename>
   │   └─► --request-csv <filename>
   │
   ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 1: Endpoint Configuration                                    │
│                                                                      │
│  IF --gateway specified:                                            │
│    Build Cody Gateway endpoint configuration                       │
│    ├─► URL: gateway/v1/completions/{provider}                      │
│    ├─► Auth: Bearer <sgd-token>                                    │
│    └─► Prompt: Provider-specific (see prompts doc)                 │
│                                                                      │
│  IF --sourcegraph specified:                                        │
│    Build Sourcegraph endpoint configuration                        │
│    ├─► URL: instance/.api/completions/{type}                       │
│    ├─► Auth: token <sgp-token>                                     │
│    └─► Prompt: Provider-specific (see prompts doc)                 │
└──────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 2: Execute Benchmarks (Sequential)                           │
│                                                                      │
│  For each configured endpoint:                                      │
│    ┌────────────────────────────────────────────────────┐          │
│    │  benchmarkCodeCompletions()                        │          │
│    │  - Initialize results arrays                       │          │
│    │  - Loop N times (--requests count)                 │          │
│    │    ├─► benchmarkCodeCompletion()                   │          │
│    │    │   ├─► HTTP POST with prompt                   │          │
│    │    │   ├─► Wait for response                       │          │
│    │    │   └─► Record duration + trace ID              │          │
│    │    ├─► Collect requestResult                       │          │
│    │    └─► Filter out failures (duration = 0)          │          │
│    │  - Return all results                              │          │
│    └────────────────────────────────────────────────────┘          │
│                                                                      │
│  Results collected:                                                 │
│  - Gateway results: endpointResult + []requestResult                │
│  - Sourcegraph results: endpointResult + []requestResult            │
└──────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 3: Statistical Analysis                                      │
│                                                                      │
│  For each endpoint's results:                                       │
│    ┌────────────────────────────────────────────────────┐          │
│    │  calculateStats(durations)                         │          │
│    │  - Sort all durations                              │          │
│    │  - Calculate percentiles:                          │          │
│    │    • p5  = durations[len*0.05]                     │          │
│    │    • p75 = durations[len*0.75]                     │          │
│    │    • p80 = durations[len*0.80]                     │          │
│    │    • p95 = durations[len*0.95]                     │          │
│    │    • median = durations[len*0.50]                  │          │
│    │  - Calculate average (sum / count)                 │          │
│    │  - Sum total duration                              │          │
│    └────────────────────────────────────────────────────┘          │
│                                                                      │
│  Result: Stats{Avg, P5, P75, P80, P95, Median, Total}              │
└──────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 4: Output Results                                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────┐            │
│  │  printResults()                                    │            │
│  │  Display console table:                            │            │
│  │                                                     │            │
│  │  Endpoint    | Avg  | Median | p95  | Success     │            │
│  │  ------------|------|--------|------|-------------│            │
│  │  gateway     | 150ms| 140ms  | 250ms| 998/1000    │            │
│  │  sourcegraph | 180ms| 170ms  | 300ms| 995/1000    │            │
│  └────────────────────────────────────────────────────┘            │
│                                                                      │
│  IF --csv specified:                                                │
│    ┌────────────────────────────────────────────────────┐          │
│    │  writeResultsToCSV()                               │          │
│    │  Export aggregate statistics:                      │          │
│    │  endpoint,avg,median,p5,p75,p80,p95,total,success  │          │
│    └────────────────────────────────────────────────────┘          │
│                                                                      │
│  IF --request-csv specified:                                        │
│    ┌────────────────────────────────────────────────────┐          │
│    │  writeRequestResultsToCSV()                        │          │
│    │  Export individual request data:                   │          │
│    │  endpoint,request_num,duration_ms,trace_id         │          │
│    │  gateway,1,145,abc123                              │          │
│    │  gateway,2,152,def456                              │          │
│    │  ...                                                │          │
│    └────────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────────┘
   │
   ▼
  Benchmark Complete
```

### Multi-Endpoint Comparison Example

When both `--gateway` and `--sourcegraph` are specified, the orchestration runs both pipelines and compares results:

```
$ src gateway benchmark-stream \
  --provider anthropic \
  --gateway https://cody-gateway.sourcegraph.com \
  --sourcegraph https://sourcegraph.com \
  --requests 100 \
  --sgd <gateway-token> \
  --sgp <instance-token>

Output:
┌─────────────┬──────────┬────────┬───────┬─────────┐
│  Endpoint   │   Avg    │ Median │  p95  │ Success │
├─────────────┼──────────┼────────┼───────┼─────────┤
│  gateway    │  145ms   │ 140ms  │ 220ms │ 98/100  │
│  sourcegraph│  180ms   │ 175ms  │ 280ms │ 97/100  │
└─────────────┴──────────┴────────┴───────┴─────────┘

Analysis: Gateway is 24% faster on average
```

### Pipeline Selection Logic

```go
// Simplified orchestration logic
func handler(args []string) error {
    var endpoints = make(map[string]httpEndpoint)

    // Configure Gateway endpoint if specified
    if *gatewayEndpoint != "" {
        endpoints["gateway"] = buildGatewayHttpEndpoint(
            *gatewayEndpoint, *sgdToken, *maxTokens, *provider, *stream
        )
    }

    // Configure Sourcegraph endpoint if specified
    if *sgEndpoint != "" {
        endpoints["sourcegraph"] = buildSourcegraphHttpEndpoint(
            *sgEndpoint, *sgpToken, *maxTokens, *provider, *stream
        )
    }

    // Run benchmarks for all configured endpoints
    results := make([]endpointResult, 0)
    for name, endpoint := range endpoints {
        result := benchmarkCodeCompletions(name, endpoint, *requestCount)
        results = append(results, result)
    }

    // Output and export
    printResults(results)
    if *csvOutput != "" {
        writeResultsToCSV(*csvOutput, results)
    }

    return nil
}
```

### Comparison Matrix

The orchestration supports comparing:

| Dimension | Options |
|-----------|---------|
| **Endpoints** | Cody Gateway vs Sourcegraph instance |
| **Providers** | Anthropic/Claude vs Fireworks/StarCoder |
| **Protocols** | HTTP vs WebSocket |
| **Streaming** | Streaming vs Non-streaming |
| **Load** | 1 to N requests (configurable) |

### Data Flow Between Prompts

**Sequential Flow for Multi-Provider Comparison:**

```
User runs: src gateway benchmark-stream --provider anthropic --requests 100
   │
   ▼
1. Anthropic Pipeline Executes
   │
   ├─► Prompt: "def bubble_sort(arr):" → Claude
   ├─► Response: Python bubble sort implementation
   ├─► Duration: Recorded for each of 100 requests
   └─► Result: Stats{avg: 150ms, p95: 220ms, ...}
   │
   ▼
2. (If provider=fireworks was also run separately)
   │
   ├─► Prompt: "#hello.ts<｜fim▁begin｜>const sayHello..." → StarCoder
   ├─► Response: TypeScript function implementation
   ├─► Duration: Recorded for each of 100 requests
   └─► Result: Stats{avg: 120ms, p95: 180ms, ...}
   │
   ▼
3. Results Combined for Comparison
   │
   └─► Analysis: StarCoder 20% faster for simple completions
```

**No Data Sharing Between Prompts:**
- Each pipeline runs independently
- No context carries over between requests
- Each prompt is stateless (consistent for benchmarking)
- Results aggregated only at the end for comparison

### Error Aggregation

The orchestration handles errors gracefully:

```
Total requests: 1000
Successful:     987  (98.7%)
Failed:         13   (1.3%)

Failure breakdown:
- Network timeout:    8
- Non-200 response:   3
- Read errors:        2

Statistics calculated on 987 successful requests only.
```

### CSV Export Format

**Aggregate CSV (`--csv`):**
```csv
endpoint,avg_ms,median_ms,p5_ms,p75_ms,p80_ms,p95_ms,total_ms,successful,total_requests
gateway,145.2,140.1,95.3,180.5,200.2,220.8,143406,987,1000
sourcegraph,180.5,175.3,120.8,210.2,230.5,280.3,179295,985,1000
```

**Request-level CSV (`--request-csv`):**
```csv
endpoint,request_number,duration_ms,trace_id
gateway,1,145,trace-abc-123
gateway,2,152,trace-def-456
gateway,3,138,trace-ghi-789
sourcegraph,1,180,trace-jkl-012
...
```

### Performance Optimization

The orchestration optimizes for:

1. **Sequential Endpoint Testing** - Tests one endpoint at a time to avoid network contention
2. **Connection Reuse** - WebSocket connections persist across requests
3. **Failure Filtering** - Failed requests excluded from statistics
4. **Streaming Progress** - Console output updates during long benchmarks
5. **CSV Buffering** - Results buffered and written once at end

### Summary

The complete orchestration provides:
- **Flexibility:** Test any combination of endpoints/providers/protocols
- **Comprehensive Metrics:** p5, p75, p80, p95, median, average, total
- **Detailed Exports:** Both aggregate and request-level CSV data
- **Error Resilience:** Graceful handling of failures
- **Comparative Analysis:** Side-by-side performance comparison

This allows thorough performance testing of Cody Gateway infrastructure under various conditions.

---

## Appendix

### All Pipelines Summary

| Pipeline | Protocol | AI Involved | Prompt Used | Metrics Collected |
|----------|----------|-------------|-------------|-------------------|
| Anthropic HTTP Streaming | HTTP/SSE | Yes | Python bubble_sort | Latency, throughput, success rate |
| Fireworks HTTP Streaming | HTTP/SSE | Yes | TypeScript FIM | Latency, throughput, success rate |
| WebSocket Ping/Pong | WebSocket | No | N/A (ping/pong) | Connection latency |
| HTTP Test Endpoint | HTTP | No | N/A (connectivity) | Network latency |

### Command Reference

**Historical Commands (Removed):**

```bash
# Stream benchmarking
src gateway benchmark-stream [flags]
  --provider <anthropic|fireworks>
  --stream
  --requests <N>
  --max-tokens <N>
  --gateway <URL>
  --sourcegraph <URL>
  --sgd <token>
  --sgp <token>
  --csv <file>
  --request-csv <file>

# WebSocket/HTTP benchmarking
src gateway benchmark [flags]
  --requests <N>
  --gateway <URL>
  --sourcegraph <URL>
  --sgp <token>
  --csv <file>
  --request-csv <file>
  --use-special-header
```

### Removal Information

All pipelines documented in this file were removed in:
- **Commit:** `ce79fc9`
- **Date:** June 19, 2025
- **Reason:** Internal testing tool no longer needed
- **Files Removed:**
  - `cmd/src/gateway.go`
  - `cmd/src/gateway_benchmark.go`
  - `cmd/src/gateway_benchmark_stream.go`

To view the original implementation:
```bash
git show ce79fc9~1:cmd/src/gateway_benchmark_stream.go
git show ce79fc9~1:cmd/src/gateway_benchmark.go
```

