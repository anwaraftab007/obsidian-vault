## Single-Threaded Limitation
By default, Node.js runs on a single CPU core. If deployed on a 16-core server without scaling strategies, 15 cores will sit idle. 

## Type 1: High Web Traffic (I/O Bound) -> Use Clustering (PM2)
To handle massive amounts of concurrent HTTP requests, we scale horizontally across cores using the `cluster` module (typically managed via PM2).
- **How it works:** PM2 forks the Express app into multiple identical, isolated processes (one per core) and load-balances incoming requests among them.
- **The Memory Trap:** Because these processes are isolated, they do not share memory. You cannot store state (like user sessions or active web sockets) in simple JavaScript arrays or objects. You must use an external, centralized store like Redis.

## Type 2: Heavy Computation (CPU Bound) -> Use Worker Threads
For tasks that require heavy CPU calculation (video processing, PDF generation, cryptography), clustering is the wrong tool.
- **How it works:** The `worker_threads` module spawns a separate V8 environment *within* the same Node process.
- **The Benefit:** The heavy math runs on a separate background thread/core. The main Event Loop remains completely unblocked and continues to serve other users' HTTP requests rapidly.

### Interview Answer

**Question:** _"How do you scale a Node.js application to handle heavy web traffic versus heavy computational tasks?"_ **Answer:** _"To handle high web traffic, Node scales horizontally across CPU cores using the `cluster` module, typically managed by a process manager like PM2. This forks the application into multiple isolated processes and load-balances incoming HTTP requests across them. However, for heavy CPU-bound tasks like video processing, clustering isn't the right fit. Instead, I would use the `worker_threads` module to spawn an isolated V8 environment on a separate thread to handle the computation, preventing the main event loop from freezing while it processes."_


### Knowledge Check

Reply with your answers to complete Phase 4:

1. **You are building a chat application. You decide to store the list of active chat rooms in a simple array: `const activeRooms = []`. You then deploy this app using PM2 across 4 CPU cores. Why will this cause a massive bug for your users?**
    
2. **A client needs to generate a massive, complex 500-page PDF report when they click "Download". Should you handle this PDF generation using PM2 Clustering or `worker_threads`? Why?**

**1. PM2 and Memory Isolation (State)** Exactly right! Because PM2 forks the Node process, each of the 4 cores gets its own completely isolated V8 engine and memory space. If a user connects to Core 1 and is added to `activeRooms`, Core 2 has absolutely no idea that room exists. To fix this, you must store state externally (like in a Redis database) so all 4 processes can query the same source of truth.

**2. PDF Generation** Perfect answer. PDF generation is pure CPU math. Offloading it to a `worker_thread` keeps the main event loop completely free to handle other users' HTTP requests.



[[BACKEND ENGINEERING]]