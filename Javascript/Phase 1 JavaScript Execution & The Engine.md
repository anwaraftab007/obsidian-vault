## Topic 1: V8 Engine Internals (JIT Compilation)

JavaScript is neither purely interpreted nor purely compiled. The V8 Engine uses **Just-In-Time (JIT) Compilation** to get the fast startup of an interpreter and the high performance of a compiler.

  

**The 3-Step Process:**

  

1. **Parsing:** Code is parsed and converted into an **Abstract Syntax Tree (AST)**.
    
      
    
2. **Ignition (The Interpreter):** Converts the AST into **Bytecode** and starts running it immediately.
    
      
    
3. **TurboFan (The Compiler):** A Profiler watches for "hot" code (frequently used functions). TurboFan takes this hot Bytecode and compiles it into highly optimized **Machine Code**.
    
      
    

> **Key Takeaway for Node.js:**
> 
> If optimized Machine Code suddenly receives a different data type (e.g., a string instead of an expected integer), V8 panics and **De-optimizes**, throwing away the fast code and falling back to slower Bytecode. Writing predictable code with consistent data types keeps Node.js running at maximum speed.
> 
>   

> **Interview Q:** _How does V8 execute JavaScript?_
> 
> **Answer:** "V8 uses JIT compilation. It parses code into an AST, which the Ignition interpreter turns into bytecode for immediate execution. A profiler monitors for 'hot' paths, sending them to the TurboFan compiler to become optimized machine code. If data types change unexpectedly, V8 performs de-optimization, reverting to the slower bytecode."
> 
>   

## Topic 2: Execution Context, Memory & Hoisting

Whenever JS runs, it happens inside an **Execution Context**. The engine processes this context in two distinct passes.

  

**Pass 1: Creation Phase (Memory Allocation)**

Before running line 1, V8 scans the file and allocates memory. This is the true mechanism of **Hoisting**.

  

- **Functions:** Stored entirely in memory.
    
      
    
- **`var` variables:** Allocated and initialized with `undefined`.
    
      
    
- **`let` & `const` variables:** Allocated but remain uninitialized in the **Temporal Dead Zone (TDZ)**.
    
      
    

**Pass 2: Execution Phase (Thread of Execution)**

V8 runs the code line-by-line, assigning actual values to the variables in memory and executing function calls.

  

> **Key Takeaway:**
> 
> Hoisting does _not_ physically move code to the top of the file. It is simply V8 allocating memory during Pass 1.
> 
>   

> **Interview Q:** _What is hoisting, and how do `var` and `let` differ?_
> 
> **Answer:** "Hoisting occurs during the Execution Context's Creation Phase, where memory is allocated before execution. `var` is hoisted and initialized with `undefined`, so accessing it early returns `undefined`. `let` and `const` are hoisted but remain uninitialized in the Temporal Dead Zone; accessing them early throws a ReferenceError."
> 
>   

## Topic 3: The Call Stack

JavaScript is **single-threaded**, meaning it has exactly one Call Stack. The Call Stack is a **Last-In, First-Out (LIFO)** data structure used to track the thread of execution.

  

**How it works:**

  

1. The **Global Execution Context** is pushed to the bottom of the stack when the script starts.
    
      
    
2. Every time a function is called, a new **Function Execution Context** is created and pushed to the top.
    
      
    
3. The engine _only_ executes the context at the absolute top of the stack.
    
      
    
4. When a function finishes (returns), it is **popped** off the stack, and execution resumes where it left off below.
    
      
    

> **Key Takeaway for Node.js:**
> 
> A single thread means V8 can only do one thing at a time. If a function takes 5 seconds to run, no other code can execute for 5 seconds. This is called **Blocking the Event Loop**. Calling functions endlessly without returning causes a **Stack Overflow**.
> 
>   

> **Interview Q:** _What does single-threaded mean regarding the Call Stack?_
> 
> **Answer:** "It means the JS engine has one Call Stack and executes synchronously, one step at a time via LIFO. When a function is called, its context is pushed to the top, and the engine focuses solely on it until it pops off. If a heavy task blocks the stack, a Node.js server cannot process any other incoming requests."
> 
>   

Whenever you have this pasted and are ready, reply with your answer to the **Lexical Scope Knowledge Check** we left off on, and we will dive straight into Phase 2, Topic 2: Closures!

  

_(Reminder of the Knowledge Check code:)_

JavaScript

```
const secretKey = "Global123";

function runServer() {
    const secretKey = "Local456";
    authenticate();
}

function authenticate() {
    console.log(secretKey);
}

runServer();
```
[[Javascript]]