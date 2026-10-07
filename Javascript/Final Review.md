## Phase 1: V8 Engine & Execution Mechanics

### 1. The V8 Compilation Pipeline

- **AST Generation:** V8 parses source code into an **Abstract Syntax Tree (AST)**.
    
      
    
- **Ignition (Interpreter):** Converts the AST into **Bytecode** for rapid initial execution. It collects type feedback during runtime.
    
      
    
- **TurboFan (JIT Compiler):** Identifies "hot functions" (frequently executed functions) and compiles the Bytecode into highly optimized **Machine Code** based on inline caches and hidden classes.
    
      
    
- **Deoptimization:** If variable types change dynamically at runtime, TurboFan invalidates the machine code and falls back to Ignition Bytecode.
    
      
    

### 2. Execution Context & Call Stack

- **Call Stack:** A LIFO (Last-In, First-Out) data structure that tracks active function calls.
    
      
    
- **Execution Context Structure:** Created during function invocation. Holds:
    
      
    1. **Variable Environment:** Storage for `var` declarations and function parameters.
        
          
        
    2. **Lexical Environment:** Storage for `let` and `const` bindings, maintaining a reference to the outer parent environment.
        
          
        
    3. **`this` Binding:** Dynamically assigned based on the call site.
        
          
        

## Phase 2: Scopes, Memory & Garbage Collection

### 1. Scope Chain & Closures

- **Lexical Scope:** Variable resolution is fixed at write-time based on physical placement in code, not run-time call stacks.
    
      
    
- **Closures (`[[Environment]]`):** When an inner function is created, V8 stores a reference to its outer lexical scope inside an internal slot named `[[Environment]]`. Even after the outer function's execution context is popped off the Call Stack, variables referenced by `[[Environment]]` remain allocated in the **Memory Heap**.
    
      
    

### 2. V8 Garbage Collection (Generational GC)

- **Scavenger (Young Generation):** Allocates new short-lived objects in a small memory space divided into _From-Space_ and _To-Space_. Surviving objects are promoted to the _Old Generation_.
    
      
    
- **Mark-and-Sweep (Old Generation):**
    
      
    1. **Marking:** Traces object references starting from **GC Roots** (Global Object, Call Stack variables).
        
          
        
    2. **Sweeping:** Unreachable objects without active root paths are cleared.
        
          
        
    3. **Compacting:** Defragments free memory chunks to optimize allocations.
        
          
        

## Phase 3: Context & Prototype Architecture

### 1. Dynamic `this` Binding Rules

`this` is determined dynamically at call-time across 4 strict rules:

  

1. **Implicit Binding:** `obj.method()` $\rightarrow$ `this` points to `obj`.
    
      
    
2. **Explicit Binding:** `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)` $\rightarrow$ `this` points to `obj`.
    
      
    
3. **Default Binding:** Standalone function invocation `fn()` $\rightarrow$ `this` points to `global` (`undefined` in strict mode). Detaching methods from objects causes loss of `this` context.
    
      
    
4. **Lexical Binding (Arrow Functions):** Arrow functions do not bind `this`; they inherit it lexically from their outer scope at creation time.
    
      
    

### 2. Prototype Delegation vs. Class Inheritance

- **Prototype Chain:** Lookups traverse `obj.__proto__` up to `Object.prototype.__proto__` (`null`).
    
      
    
- **Memory Optimization:** Method definitions attached to `Constructor.prototype` reside as **a single shared memory instance** in the Heap. Methods declared directly inside constructor bodies allocate duplicate function instances for every object instantiation, leading to RAM bloat.
    
      
    

## Phase 4: Asynchronous JS & The Event Loop

### 1. libuv Event Loop Phases

Macrotasks execute across 6 distinct libuv loop phases:

  

$$\text{Timers} \longrightarrow \text{Pending Callbacks} \longrightarrow \text{Idle/Prepare} \longrightarrow \text{Poll (I/O)} \longrightarrow \text{Check (\texttt{setImmediate})} \longrightarrow \text{Close Callbacks}$$

### 2. Task Queue Priorities

- **Microtask Draining Rule:** Microtasks are drained **completely** after synchronous code, after every single macrotask, and between **every single libuv phase transition**.
    
      
    
- **Priority Order:**
    
      
    
    $$\text{Synchronous Code} \longrightarrow \text{\texttt{process.nextTick()}} \longrightarrow \text{Promise Microtasks} \longrightarrow \text{Macrotasks (Timers / I/O / Check)}$$
    
- **`setImmediate` vs. `setTimeout(fn, 0)`:** Inside an I/O callback (Poll Phase), `setImmediate` is guaranteed to execute before `setTimeout` because the Check phase immediately follows the Poll phase.
    
      
    

### 3. Async/Await Internal Mechanics

- Marking a function as `async` wraps return values in `Promise.resolve()`.
    
      
    
- Reaching `await` pauses execution, pops the `async` context off the Call Stack, and converts the remaining downstream code into a **Promise Microtask continuation**.
    
      
    

## Phase 5: Node.js Core Architecture & Systems

### 1. Module Systems Mechanics

- **CommonJS (CJS):** Synchronous execution at runtime. Wraps modules in `(function(exports, require, module, __filename, __dirname){ ... })`. Exports primitive **copies** (snapshots) and caches exports in `require.cache`.
    
      
    
- **ES Modules (ESM):** Asynchronous static parsing prior to execution. Exports **live bindings** (dynamic variable references).
    
      
    

### 2. Buffers, Streams & Backpressure

- **Buffers:** Fixed-length raw binary memory blocks allocated **Off-Heap** directly in C++ RAM, bypassing V8 Garbage Collection overhead.
    
      
    
- **Streams:** Process data sequentially in 64 KB chunks, maintaining flat $O(1)$ memory usage.
    
      
    
- **Backpressure:** `.pipe()` automatically pauses readable streams when target write buffers exceed `highWaterMark`, preventing Out of Memory (OOM) crashes.
    
      
    

### 3. Cluster vs. Worker Threads

- **Cluster Module:** Multi-process scaling. Spawns isolated Node.js OS processes, each with its own V8 instance and Event Loop. Used to load-balance **I/O network requests** across CPU cores.
    
      
    
- **Worker Threads:** Multi-thread scaling within a single process. Shares memory via `SharedArrayBuffer`. Used to offload **CPU-bound computations** (crypto, image processing) off the main Event Loop.
    
      
    

### 4. Production Diagnostics & Memory Leaks

- **Uncaught Exceptions:** Log the failure, gracefully terminate open sockets, run `process.exit(1)`, and let process supervisors (PM2/Kubernetes) restart clean workers.
    
      
    
- **Memory Leak Tracing:** Detect via monotonically increasing `heapUsed` metrics. Capture heap dumps with `v8.getHeapSnapshot()` and inspect Retainer trees in Chrome DevTools to trace dangling global scope links, unremoved event listeners, or uncleared caches.
    
      
    

[[Javascript]]