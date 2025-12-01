# Architecture Decision Record (ADR)
**Status:** Approved | **Date:** 2025-12-01 | **Author:** Architect Zero

## ADR-001: Adoption of Java Panama (FFM) vs. `sun.misc.Unsafe`

### Context
To achieve <1µs latency for the HyperStream ingestor, we required manual memory management outside the JVM Garbage Collector. The traditional approach in High-Frequency Trading (HFT) is to use `sun.misc.Unsafe` for raw pointer arithmetic.

### The Decision
We have chosen to **REJECT** `sun.misc.Unsafe` and **ADOPT** the new **Java Foreign Function & Memory (FFM) API (Project Panama)**.

### Rationale
1.  **Lifecycle Safety:** `Unsafe` allows wild pointer access which can crash the JVM (SIGSEGV). Panama uses `Arena` scopes (Confined/Shared) which guarantee temporal safety. If an Arena is closed, the memory cannot be accessed, preventing "Use-After-Free" vulnerabilities.
2.  **Future Proofing:** `Unsafe` is being deprecated in future JDKs. Building a billion-dollar asset on deprecated APIs is a strategic error.
3.  **Optimization:** The C2 JIT Compiler optimizes `MemorySegment` access patterns better than raw `Unsafe` calls in JDK 21+, allowing for superior loop unrolling and vectorization (AVX-512) support.

### Consequences
* **Positive:** We gain "Rust-like" memory safety semantics within Scala.
* **Negative:** Requires JDK 21+ runtime, limiting backward compatibility (Acceptable: Our target market runs modern infra).

---

## ADR-002: Rejection of Akka Streams for Core Loop

### Context
Scala's ecosystem defaults to Akka/Pekko Streams for data handling.

### The Decision
We **REJECT** Akka Streams for the hot-path and implement a custom **Lock-Free Ring Buffer**.

### Rationale
* Akka Streams allocates `Envelope` objects for every message. At 40M ops/sec, this creates huge GC pressure.
* Our Ring Buffer allocates **zero objects** during the steady state. We reuse the same off-heap memory slots cyclically.
