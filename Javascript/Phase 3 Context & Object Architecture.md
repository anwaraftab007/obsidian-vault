## Topic 1: Demystifying `this`

Unlike variables (which follow Lexical Scope), the `this` keyword in regular functions is **dynamically bound at runtime** based strictly on how a function is invoked.

### The 4 Execution Binding Rules:

1. **Implicit Binding (Left of Dot):** Called as an object method (`obj.fn()`) $\rightarrow$ `this` points to `obj`.
    
2. **Explicit Binding (`call`, `apply`, `bind`):** Forced using `.call(obj)`, `.apply(obj, [args])`, or `.bind(obj)` $\rightarrow$ `this` points to the explicitly passed object.
    
3. **Default / Standalone Binding:** Called as a standalone function (`fn()`) $\rightarrow$ `this` points to `global` (`window` in browser) or `undefined` in strict mode.
    
4. **Lexical Binding (Arrow Functions):** Arrow functions do **not** have their own `this`. They capture `this` lexically from their outer parent enclosing context at creation time.
    

> **Key Takeaway for Node.js:**
> 
> Extracting a method from an object and passing it as a callback (e.g., passing `obj.method` into `setTimeout` or an event listener) detaches `this` from the object and resets it to default standalone binding (`global`/`undefined`). Use arrow functions or `.bind()` to preserve context.

> **Interview Q:** _How does `this` work in JavaScript, and what happens when a method reference is assigned to a variable?_
> 
> **Answer:** "In regular functions, `this` is dynamically determined at call time. If called as an object method, implicit binding points `this` to the object left of the dot. However, if that method reference is assigned to a variable and invoked independently, it loses its context and defaults to standalone binding, causing `this` to point to the global object or `undefined` in strict mode. Arrow functions avoid this because they do not bind `this`, inheriting it lexically from their outer scope."


### Knowledge Check: Topic 1

Read this code carefully before answering.

JavaScript

```
const logger = {
    prefix: "[LOG]:",
    logMessage: function(msg) {
        console.log(this.prefix + " " + msg);
    }
};

const execute = logger.logMessage;

execute("Server booted successfully");
```

1. **What will be printed to the console when `execute("Server booted successfully")` runs in non-strict Node.js?**
    
2. **Which of the 4 binding rules explains why this happens?**


You fell into the most famous trap in JavaScript! This exact scenario trips up almost everyone when learning `this`.

The actual output is **`undefined Server booted successfully`**.

Here is why:

On line 8, `const execute = logger.logMessage;`, there are **no parentheses `()`**. We are not executing `logger.logMessage`. We are taking the raw function reference and copying it into a standalone variable named `execute`.

Then, on line 10, we call `execute("Server booted successfully")`. Notice that when we invoke `execute()`, **there is no dot to the left of it**. It is a standalone function invocation.

Because it is called standalone, **Rule 3 (Default Binding)** applies, _not_ Rule 1. Inside `execute()`, `this` points to the `global` object. Because `global.prefix` does not exist, `this.prefix` evaluates to `undefined`.

### How to fix this in real-world code:

If you need to pass a method around while preserving `this`, you must explicitly bind it:

JavaScript

```
const execute = logger.logMessage.bind(logger);
execute("Server booted successfully"); // Output: [LOG]: Server booted successfully
```

## Phase 3, Topics 2 & 3: Prototypes & Prototypal Inheritance

In languages like Java or C++, inheritance works through **Classes**, which act as blueprints to create brand-new objects.

JavaScript does not have traditional class-based inheritance under the hood. Even though JS introduced the `class` keyword in ES6, it is just **syntactic sugar**. At the V8 engine level, JavaScript uses **Prototype Delegation**.

### Topic 2: The Prototype Chain

Every object in JavaScript maintains a hidden link to another object called its **Prototype** (accessed internally via `[[Prototype]]`, exposed as `__proto__` or via `Object.getPrototypeOf()`).

When you create an object:

JavaScript

```
const user = { username: "alex99" };
```

V8 doesn't just create `username: "alex99"`. It automatically links `user` to JavaScript's built-in `Object.prototype`. That is why you can call `user.toString()` or `user.hasOwnProperty("username")` even though you never defined those methods yourself.

#### The Lookup Chain Mechanism

When you access a property or method on an object (e.g., `obj.someMethod()`):

1. **Local Search:** V8 checks if `someMethod` exists directly on `obj`.
    
2. **Prototype Lookup:** If missing, V8 follows the `__proto__` link to `obj`'s prototype object and checks there.
    
3. **Chain Traversal:** If missing, it follows _that_ prototype's `__proto__` link to its parent prototype.
    
4. **End of Chain:** V8 keeps climbing until it reaches `Object.prototype.__proto__`, which is `null`. If still not found, it returns `undefined`.
    

```
[ appUser ] 
     | 
  __proto__ 
     v
[ User.prototype ] ------ (holds custom methods like login)
     | 
  __proto__ 
     v
[ Object.prototype ] ---- (holds built-in methods like toString)
     | 
  __proto__ 
     v
   null ----------------- (End of chain)
```

### Topic 3: Prototypal Inheritance & Memory Efficiency

Understanding prototypes isn't just about syntax—it has a direct impact on **RAM consumption** in long-running Node.js servers.

Imagine you are building a chat application in Node.js, and 50,000 users connect simultaneously.

#### The Anti-Pattern (Methods inside Constructor):

JavaScript

```
function User(name) {
    this.name = name;
    
    // BAD: Function created inside constructor
    this.sendMessage = function(msg) {
        console.log(`${this.name}: ${msg}`);
    };
}

const user1 = new User("Alice");
const user2 = new User("Bob");
```

**What V8 does in memory:** Every time `new User()` runs, V8 allocates memory for a **brand-new function object** in the Memory Heap for `sendMessage`.

If 50,000 users connect, V8 creates **50,000 duplicate function objects** in RAM. This wastes significant memory and forces the Garbage Collector to work much harder.

#### The Optimal Pattern (Prototype Delegation):

JavaScript

```
function User(name) {
    this.name = name;
}

// GOOD: Attach method ONCE to the Prototype
User.prototype.sendMessage = function(msg) {
    console.log(`${this.name}: ${msg}`);
};

const user1 = new User("Alice");
const user2 = new User("Bob");
```

**What V8 does in memory:** When `sendMessage` is attached to `User.prototype`, V8 creates **exactly one function object** in the Memory Heap.

Whether you have 2 users or 50,000 users, all instances delegate to that single function in RAM via their `__proto__` links. When `user1.sendMessage()` is called, V8 doesn't find it locally on `user1`, steps up the prototype chain to `User.prototype`, and executes the shared function.

### Interview Answer: Prototypes & Prototypal Inheritance

**Question:** _"How does prototypal inheritance work in JavaScript, and why is prototype delegation preferred over defining methods inside constructors for Node.js applications?"_

**Your Answer:**

> "JavaScript uses prototype delegation rather than traditional class-based inheritance. Every object maintains a hidden internal link to a prototype object. When accessing a property or method, V8 searches the object locally first; if missing, it climbs the prototype chain layer-by-layer until reaching `Object.prototype` or `null`.
> 
> Defining methods inside a constructor function causes V8 to allocate a new function instance in the Memory Heap for every single object created. In long-running Node.js applications handling thousands of concurrent requests, this leads to massive memory overhead. Attaching methods to `.prototype` ensures a single shared function instance lives in the heap, drastically reducing RAM usage through prototype delegation."


[[Javascript]]