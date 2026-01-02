# Event Loop Architecture

## Threading Model
* **Synchronous Code**: Runs on the Main Thread (V8).
* **Asynchronous I/O**: Offloaded to **Libuv**.
* **Network I/O**: Handled directly by the OS Kernel (`epoll`, `kqueue`) whenever possible (non-blocking).
* **CPU-Intensive Async**: Handled by the **Libuv Thread Pool** (default 4 threads). This includes:
    * File System APIs (`fs`)
    * Cryptography (`crypto`)
    * Compression (`zlib`)
    * DNS Lookups (`dns.lookup`)

## Phases (Macro Tasks)
The loop executes these phases sequentially:
1.  **Timers**: Executes `setTimeout` and `setInterval` callbacks.
2.  **Poll**: Executes I/O callbacks (incoming data, file read returns). If the queue is empty, it calculates how long to block and wait for I/O.
3.  **Check**: Executes `setImmediate` callbacks.
4.  **Close**: Executes close events (e.g., `socket.on('close')`).

## Microtasks & nextTick (The Priority Queues)
These are not part of the phases. They run **immediately after any currently executing operation completes**, before the event loop continues.
* **Priority 1**: `process.nextTick()` queue (Runs first).
* **Priority 2**: Promise `then`/`catch`/`finally` queue (Runs after `nextTick` is empty).

---

## ⚠️ Advanced Patterns & Gotchas

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
* **Loop Lag:** The time it takes for the Event Loop to complete one full cycle.
* **Impact:** High lag (>100ms) means the CPU is blocked by synchronous code (e.g., `JSON.parse` on a large payload), causing high API latency.
* **Detection:** Use Node.js `perf_hooks` to measure and alert on lag.

```javascript
const { monitorEventLoopDelay } = require('perf_hooks');

// Create a histogram to sample the loop delay every 20ms
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
    // Get the 99th percentile lag in milliseconds (P99)
    let percentileLag = h.percentile(99) / 1e6;
    
    console.log(`P99 Lag: ${percentileLag.toFixed(2)}ms`);

    if (percentileLag > 100) {
        console.error("🚨 ALERT: The main thread is blocked! Potential CPU bottleneck.");
    }
}, 5000);
```

<p align="center">
  <br>
  <img src="./assets/Figure1.png" alt="Event Loop Architecture" width="700">
  <br>
  <em>Figure 1: High-level architecture of the Node.js Runtime (V8 + Libuv).</em>
</p>
