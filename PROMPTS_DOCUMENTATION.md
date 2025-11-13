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

