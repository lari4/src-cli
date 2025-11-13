# AI Prompts Documentation

## Overview

This document contains all AI prompts that were used in the src-cli application.

**IMPORTANT NOTE:** All AI prompts documented here were **removed from the codebase** in commit `ce79fc9` on June 19, 2025. They are documented here for historical reference and understanding of the benchmarking functionality that existed.

### Historical Context

The prompts were part of the **Cody Gateway Benchmarking Suite**, a tool used internally at Sourcegraph to:
- Test and benchmark Cody Gateway performance
- Compare streaming vs non-streaming completion endpoints
- Measure latency and throughput for different AI providers
- Validate code completion functionality

The benchmarking suite was removed as it was only needed for internal testing during the Cody Gateway development phase.

---

## Table of Contents

1. [Code Completion Prompts](#code-completion-prompts)
   - [Anthropic/Claude Code Completion](#anthropicclaude-code-completion)
   - [Fireworks/StarCoder Code Completion](#fireworksstarcoder-code-completion)

---

## Code Completion Prompts

### Anthropic/Claude Code Completion

**Category:** Code Completion / Benchmarking
**Provider:** Anthropic (Claude)
**Model:** claude-3-haiku-20240307
**Endpoint:** Cody Gateway `/v1/completions/anthropic-messages`
**Source File:** `cmd/src/gateway_benchmark_stream.go` (removed)
**Git Reference:** `git show ce79fc9~1:cmd/src/gateway_benchmark_stream.go`

#### Purpose

This prompt was used to benchmark **code completion latency** for the Anthropic Claude model through Cody Gateway. The prompt simulates a real-world scenario where a developer starts writing a `bubble_sort` function in Python, and the AI assistant needs to complete the implementation.

The benchmark measures:
- Time to first token (TTFT)
- Total completion time
- Token throughput
- Streaming vs non-streaming performance

#### Use Cases

1. **Performance Testing:** Measure response times for Claude-based code completions
2. **Load Testing:** Send thousands of concurrent requests to test Cody Gateway scalability
3. **Comparison:** Compare Claude performance against other models (e.g., StarCoder)
4. **Regression Testing:** Detect performance degradations in Cody Gateway updates

#### Configuration Options

- **Streaming:** `true` or `false` - whether to use Server-Sent Events (SSE) streaming
- **Max Tokens:** Configurable (default: 256) - maximum tokens to generate
- **Temperature:** 0.0 (deterministic) - for consistent benchmarking results

#### Prompt Structure

**Format:** Anthropic Messages API format
**Conversation:** User-Assistant pair to provide context

#### Prompt Code (Cody Gateway Format)

```json
{
    "model": "claude-3-haiku-20240307",
    "messages": [
        {
            "role": "user",
            "content": "def bubble_sort(arr):"
        },
        {
            "role": "assistant",
            "content": "Here is a bubble sort:"
        }
    ],
    "max_tokens": 256,
    "temperature": 0.0,
    "stream": true
}
```

#### Prompt Code (Sourcegraph Instance Format)

When testing against a Sourcegraph instance directly (instead of Cody Gateway), the format differs slightly:

```json
{
    "model": "anthropic::2023-06-01::claude-3-haiku",
    "messages": [
        {
            "speaker": "human",
            "text": "def bubble_sort(arr):"
        },
        {
            "speaker": "assistant",
            "text": "Here is a bubble sort:"
        }
    ],
    "maxTokensToSample": 256,
    "temperature": 0.0,
    "stream": true
}
```

**Key Differences:**
- `role` vs `speaker` field naming
- `content` vs `text` field naming
- `max_tokens` vs `maxTokensToSample` parameter naming
- Different model identifier format

#### Expected Response

The AI should complete the bubble sort implementation in Python. Example expected output:

```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(0, n - i - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
    return arr
```

#### Technical Implementation Details

**HTTP Endpoint:** `POST /v1/completions/anthropic-messages`
**Authentication:** Bearer token (Sourcegraph Dotcom user key)
**Headers:**
```
Content-Type: application/json
Authorization: Bearer <sgd-token>
X-Sourcegraph-Should-Trace: true
X-Sourcegraph-Feature: code_completions
```

**Benchmark Metrics Collected:**
- Request duration (start to completion)
- HTTP status codes
- Response trace IDs (X-Trace header)
- Success/failure rates
- Statistical analysis (p5, p75, p80, p95, median, average)

---

### Fireworks/StarCoder Code Completion

**Category:** Code Completion / Benchmarking
**Provider:** Fireworks AI
**Model:** StarCoder
**Endpoint:** Cody Gateway `/v1/completions/fireworks`
**Source File:** `cmd/src/gateway_benchmark_stream.go` (removed)
**Git Reference:** `git show ce79fc9~1:cmd/src/gateway_benchmark_stream.go`

#### Purpose

This prompt was used to benchmark **Fill-In-the-Middle (FIM) code completion** using the StarCoder model through Cody Gateway. StarCoder is specifically designed for code completion tasks and uses a special FIM token format to indicate where code should be inserted.

The prompt simulates a TypeScript file where the developer has written part of a function signature and needs the AI to complete the implementation.

#### Use Cases

1. **FIM Benchmarking:** Test Fill-In-the-Middle completion performance
2. **StarCoder Performance:** Measure StarCoder-specific completion latency
3. **Multi-Provider Comparison:** Compare StarCoder vs Claude for code completion
4. **Token Throughput:** Measure tokens per second for code generation

#### FIM Token Format

StarCoder uses special Unicode tokens to indicate the fill-in-the-middle positions:
- `<｜fim▁begin｜>` - Start of the entire context
- `<｜fim▁hole｜>` - Where the completion should be inserted
- `<｜fim▁end｜>` - End of the context

This format allows the model to understand surrounding code context and generate appropriate completions.

#### Configuration Options

- **Streaming:** `true` or `false` - whether to use Server-Sent Events (SSE) streaming
- **Max Tokens:** Configurable (default: 256) - maximum tokens to generate
- **Temperature:** 0.2 (slightly creative) - allows some variation while staying deterministic
- **Top K:** 0 (disabled) - no top-k sampling
- **Top P:** 0 (disabled) - no nucleus sampling
- **Stop Sequences:** Multiple tokens to halt generation at appropriate boundaries

#### Prompt Structure

**Format:** Fireworks Completions API format
**Context:** TypeScript file with FIM tokens indicating completion point

#### Prompt Code (Cody Gateway Format)

```json
{
    "model": "starcoder",
    "prompt": "#hello.ts<｜fim▁begin｜>const sayHello = () => <｜fim▁hole｜><｜fim▁end｜>",
    "max_tokens": 256,
    "stop": [
        "\n\n",
        "\n\r\n",
        "<｜fim▁begin｜>",
        "<｜fim▁hole｜>",
        "<｜fim▁end｜>, <|eos_token|>"
    ],
    "temperature": 0.2,
    "topK": 0,
    "topP": 0,
    "stream": true
}
```

#### Prompt Code (Sourcegraph Instance Format)

When testing against a Sourcegraph instance directly:

```json
{
    "model": "fireworks::v1::starcoder",
    "messages": [
        {
            "speaker": "human",
            "text": "#hello.ts<｜fim▁begin｜>const sayHello = () => <｜fim▁hole｜><｜fim▁end｜>"
        }
    ],
    "maxTokensToSample": 256,
    "stopSequences": [
        "\n\n",
        "\n\r\n",
        "<｜fim▁begin｜>",
        "<｜fim▁hole｜>",
        "<｜fim▁end｜>, <|eos_token|>"
    ],
    "temperature": 0.2,
    "topK": 0,
    "topP": 0,
    "stream": true
}
```

**Key Differences:**
- `prompt` vs `messages` array format
- `stop` vs `stopSequences` parameter naming
- `max_tokens` vs `maxTokensToSample` parameter naming
- Different model identifier format

#### Expected Response

The AI should complete the TypeScript arrow function implementation. Example expected outputs:

**Option 1 (Console log):**
```typescript
const sayHello = () => {
    console.log("Hello, World!");
}
```

**Option 2 (Return statement):**
```typescript
const sayHello = () => "Hello, World!"
```

**Option 3 (More elaborate):**
```typescript
const sayHello = (name: string = "World") => {
    return `Hello, ${name}!`;
}
```

#### Technical Implementation Details

**HTTP Endpoint:** `POST /v1/completions/fireworks`
**Authentication:** Bearer token (Sourcegraph Dotcom user key)
**Headers:**
```
Content-Type: application/json
Authorization: Bearer <sgd-token>
X-Sourcegraph-Should-Trace: true
X-Sourcegraph-Feature: code_completions
```

**Stop Sequences Explained:**
- `\n\n` - Stop at double newline (end of function block)
- `\n\r\n` - Stop at Windows-style line ending
- `<｜fim▁begin｜>`, `<｜fim▁hole｜>`, `<｜fim▁end｜>` - Stop if model generates FIM tokens
- `<|eos_token|>` - Stop at end-of-sequence token

**Benchmark Metrics Collected:**
- Request duration (start to completion)
- HTTP status codes
- Response trace IDs (X-Trace header)
- Success/failure rates
- Statistical analysis (p5, p75, p80, p95, median, average)

#### FIM vs Traditional Prompting

**Traditional Prompting:**
```
Generate code for: const sayHello = () =>
```

**FIM Prompting (StarCoder):**
```
#hello.ts<｜fim▁begin｜>const sayHello = () => <｜fim▁hole｜><｜fim▁end｜>
```

FIM provides better context awareness because:
1. Model sees both prefix and suffix
2. Better handles middle insertions (not just appending)
3. More accurate for real-world IDE scenarios
4. Can respect surrounding code structure

---

## Summary

### Total Prompts Documented: 2

| Category | Provider | Model | Use Case | Format | Status |
|----------|----------|-------|----------|--------|--------|
| Code Completion | Anthropic | Claude 3 Haiku | Python bubble sort | Messages API | Removed |
| Code Completion | Fireworks | StarCoder | TypeScript FIM | Completions API | Removed |

### Common Characteristics

All prompts shared these characteristics:
- **Purpose:** Benchmarking Cody Gateway performance
- **Metrics:** Latency, throughput, success rate
- **Streaming:** Configurable (SSE support)
- **Authentication:** Sourcegraph tokens
- **Tracing:** Enabled via X-Trace headers
- **Status:** Removed from codebase (June 2025)

### Removal Rationale

The prompts were removed in commit `ce79fc9` because:
1. Internal benchmarking tool only
2. Not part of public CLI functionality
3. Cody Gateway development phase completed
4. Maintenance overhead without active usage

### Accessing Historical Prompts

To view the original implementation:

```bash
# View the complete gateway benchmark stream file
git show ce79fc9~1:cmd/src/gateway_benchmark_stream.go

# View the gateway benchmark file
git show ce79fc9~1:cmd/src/gateway_benchmark.go

# View the removal commit
git show ce79fc9
```

---

## Appendix

### Related Commands (Historical)

These commands existed when the prompts were active:

```bash
# Benchmark streaming completions (both providers)
src gateway benchmark-stream --requests 50 --csv results.csv --sgd <token> --sgp <token>

# Benchmark with custom endpoints
src gateway benchmark-stream \
  --gateway http://localhost:9992 \
  --sourcegraph http://localhost:3082 \
  --sgd <token> \
  --sgp <token>

# Benchmark Fireworks/StarCoder specifically
src gateway benchmark-stream \
  --requests 250 \
  --provider fireworks \
  --stream \
  --max-tokens 50
```

### Performance Characteristics (Historical Data)

Based on the benchmarking implementation:

**Measured Metrics:**
- Average latency
- Median latency
- P5, P75, P80, P95 percentiles
- Total requests
- Successful requests
- Failed requests

**Output Formats:**
- Console table display
- CSV export (aggregate results)
- CSV export (request-level results with trace IDs)

### Architecture Notes

The benchmarking suite supported:
- Concurrent request execution (configurable count)
- Multiple endpoint types (HTTP, WebSocket)
- Both Cody Gateway and direct Sourcegraph instance testing
- Streaming and non-streaming modes
- Multiple AI providers (Anthropic, Fireworks)

