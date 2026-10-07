## The Three Pillars of Node.js
1. **V8 Engine:** Google's JS engine. Compiles JS to machine code. **Strictly single-threaded** (one Call Stack, one Memory Heap).
2. **Node APIs (Bindings):** The C++ bridge. Allows JavaScript to interact with the OS (file system, network, crypto).
3. **libuv:** A C library providing the asynchronous, non-blocking I/O architecture. Contains the **Event Loop** and **Worker Thread Pool**.

## Non-Blocking I/O in a Single Thread
Node.js avoids freezing the main thread by offloading work:
- **Network I/O:** Handled asynchronously by the OS kernel (e.g., epoll).
- **File System / Crypto:** Offloaded to libuv's **Worker Thread Pool** (background threads).
- *Crucial Note:* Wrapping heavy CPU computation (like a massive loop) in a Promise does **not** make it non-blocking. CPU-bound work always blocks the main V8 thread unless moved to a `worker_thread`.

## Event Loop Phases
The Event Loop executes in sequential phases:
1. **Timers:** `setTimeout`, `setInterval`.
2. **Pending Callbacks:** Deferred OS I/O callbacks.
3. **Poll:** Retrieves new I/O events (HTTP requests, file reads). *The loop will pause here and wait if queues are empty.*
4. **Check:** `setImmediate`.
5. **Close:** `socket.on('close')`.

## The Microtask Queue (VIP Priority)
Executed immediately *after* the current operation completes, before the Event Loop moves to the next phase.
1. `process.nextTick()` (Highest priority)
2. Promises (`.then`, `.catch`)

## 🎤 Interview Q&A
**Q: Node.js is single-threaded. How does it handle concurrent I/O operations without blocking?**
**A:** While V8 executes JS on a single main thread, I/O is handled by libuv in C++. For network operations, libuv delegates to the OS kernel's async polling. For file system or crypto tasks, libuv offloads them to a background worker thread pool. When these tasks finish, their callbacks are pushed to the event loop to be executed by the main thread once the call stack is empty.


1. **What happens to your Express server if you run a computationally heavy, synchronous block of code (like calculating primes for 10 seconds) inside a route handler?**
    
2. **If you run `setTimeout(fn, 0)` and `process.nextTick(fn)` at the exact same time, which one will execute first, and why?**
    
3. **If Node.js is "single-threaded," how does it manage to read a 2GB file from the hard drive without stopping other users from loading the website?**

**1. The Heavy Sync Code & The Promise Misconception** You were **100% correct** that the server will freeze. However, wrapping heavy computational code in a Promise **will NOT** stop it from blocking the main thread. This is a very common and dangerous misconception in Node.js.

- **Why Promises don't fix CPU blocking:** Promises are designed to handle _I/O-bound_ waiting (like waiting for a database to reply or a file to be read by libuv). But the JavaScript code _inside_ the Promise still executes on V8's single main thread. If you put a 10-second math calculation inside a Promise, V8 still has to do that math on the main thread, freezing everything else.
    
- **The actual solution:** To unblock CPU-heavy tasks (like processing images or calculating primes), you must move the work to a completely separate thread using Node's `worker_threads` module, or run it in a separate background process.
    

**2. process.nextTick vs setTimeout** **Spot on.** `process.nextTick` runs first. The Event Loop completely drains the Microtask Queue (where `nextTick` lives) _before_ it even looks at the Timers phase (where `setTimeout` lives).

**3. Reading a 2GB File** **Excellent answer.** You hit two massive concepts here. First, libuv offloads the actual file system read to the background Worker Thread Pool so the main thread isn't blocked. Second, you mentioned **streams** (chunking data at ~64kb). If you tried to read a 2GB file into memory all at once using `fs.readFile`, V8 would crash because it exceeds the default memory limit. Streaming is exactly how senior backend engineers handle large data.



[[BACKEND ENGINEERING]]