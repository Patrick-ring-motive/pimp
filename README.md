PIMP — Polling Inverse Messaging Protocol
Version: 0.3.0
Status: DRAFT
Encoding: Hybrid (Base64 / Identity)
1. Motivation
Serverless platforms frequently impose two critical constraints: they do not support persistent streaming connections (HTTP Chunked/SSE), and they often restrict response bodies to valid UTF-8 text strings.
PIMP enables incremental data delivery over these stateless transports by utilizing a shared buffer and a hybrid encoding strategy. This bypasses platform-level buffering and text-only restrictions while maintaining performance for text-based streams.
2. Roles & Actors
• Initiator: Starts the job (e.g., a Client or Webhook).
• Sender: The worker (e.g., Lambda, Cloudflare Worker) that produces data in chunks.
• Buffer: A stateless shared store (e.g., Redis, KV, D1) that holds chunks indexed by job_id.
• Consumer: The client that polls the Buffer to reconstruct the stream.
3. Encoding Strategies
PIMP uses two primary modes to balance platform safety with performance:
3.1 Base64 Mode (Binary-Safe)
Used for non-textual data (images, PDFs, binary assets). Data chunks are encoded as Base64 strings.
• Pros: 100% safe for all text-only transports.
• Cons: ~33% bandwidth overhead.
3.2 Identity Mode (Text Pass-through)
Used for textual data (JSON, Markdown, plain text). Data chunks are sent as raw UTF-8 strings.
• Pros: Zero overhead; no double-parsing.
• Cons: Only safe for text-based content.
4. Metadata Chunk (Index 0)
The first chunk (index 0) is reserved for job metadata. The Sender MUST append this chunk immediately upon job start, bypassing any batching or flush intervals.
Schema
5. Buffer Interface
Chunks are stored as strings. The buffer must support atomic appends and ordered reads.
Chunk Shape
6. Sender Behavior
Immediate Metadata Flush
The Metadata Chunk (index 0) MUST NOT be buffered. The Sender must write it to the buffer before beginning the primary workload.
Data Batching
Subsequent data chunks (index 1+) should be buffered locally and flushed when:
1.	Local buffer reaches sender.bufferSize.
2.	sender.flushInterval expires.
7. Configuration & Defaults
Implementations MUST provide the following defaults:
Property
Default
Description
job.ttl_ms
300000
5 minute buffer retention.
sender.bufferSize
10
Chunks per batch flush.
sender.flushInterval
250ms
Max wait before batch flush.
consumer.interval
500ms
Initial polling frequency.
consumer.minInterval
100ms
Fastest possible adaptive poll.
consumer.maxInterval
5000ms
Slowest possible adaptive poll.

8. Constraints & Non-Goals
Constraints
• Text-Safe Transport: Payloads must be representable as strings.
• Ordering: Chunk indices are the sole authority for ordering.
• Isolation: Initiator, Sender, and Consumer share no in-process state.
Non-Goals
• Transport Specification: Does not define HTTP endpoints or Auth.
• Backpressure: Does not define flow control from Consumer to Sender.
• Multiplexing: Each job_id is an independent session.
