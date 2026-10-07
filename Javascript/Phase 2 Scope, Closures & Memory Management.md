## Topic 1: Lexical Scope & The Scope Chain

Every Execution Context contains a **Lexical Environment** composed of two parts:

  

1. **Environment Record:** The local memory storing variables and functions defined inside the current context.
    
      
    
2. **Outer Environment Reference:** A hidden link pointing to the Lexical Environment of the outer parent scope.
    
      
    

JavaScript uses **Lexical Scoping**, meaning scope resolution is determined strictly by where functions are **physically written in the source code**, completely independent of where or how they are invoked at runtime.

  

### The Scope Chain Lookup Mechanism

When accessing a variable, V8 performs a multi-step search:

  

1. Searches the local Environment Record.
    
      
    
2. If missing, traverses up the Outer Reference link to the parent's Environment Record.
    
      
    
3. Continues climbing layer-by-layer up the **Scope Chain** until it finds the variable or reaches the Global scope.
    
      
    
4. Throws a `ReferenceError` if the variable is not found in the Global scope.
    
      
    

> **Key Takeaway for Node.js:**
> 
> Scope is static and defined at compile/parse time. A function invoked deep inside an asynchronous handler or external module will still resolve its free variables based on where it was declared in your codebase.
> 
>   

> **Interview Q:** _What is Lexical Scope, and how does the Scope Chain work?_
> 
> **Answer:** "Lexical Scope means that variable accessibility is determined by the physical placement of functions within the source code. During creation, an Execution Context retains an outer lexical reference to its parent scope. If a variable isn't found in the local scope, the engine follows these outer references up the Scope Chain to the global environment. The call stack location has zero impact on variable lookup."
> 
>   

## Topic 2: Closures

A **Closure** is created when an inner function is defined inside an outer function, retaining access to the outer function’s lexical scope even after the outer function's Execution Context has finished and popped off the Call Stack.

  

### V8 Internal Mechanics

- **Variable Scope Analysis:** During parsing, V8 analyzes inner functions to determine which outer variables are actively referenced.
    
      
    
- **Heap Allocation:** When the outer function returns, V8 extracts _only_ the referenced variables from the dying stack context and allocates them in a dedicated Closure object in the **Memory Heap**.
    
      
    
- **Hidden Link:** V8 binds this Closure object to the inner function's hidden internal property (`[[Environment]]`).
    
      
    

> **Key Takeaway for Node.js:**
> 
> Closures power Node's asynchronous callback architecture. They allow callbacks, promises, and event handlers to access original request variables long after the initial router function has completed execution.
> 
>   

> **Interview Q:** _What is a closure and how does V8 manage it under the hood?_
> 
> **Answer:** "A closure is a function that retains access to its parent lexical scope after the outer function has returned. Under the hood, V8 performs scope analysis during parsing. Instead of destroying the entire outer context when it pops off the Call Stack, V8 allocates only the referenced outer variables to a Closure object in the Memory Heap and attaches it to the inner function's hidden `[[Environment]]` property."
> 
>   

## Topic 3: The Memory Heap & Garbage Collection

### Memory Allocation

- **The Call Stack:** Manages execution flow and stores primitive values (numbers, strings, booleans, `undefined`, `null`).
    
      
    
- **The Memory Heap:** A large, unstructured region in RAM storing reference types (Objects, Arrays, Functions) and Closures.
    
      
    

### The Mark & Sweep Algorithm

To prevent running out of RAM, V8's Garbage Collector automatically frees unneeded heap memory using **Mark & Sweep**:

  

1. **Root Phase:** The Garbage Collector starts at the Global Root (and active stack frames).
    
      
    
2. **Mark Phase:** It recursively traces all object references, marking every accessible object as **Reachable** (alive).
    
      
    
3. **Sweep Phase:** It scans the Memory Heap and deallocates any memory not marked as reachable.
    
      
    

### Memory Leaks in Node.js

A **Memory Leak** occurs when references to unused data remain attached to a reachable root (e.g., pushing data into global arrays, uncleared event listeners, or unintended closures holding large scope references). Because the data remains "reachable," V8 cannot sweep it, eventually crashing the process with an `Out of Memory` (OOM) error.

  

> **Key Takeaway for Node.js:**
> 
> Always unregister event listeners, clear global references, and isolate closures to avoid keeping unused memory reachable in long-running Node.js processes.
> 
>   

> **Interview Q:** _How does V8's Garbage Collection work, and what causes a memory leak in Node.js?_
> 
> **Answer:** "V8 manages memory using the Mark & Sweep algorithm. Starting from global roots and active call stack frames, it traces all accessible memory references and marks them as reachable. Unreachable memory is swept during garbage collection. A memory leak occurs when objects that are no longer needed remain referenced by reachable objects—such as global variables or active closures—preventing V8 from sweeping them."


[[Javascript]]