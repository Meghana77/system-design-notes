# Event Loop Architecture

## Threading Model
**Synchronous Code**: Runs on the Main Thread (V8).
**Asynchronous I/O**: Offloaded to Libuv.
**Network I/O**: Handled directly by the OS Kernel (epoll/kqueue) whenever possible (non-blocking).
**CPU-Intensive Async**: Handled by the Libuv Thread Pool (default 4 threads). This includes:

    *File System APIs (fs)
    *Cryptography (crypto)
    *Compression (zlib)
    *DNS Lookups (dns.lookup)

## Phases (Macro Tasks) The loop executes these phases sequentially:
**Timers**: Executes setTimeout and setInterval callbacks.
**Poll**: Executes I/O callbacks (incoming data, file read returns). If the queue is empty, it calculates how long to block and wait for I/O.
**Check**: Executes setImmediate callbacks.
**Close**: Executes close events (e.g., socket.on('close')).

## Microtasks & nextTick (The Priority Queues) These are not part of the phases. They run immediately after any currently executing operation completes, before the event loop continues.
**Priority 1**: `process.nextTick()` queue (Runs first).
**Priority 2**: Promise then/catch/finally queue (Runs after nextTick is empty).


## Advanced Patterns & Gotchas

### 1. `setImmediate` vs `setTimeout(0)`
* **Context Matters:**
    * **Main Module:** The order is random (non-deterministic) because it depends on system performance at startup.
    * **Inside I/O Callback:** `setImmediate` is **guaranteed** to run before `setTimeout`.
    * *Reason:* Inside I/O, the loop is in the **Poll** phase. The next phase is **Check** (setImmediate), whereas **Timers** requires a full loop iteration.

### 2. Starvation (`process.nextTick` vs `setImmediate`)
* **`process.nextTick`:** The "Priority Queue." It runs *immediately* after the current operation.
    * 🛑 **Danger:** If you recursively call `nextTick`, the Event Loop will **never** proceed to the next phase. I/O will starve, and the server will freeze.
* **`setImmediate`:** Runs in the **Check** phase.
    * ✅ **Safe:** Even if recursive, it allows the Event Loop to complete a full tick (handling I/O and timers) before running again.

### 3. Monitoring "Event Loop Lag"
* As a Senior Engineer, we don't just write code; we monitor health.
* **Loop Lag:** The time it takes for the Event Loop to complete one full cycle.
* **Impact:** High lag (>100ms) means the CPU is blocked by synchronous code (e.g., `JSON.parse` on a large payload), causing high API latency.
* **Detection:** Use Node.js `perf_hooks` to measure and alert on lag.

<p align="center">
  <img src="./assets/event-loop-architecture.png" alt="Event Loop Architecture" width="700">
  <br>
  <em>Figure 1: High-level architecture of the Node.js Runtime (V8 + Libuv).</em>
</p>