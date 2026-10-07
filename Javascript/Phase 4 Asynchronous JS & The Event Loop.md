## Topic 1: Sync vs Async & Background APIs

JavaScript is single-threaded, operating on a single Call Stack. Non-blocking asynchronous I/O in Node.js is achieved by offloading tasks to C++ background APIs and the **libuv** library.

### The Asynchronous Execution Flow:

1. **Invocation:** Synchronous statements execute immediately on the Call Stack.
    
2. **Offloading:** Asynchronous APIs (`fs.readFile`, `setTimeout`, database drivers) hand off execution to background C++ threads (libuv) and immediately pop off the Call Stack.
    
3. **Background Execution:** C++ threads handle I/O concurrently without blocking the main V8 JavaScript thread.
    
4. **Queueing:** Upon completion, background workers push the associated callback function to the task queue.
    
5. **Stack Resolution:** Callbacks execute on the Call Stack only after all synchronous code finishes and the Call Stack is completely empty.
    

> **Key Takeaway for Node.js:** Heavy CPU-bound computation (e.g., cryptographic hashing in a `for` loop) stays on the main V8 stack and blocks the server. Heavy I/O computation (file reads, network calls) is offloaded to background threads and remains non-blocking.

> **Interview Q:** _How does non-blocking I/O work in single-threaded JavaScript?_ **Answer:** "V8 offloads asynchronous tasks like file system access or network requests to Node.js background C++ bindings and libuv. The asynchronous call immediately pops off the Call Stack, allowing synchronous execution to continue. Once background threads finish the I/O task, the callback is queued and eventually executed when the Call Stack empties."

### Knowledge Check: Topic 1

Analyze this code and determine the exact order of `console.log` outputs:

JavaScript

```
const fs = require('fs');

console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

console.log("C");
```

1. **What is the exact output order printed to the console?**
    
2. **Why does `"B"` print after `"C"`, even though the `setTimeout` timer is set to `0` milliseconds?**

Because setTimeout is a asynchronous operation so executiom context put this operation outside the call stack and V8 offloads them to Node's C++ APIs and libuv thread pool. and the synchronous operations continues.. once the asynchronous task is completed its callback is placed into a task queue and once call stack is empty, then the event loop pushes the callback onto the call stack to execute..  this is my friend how node keeps the main thread non blocking. 

## Topic 2: Event Loop Architecture (Microtasks vs Macrotasks)

The Event Loop categorizes asynchronous callbacks into different queues with strict priority rules.

### Queue Categorization:

1. **Microtask Queue (High Priority):**
    
    - **`process.nextTick()` Queue:** Executes before any other microtask in Node.js.
        
    - **Promise Queue:** Handles `.then()`, `.catch()`, `.finally()`, `async/await` resumptions, and `queueMicrotask()`.
        
2. **Macrotask Queue (Lower Priority):**
    
    - Timers (`setTimeout`, `setInterval`), I/O callbacks, `setImmediate()`.
        

### Draining Mechanics:

- **Step 1:** Run all synchronous code on the Call Stack.
    
- **Step 2:** Drain the **entire** Microtask Queue (first `process.nextTick`, then Promises) until empty.
    
- **Step 3:** Execute **one** Macrotask from the Macrotask Queue.
    
- **Step 4:** Repeat step 2 (drain Microtasks again if any were queued during the Macrotask execution).
    

> **Key Takeaway for Node.js:** Recursively calling `process.nextTick()` creates an infinite Microtask loop that starves the Macrotask queue, completely blocking I/O and timers from ever running.

> **Interview Q:** _How does the Event Loop handle Promise microtasks versus `setTimeout` macrotasks?_ **Answer:** "Promise callbacks enter the Microtask Queue, while `setTimeout` callbacks enter the Macrotask Queue. The Event Loop prioritizes Microtasks; once the Call Stack is empty, it drains the entire Microtask Queue completely before moving on to process Macrotasks. Therefore, resolved Promises will always execute before `setTimeout` callbacks, even if `setTimeout` is set to 0ms."

### Knowledge Check: Topic 2

Analyze this code and state the exact output order:

JavaScript

```
setTimeout(() => console.log("A"), 0);

Promise.resolve().then(() => {
    console.log("B");
    process.nextTick(() => console.log("C"));
});

process.nextTick(() => console.log("D"));

console.log("E");
```

1. **What is the exact output order printed to the console?**
    
2. **Explain why `"C"` prints before `"A"`.**

Settimeout goes to Macrotask queue
then Promise.resolve().then() goes to MicroTask queue
then process.nextTick() goes to MicroTask queue with the highest priority..
then console will go to call stack which gets executed first..

### 1. Simple FIFO Order

If you enqueue multiple tasks of the same type at the top level, they execute in the exact order they were registered.
When two or more microtasks have the exact same priority, the Event Loop executes them in strict **First-In, First-Out (FIFO)** order based on when they were queued, and it will completely **drain the entire microtask queue** before moving on to the macrotask queue.

### 2. Nested Microtasks (Enqueued During Execution)

If a microtask enqueues _another_ same-priority microtask while running, the new microtask is pushed to the back of the microtask queue. It will run after all currently waiting microtasks finish, but **still before** any macrotask (`setTimeout`, `setImmediate`, etc.) can run.

### 3. Queue Starvation Risk

Because the Event Loop insists on completely emptying (draining) the microtask queue before rendering UI or moving to macrotasks, recursively spawning microtasks will block the main thread forever.

JavaScript

```
function infiniteMicrotasks() {
  Promise.resolve().then(() => {
    console.log('Looping microtask...');
    infiniteMicrotasks(); // Enqueues another microtask infinitely
  });
}

setTimeout(() => console.log('Will never run!'), 0);
infiniteMicrotasks();
```

In this scenario, `setTimeout` will **never** execute because the microtask queue is continuously refilled and never reaches an empty state.


# Phase 4: Asynchronous JS & The Event Loop

## Topic 3: libuv Event Loop Phases & `setImmediate`

Node's libuv Event Loop executes macrotasks across 6 distinct phases in a circular order:

1. **Timers Phase:** Executes callbacks scheduled by `setTimeout()` and `setInterval()`.
    
2. **Pending Callbacks Phase:** Executes deferred system I/O callbacks.
    
3. **Idle/Prepare Phase:** Internal engine operations.
    
4. **Poll Phase:** Retrieves new I/O events (file system, network) and executes their handlers.
    
5. **Check Phase:** Executes callbacks registered with `setImmediate()`.
    
6. **Close Callbacks Phase:** Executes close event handlers (e.g., `socket.on('close')`).
    

### Phase Transitioning & Ordering

- **Microtask Draining:** Microtasks (`process.nextTick` and Promises) are drained **between every single phase** transition.
    
- **Main Script Order:** `setTimeout(fn, 0)` vs `setImmediate(fn)` at the root level is non-deterministic due to system-level timer instantiation delay.
    
- **I/O Callback Order:** Inside an I/O handler (Poll Phase), `setImmediate` is **guaranteed** to execute before `setTimeout(fn, 0)` because the Check Phase immediately follows the Poll Phase.
    

> **Key Takeaway for Node.js:** Use `setImmediate()` when you want to execute code right after the current I/O polling phase completes without risking microtask starvation.

> **Interview Q:** _Why does `setImmediate` take precedence inside I/O callbacks?_ **Answer:** "In an I/O callback, execution occurs within the Poll phase of libuv. Moving to the next phase immediately transitions to the Check phase where `setImmediate` handlers live. `setTimeout` handlers reside in the Timers phase, requiring the loop to complete a full iteration before executing."

### Interview Answer: Event Loop Phases

**Question:** _"What is the difference between `setImmediate` and `setTimeout(fn, 0)`, and why does `setImmediate` execute first inside an I/O callback?"_

**Your Answer:**

> "The libuv Event Loop is divided into specific phases. `setTimeout` callbacks run in the Timers phase, while `setImmediate` callbacks run in the Check phase.
> 
> When called in the main module, execution order is non-deterministic due to timer preparation overhead. However, inside an I/O callback (like `fs.readFile`), the Event Loop is actively in the Poll phase. Moving to the next phase lands directly in the Check phase, forcing `setImmediate` to execute before the Event Loop can cycle back around to the Timers phase for `setTimeout`."


### Knowledge Check: Topic 3

Analyze this code and determine the exact output order:

JavaScript

```
const fs = require('fs');

fs.readFile(__filename, () => {
    setTimeout(() => console.log("1. Timer"), 0);
    setImmediate(() => console.log("2. Immediate"));
    
    process.nextTick(() => {
        console.log("3. NextTick");
    });
});
```

1. **What is the exact output order printed to the console?**
    
2. **Explain the step-by-step phase transitions that produce this output.**

**`process.nextTick` sneaks in before both of them!**

Here is why:

Microtasks do not wait for phase transitions to complete. The exact moment the synchronous code inside the `fs.readFile` callback finishes, the Call Stack becomes empty. **Before libuv is allowed to transition from the Poll Phase to the Check Phase, Node.js pauses to drain the Microtask Queue.**

## Topic 4: Async/Await Mechanics

`async/await` is syntactic sugar over native Promises, providing synchronous-looking syntax while executing non-blocking asynchronous code.

### V8 Engine Mechanics:

- **`async` Functions:** Automatically wrap return values in a `Promise.resolve()`. Uncaught errors inside an `async` function result in a rejected Promise.
    
- **`await` Suspension:** When reaching `await`, V8 pauses execution of the `async` function and pops its context off the Call Stack.
    
- **Microtask Continuation:** The code following `await` is registered as a microtask callback tied to the completion of the awaited Promise.
    

> **Key Takeaway for Node.js:**
> 
> Using `await` inside loops sequentially pauses execution (e.g., awaiting 10 DB queries in a `for` loop takes $10 \times N$ ms). Use `Promise.all()` to trigger asynchronous tasks concurrently in parallel when tasks are independent.

> **Interview Q:** _What happens to the call stack when an `async` function hits an `await` statement?_
> 
> **Answer:** "When an `async` function encounters an `await`, V8 pauses the function's execution and pops its context off the Call Stack, allowing the main thread to process other tasks. The code following `await` is treated as a promise callback and queued in the Microtask Queue once the awaited value resolves."

### Interview Answer: Async/Await Under the Hood

**Question:** _"How does `async/await` work internally in V8, and how does it relate to the Event Loop?"_

**Your Answer:**

> "`async/await` is built directly on top of native Promises and generators. Marking a function as `async` guarantees it returns a Promise. When the engine encounters an `await` keyword, it evaluates the target expression and yields execution, popping the `async` function off the Call Stack so the main thread remains non-blocking.
> 
> The remaining lines of code inside the `async` function are converted into a microtask continuation. Once the awaited Promise resolves, that continuation is placed into the Promise Microtask Queue and executed as soon as the Call Stack is empty."

### Final Phase 4 Knowledge Check

This is a comprehensive code challenge that combines everything in Phase 4: synchronous execution, `nextTick`, Promise microtasks, `setTimeout`, and `async/await`.

Trace this code and determine the exact output order:

JavaScript

```
async function main() {
    console.log("1");
    
    setTimeout(() => console.log("2"), 0);
    
    await Promise.resolve();
    console.log("3");
    
    process.nextTick(() => console.log("4"));
}

console.log("5");
main();
console.log("6");
```

1. **What is the exact output order printed to the console?**
    
2. **Break down the sequence step-by-step.**


**1. Synchronous Pass (`5`, `1`, `6`)**

- `console.log("5")` runs first.
    
- `main()` is called $\rightarrow$ logs `"1"`, offloads `setTimeout` to the **Macrotask Queue**, and hits `await`.
    
- `await` pauses `main()` and queues the rest of the function into the **Microtask Queue**.
    
- `console.log("6")` runs. The Call Stack is now empty.


**2. Microtask Queue Draining (`3`, `4`)**

- `main()` resumes from the Promise Microtask queue $\rightarrow$ logs `"3"`.
    
- `process.nextTick` is called inside `main()`, queueing a new microtask which drains immediately before moving to macrotasks $\rightarrow$ logs `"4"`.


**3. Macrotask Queue Execution (`2`)

- Microtasks are empty. The Event Loop picks `setTimeout` from the Macrotask queue $\rightarrow$ logs `"2"`.


[[Javascript]]