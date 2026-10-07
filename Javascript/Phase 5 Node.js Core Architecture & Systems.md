## Topic 1: Module Systems & Loading Mechanics

Node.js supports two module architectures: CommonJS (synchronous, runtime execution) and ES Modules (asynchronous, static analysis).

### CommonJS Execution Lifecycle:

1. **Resolution:** Maps module identifier to absolute file path.
    
2. **Loading & Wrapping:** Wraps file contents inside `(function(exports, require, module, __filename, __dirname) { ... })`.
    
3. **Evaluation:** V8 runs the wrapper function and populates `module.exports`.
    
4. **Caching:** Stores exports in `require.cache` for instant lookup on future calls.
    

### Key Conceptual Differences:

- **CJS Primitive Copies:** Exporting primitives in CJS creates static snapshots. Mutating the variable inside the module later does not update consumers.
    
- **ESM Live Bindings:** Importing variables in ESM creates live references pointing directly to the module's exported bindings.
    

> **Interview Q:** _Why are `__dirname` and `require` available in CommonJS without importing them?_ **Answer:** "Before executing a CommonJS module, Node.js wraps the file inside a Module Wrapper Function, passing `exports`, `require`, `module`, `__filename`, and `__dirname` as parameters into the function scope at runtime."

### Knowledge Check: Topic 1

Analyze this code snippet involving two separate files in CommonJS:

JavaScript

```
// counter.js
let count = 0;

function increment() {
    count++;
}

module.exports = { count, increment };
```

JavaScript

```
// app.js
const { count, increment } = require('./counter');

console.log(count); 
increment();
console.log(count); 
```

1. **What will be logged to the console on line 4 and line 6 in `app.js`?**
    
2. **Explain why this happens based on how CommonJS exports values.**

The actual output is **`0` and `0`**.

### Why `count` remains `0`

1. **Copy-by-Value (Primitives):** When `module.exports = { count, increment }` runs in `counter.js`, V8 copies the **primitive value** (`0`) into the exported object.
    
2. **Destructuring Copy:** In `app.js`, `const { count } = require('./counter')` creates a local variable `count` and copies the number `0` into it.
    
3. **Isolated Scope:** When `increment()` runs, it increases `count` inside `counter.js`'s internal scope, but it **does not** update the snapshot value attached to `module.exports`, nor does it update the local `count` variable in `app.js`.
    

> **Contrast with ESM:** If this were written in ES Modules (`export let count = 0`), `import { count, increment }` would create a **live binding**. Calling `increment()` would cause the second `console.log(count)` to print `1`.


## Topic 2: Buffers, Binary Data & Streams

### Buffers

- **Off-Heap Allocation:** Buffers represent raw binary data allocated outside V8's Memory Heap directly in RAM via C++ bindings.
    
- **Garbage Collection Bypass:** Because Buffers live outside V8's heap, they are exempt from standard V8 Garbage Collection cycles, avoiding CPU/GC overhead during high-throughput I/O.
    

### Streams & Memory Optimization

Streams process data sequentially in discrete chunks (default highWaterMark of 64 KB for file streams, 16 KB for standard streams).

- **Types:** `Readable`, `Writable`, `Duplex`, `Transform`.
    
- **Memory Footprint:** Using `fs.readFile()` scales memory usage linearly $O(N)$ with file size. Using `fs.createReadStream()` keeps memory consumption constant $O(1)$ regardless of file size.
    
- **Backpressure:** Occurs when data production outpaces consumption. `.pipe()` and `stream.pipeline()` pause the readable source when internal write buffers exceed `highWaterMark`, preventing RAM bloat.
    

> **Interview Q:** _Why use streams instead of `fs.readFile` for large data payloads?_
> 
> **Answer:** "`fs.readFile` buffers the entire dataset into V8 RAM before processing, leading to high memory spikes and potential process crashes for large files. Streams process data in small fixed-size chunks (e.g., 64KB), maintaining a flat, predictable RAM footprint and automatically handling backpressure between slow consumers and fast producers."

### Knowledge Check: Topic 2

Consider an HTTP server endpoint that receives a 1 GB file upload and writes it to disk:

JavaScript

```
const http = require('http');
const fs = require('fs');

http.createServer((req, res) => {
    const writeStream = fs.createWriteStream('./upload.dat');
    
    req.pipe(writeStream);
    
    writeStream.on('finish', () => {
        res.end("Upload complete");
    });
}).listen(3000);
```

1. **Why does this code use minimal RAM even though it receives a 1 GB payload?**
    
2. **What role does `req` play in terms of stream types, and what role does `writeStream` play?**

- **Memory Efficiency:** Spot on. The data is transferred in small chunks (default 64 KB buffers) outside the V8 heap, keeping RAM usage low and flat regardless of file size.
    
- **Stream Roles:**
    
    - `req` is a **Readable Stream** (receiving incoming data from the client).
        
    - `writeStream` is a **Writable Stream** (writing incoming chunks to disk).
        
    - `req.pipe(writeStream)` pipes data directly from the Readable request stream into the Writable file stream, handling backpressure automatically.

## Topic 3: Worker Threads & Cluster Module

### CPU-Bound vs I/O-Bound Bottlenecks

- **I/O-Bound (Non-Blocking):** Handled natively by libuv background threads and non-blocking sockets.
    
- **CPU-Bound (Blocking):** Synchronous computation directly blocks the V8 Call Stack, preventing the Event Loop from processing queued I/O tasks.
    

### Scaling Architecture Options

#### 1. Cluster Module (Multi-Process Scaling)

- Spawns multiple instances of the Node.js process using `child_process.fork()`.
    
- Primary process accepts network traffic and load-balances incoming connections across worker processes using OS-level socket sharing / round-robin routing.
    
- **Use Case:** Scaling web servers across multi-core systems to increase concurrent I/O capacity.
    

#### 2. Worker Threads (`worker_threads`)

- Executes JS concurrently within the _same_ OS process on dedicated threads.
    
- Communicates with the main thread via MessagePorts (`postMessage`) or zero-copy shared memory (`SharedArrayBuffer`).
    
- **Use Case:** Delegating heavy CPU calculations (hashing, image transformation, complex mathematical modeling) off the main Event Loop thread.
    

> **Interview Q:** _When should you use Worker Threads instead of the Cluster module?_ **Answer:** "Use the Cluster module to scale HTTP request throughput across CPU cores by running multiple isolated process instances. Use Worker Threads within a process when a specific CPU-heavy algorithm (like video transcoding or encryption) threatens to block the main Event Loop thread."

### Knowledge Check: Topic 3

Imagine you are building a photo-sharing API in Node.js.

When a user uploads an image, your server must perform two steps:

1. Save the raw upload to a database directory (I/O operation).
    
2. Generate 5 resized image thumbnails (High CPU image processing).
    
3. **If you run step 2 directly in a standard Node.js request handler, what happens to other incoming user requests while the thumbnail is being generated?**
    
4. **Which feature (`cluster` or `worker_threads`) should you use specifically to handle the thumbnail resizing logic, and why?**

- Running heavy CPU computations (like image resizing) directly on the main thread blocks the V8 Call Stack. While processing, the Event Loop cannot handle any incoming TCP connections or process other HTTP requests, freezing the server for all users.
    
- `worker_threads` is the correct tool here because it offloads the heavy CPU calculation to a background thread within the same process, keeping the main Event Loop completely responsive to incoming HTTP requests.

## Topic 4: Production Error Handling & Diagnostics

### Error Categorization

- **Operational Errors:** Expected failures (network timeouts, disk full, bad input). Handle gracefully without restarting the process.
    
- **Programmer Errors:** Bugs causing undefined states (`TypeError`, null pointer access). Require process termination and supervisor restart.
    

### Process Stability Pattern

JavaScript

```
process.on('uncaughtException', (error) => {
    logger.fatal(error, 'Uncaught Exception thrown');
    server.close(() => {
        process.exit(1); // Allow supervisor (PM2/K8s) to restart process
    });
});
```

### Memory Leak Analysis Workflow

1. **Symptom:** Steady upward slope in `process.memoryUsage().heapUsed` (RSS) without dropping post-GC.
    
2. **Common Causes:** Unbounded global arrays/maps, detached closures holding scope references, unremoved event listeners (`EventEmitter.on`).
    
3. **Tooling:** Capture snapshots using `v8.getHeapSnapshot()` or `node --inspect`. Analyze Retainer trees in Chrome DevTools Memory profiler to trace root paths.
    

> **Interview Q:** _Why shouldn't you keep a Node process alive after an `uncaughtException`?_ **Answer:** "An `uncaughtException` means an unhandled error occurred in an unpredictable state. Attempting to keep the process running risks cascading bugs, memory leaks, and silent data corruption. The process should clean up resources and exit, relying on an external supervisor (PM2 or Kubernetes) to start a clean instance."

### Interview Answer: Production Error Handling & Memory Leaks

**Question:** _"How should you handle `uncaughtException` in a production Node.js service, and how do you debug memory leaks?"_

**Your Answer:**

> "When an `uncaughtException` occurs, the application state is considered corrupted and unpredictable. The best practice is to log the error, gracefully close open network connections and database pools, and execute `process.exit(1)`. A process supervisor like PM2 or Kubernetes should then immediately restart a fresh worker process.
> 
> To debug a memory leak, monitor `process.memoryUsage()` over time. If `heapUsed` steadily increases despite garbage collection, take heap snapshots using `v8.getHeapSnapshot()` or `node --inspect`. Loading these snapshots into Chrome DevTools allows you to inspect the Retainer tree to identify the root objects—such as uncleared global caches or dangling event listeners—that are holding memory allocations hostage."

### Final Knowledge Check for Phase 5

Consider this code snippet running in a long-running production server:

JavaScript

```
const EventEmitter = require('events');
const userEmitter = new EventEmitter();

function trackUserSession(userId) {
    const sessionData = { id: userId, payload: new Array(1000000).fill("data") };
    
    userEmitter.on('userActivity', () => {
        console.log(`Activity for user: ${sessionData.id}`);
    });
}

// Called every time a user logs in
app.post('/login', (req, res) => {
    trackUserSession(req.body.userId);
    res.send("Logged in");
});
```

1. **Why will running this server cause a severe Memory Leak under heavy traffic?**
    
2. **Which exact object is being retained in memory, and why can V8's Garbage Collector not sweep it?**

### Step-by-Step Leak Analysis

1. **Global Root Reference:** `userEmitter` is created at the top level of the file. It is a global object, meaning V8 considers it permanently **Reachable from the Root**.
    
2. **Closure Memory Capture:** Inside `trackUserSession`, the callback function `() => { console.log(sessionData.id); }` references `sessionData.id`. V8 creates a **Closure** in the Memory Heap to hold `sessionData`. Because `sessionData` contains the huge 1,000,000-element `payload` array, that entire array gets trapped in the closure!
    
3. **Dangling Listener:** Every time a user hits `/login`, `userEmitter.on('userActivity', ...)` pushes a **new** listener callback into `userEmitter`'s internal array of listeners.
    
4. **Why Garbage Collection Fails:** V8 traces references starting from the Global Root:
    
    ```
    Global Scope 
      └── userEmitter 
            └── Internal Listeners Array 
                  └── Callback Function 
                        └── [[Environment]] (Closure) 
                              └── sessionData 
                                    └── payload (1,000,000 elements)
    ```
    
    Because `userEmitter` never dies and old listeners are never removed with `.off()` or `.removeListener()`, the reference path stays alive **permanently**. The Garbage Collector can _never_ sweep `sessionData` or `payload`.
    

If 1,000 users log in, you leak 1,000 copies of that multi-megabyte array until Node crashes with an `Out of Memory` (OOM) error.

### How to Fix It:

If an event listener only needs to run once, use `.once()` so Node automatically removes the listener after execution:

JavaScript

```
// Automatically unbinds itself after running once
userEmitter.once('userActivity', () => {
    console.log(`Activity for user: ${sessionData.id}`);
});
```

Or manually remove the listener when the session ends using `userEmitter.removeListener()`.

[[Javascript]]