## The Cost of Connections
A database connection is a TCP network socket. Opening a new connection per HTTP request is extremely slow (due to the TCP handshake) and can easily exhaust the database's maximum connection limits, crashing the system.

## Connection Pooling (The Solution)
Instead of opening/closing connections constantly, backend apps use a **Connection Pool**.
- **How it works:** On startup, the app opens a set number of persistent connections (e.g., 10). 
- When a request comes in, it "checks out" a connection from the pool.
- When the query finishes, the app "checks in" (releases) the connection back to the pool.
- **Queueing:** If the pool is empty (all connections are currently in use), incoming requests wait asynchronously in a queue until a connection is freed.

## Connection Leaks & Pool Exhaustion
If a developer forgets to release a connection back to the pool (especially inside an error handler), that connection is permanently lost to the app. Eventually, the entire pool is exhausted, and all subsequent database requests will hang infinitely.

## Async Database Patterns
Database queries are network I/O, meaning they must be handled asynchronously via libuv.
Using `await db.query(...)` pauses the execution of that specific Express route, allowing the main V8 thread to serve other incoming HTTP requests while waiting for the database to reply.


### Phase 3 Knowledge Check

Reply with your answers to these two questions to complete Phase 3:

1. **If your Node app has a connection pool size of 5, and 100 users hit a specific API route at the exact same millisecond, what happens to the remaining 95 requests? Do they crash the server?**
    
2. **If you forget to release/return a database connection back to the pool inside your Express route handler, what will eventually happen to your application as traffic increases?

**1. Queueing vs Crashing** You are exactly right. The remaining 95 requests wait in a queue asynchronously. Once a connection is freed up, the next request in line grabs it. The server doesn't crash; the requests just take a few milliseconds longer to respond.

**2. Connection Exhaustion** Perfectly stated. This is known as a **Connection Leak** resulting in **Pool Exhaustion**. If you don't release the connection back to the pool, the pool thinks it's still busy. Once all 5 connections are "leaked," the 6th request will wait in the queue forever (until it times out). The database is fine, but your Node app is starved of connections.


[[BACKEND ENGINEERING]]