# Variables, memory, and runtime behavior: JavaScript, Python, and Java

A practical comparison of how JavaScript (including Node.js), Python, and Java handle variables, objects, functions, memory, and execution. The details below describe common implementations—especially V8, CPython, and the JVM—rather than rules that every implementation must follow.

## 1. Variable declarations and typing

| Language | Typical declaration | What the declaration means |
|---|---|---|
| JavaScript | `let count = 1;`, `const name = "Ada";`, `var oldStyle = 1;` | `let` and `const` are block-scoped. `const` prevents rebinding the variable, not mutation of an object it refers to. `var` is function-scoped (or global-scoped) and is generally avoided in new code. |
| Python | `count = 1`, `name = "Ada"` | Assignment binds a name to an object. There is no separate declaration keyword in ordinary Python. |
| Java | `int count = 1;`, `String name = "Ada";` | A local variable has a declared type. The compiler checks that assigned values and operations are compatible with that type. |

### Static typing and dynamic typing

Java is statically typed: type rules are checked by the compiler, before the program runs, and also enforced by the runtime. Some errors can still happen at runtime, such as a null dereference or a failed cast.

JavaScript and Python are dynamically typed: a variable/name can be bound to values of different types at different times. The runtime checks whether each operation is valid. For example, adding two numbers works, while an unsupported combination can raise an error at runtime. Dynamic typing does not mean “no types”; values still have types.

“Compile time versus runtime” is a useful first distinction, but not the whole story. JavaScript engines and Python implementations may compile source to bytecode or machine code internally, and Java checks some conditions at runtime too.

## 2. Primitive values, objects, and references

- **JavaScript:** Primitive values include `number`, `string`, `boolean`, `bigint`, `symbol`, `undefined`, and `null`. Objects include arrays, functions, and ordinary objects. A variable can hold a primitive value or an object reference.
- **Python:** All values are objects, including integers, strings, lists, and functions. A name is bound to an object.
- **Java:** Primitive types (`int`, `double`, `boolean`, and others) hold primitive values. Class instances and arrays are objects, and variables of class or array type hold references. `String` is a reference type, not a primitive.

An **object reference** identifies an object; it is not the object itself. The exact representation of references is implementation-dependent.

### Assignment is not necessarily copying an object

Assignment usually copies a value or reference into a variable; it does not automatically clone the referenced object.

**JavaScript**

```javascript
const a = { score: 1 };
const b = a;           // Copies the object reference
b.score = 2;
console.log(a.score);  // 2: both names reach the same object

let x = 5;
let y = x;             // Copies the primitive value
y = 8;
console.log(x);        // 5
```

**Python**

```python
a = ["red"]
b = a                  # Binds b to the same list
b.append("blue")
print(a)               # ['red', 'blue']

x = 5
y = x                  # Both names initially refer to the integer object
y = 8                  # Rebinds y; it does not change the integer
print(x)               # 5
```

**Java**

```java
int x = 5;
int y = x;              // Copies the primitive value
y = 8;                  // x remains 5

int[] a = {1};
int[] b = a;            // Copies the array reference
b[0] = 9;
System.out.println(a[0]); // 9: both references identify the same array
```

To make an independent copy of a mutable object, use an appropriate copying operation. A shallow copy duplicates the outer container but may still share nested objects; a deep copy duplicates nested objects too. Neither assignment nor a `const` declaration automatically performs either kind of copy.

### Mutable and immutable values

- **JavaScript:** Strings and primitive values are immutable. Arrays and ordinary objects are mutable. `const` prevents rebinding, but does not freeze an object: `const items = []; items.push("x");` is valid. `Object.freeze` is shallow unless nested objects are separately frozen.
- **Python:** Strings, integers, floats, and tuples are immutable; lists, dictionaries, and sets are mutable. A tuple can still contain a mutable object, so immutability of the tuple does not make that nested object immutable.
- **Java:** Primitive values are assigned as values. `String` objects are immutable. Arrays and most collection objects are mutable, subject to their particular API and implementation.

Mutation changes an existing mutable object. Rebinding changes which object a variable/name refers to. They are different operations.

## 3. Stack, heap, and the limits of the simple model

The **call stack** tracks active function or method calls. A call has execution state such as where to return and the values needed to continue. Implementations commonly store some local values there, but the language does not promise that every local variable lives in a physical stack slot.

The **heap** is memory used for data whose lifetime and layout are not simply tied to one active call, commonly including objects. Runtimes manage heap allocation and reclamation in implementation-specific ways.

“Variables are on the stack; objects are on the heap” is an oversimplification:

- A local variable may be optimized away, kept in a register, or stored elsewhere.
- A local variable may hold a reference to an object, while the object itself is elsewhere.
- Primitive-like values can be stored inside objects or arrays.
- Compilers and JITs can eliminate allocations or move data while preserving observable behavior.
- Python and JavaScript language specifications do not require a particular stack/heap layout.

The useful mental model is about **ownership and reachability**, not a guaranteed physical address: calls have scoped execution state; objects may remain alive as long as the runtime considers them reachable.

## 4. Function calls, parameters, and return values

A function call normally evaluates its arguments, establishes a new logical call context, executes the body, and then returns a value (or an implicit “no useful value” result). The call stack tracks nested calls, for example `main → handleRequest → queryDatabase`. When a call returns, its active call context is no longer needed. An object created during the call can remain alive if a reference to it escaped—for example, by being returned or stored elsewhere.

Parameters are local names/variables initialized from the argument values. The details of what is copied differ in representation, but they do not change the key rule below: **Java, JavaScript, and Python all pass arguments by value.** For object arguments, the value being passed is a reference.

### Pass-by-value versus pass-by-reference

In pass-by-value, a function receives its own parameter initialized with a copy of the argument value. If that value is an object reference, the copied reference still identifies the same object. Therefore:

1. Mutating the shared object inside the function can be observed by the caller.
2. Reassigning the parameter to another object does not reassign the caller’s variable.

**Python**

```python
def update(items):
    items.append("inside")   # Mutates the shared list

def rebind(items):
    items = ["new list"]     # Rebinds only the local parameter

values = ["start"]
update(values)
print(values)                # ['start', 'inside']
rebind(values)
print(values)                # Still ['start', 'inside']
```

The same distinction applies in JavaScript and Java. Saying that these languages “pass objects by reference” is misleading: the caller’s variable itself is not passed by reference. Java, in particular, has no general pass-by-reference parameter mechanism.

### Functions as values

- **JavaScript / Node.js:** Functions are first-class values. They can be assigned to variables, passed as arguments, and returned from functions.
- **Python:** Functions are first-class objects and can likewise be passed, stored, and returned.
- **Java:** Methods are not standalone values in the same way. A method reference or lambda can be used where a functional interface is expected, such as `Runnable`, `Function<T, R>`, or a custom interface.

## 5. Closures and captured variables

A **closure** is a function together with access to variables from its surrounding lexical scope. It allows a function to use an enclosing binding after the enclosing call has returned.

**JavaScript**

```javascript
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const next = makeCounter();
console.log(next()); // 1
console.log(next()); // 2
```

**Python**

```python
def make_counter():
    count = 0

    def next_count():
        nonlocal count
        count += 1
        return count

    return next_count
```

In Python, `nonlocal` is needed to rebind an enclosing function’s local name; mutating a captured list or dictionary does not require it. Closures capture bindings, so a closure may observe a later value of a binding rather than a snapshot of its value.

**Java**

```java
int amount = 3; // Must be final or effectively final to capture
Runnable task = () -> System.out.println(amount);
```

Java lambdas can capture local variables only when they are final or effectively final. A captured local value does not become a mutable shared local variable; an object referenced by that value may itself still be mutable.

Captured state can outlive the original function call because the returned closure still needs that state. The runtime keeps the required environment reachable for as long as the closure can be used. This is useful, but retaining a closure can also retain a large object graph unintentionally.

## 6. Garbage collection and object lifetime

Garbage collection exists to reclaim memory occupied by objects the program can no longer use, reducing the need for programmers to manually free each object. Collection timing is generally nondeterministic: becoming eligible does not mean memory is released immediately.

An object is generally **eligible for collection** when it is no longer reachable from the runtime’s roots (such as active execution state, global/static state, or other live objects). Exact root sets and collection details vary by runtime.

- **JavaScript in V8 (including Node.js):** Primarily uses tracing garbage collection. The collector traces from roots and can reclaim unreachable objects.
- **CPython:** Primarily uses reference counting, with a cyclic garbage collector to find certain unreachable reference cycles. Other Python implementations may use different strategies. Dropping a reference can cause an object’s reference count to reach zero, but this is not a portable promise of immediate operating-system memory release.
- **Java on the JVM:** Uses tracing garbage collectors. The JVM chooses when to collect; application code should not depend on a particular collection time.

Deleting or removing a reference is not the same as immediately returning memory to the operating system:

- Other references may still keep the object reachable.
- The collector may wait until a later collection.
- A runtime may keep reclaimed memory available for reuse rather than return it to the OS.
- In Python, `del name` removes a binding; it does not directly destroy the object if other references remain. JavaScript has no general `delete` operation for local variables; `delete` applies to object properties. In Java, assigning `null` can remove one reference, but does not force collection.

### Memory leaks despite garbage collection

Garbage collection cannot reclaim objects that are still reachable, even if the program no longer finds them useful. Common causes include:

- Unbounded or incorrectly sized caches.
- Unintended global, static, or module-level references.
- Event listeners, callbacks, or timers that are not removed.
- Long-lived collections that keep old request or user data.
- Closures that retain large surrounding objects.

The fix is usually to correct ownership and lifetime—for example, remove listeners, bound caches, or clear references—not to force garbage collection.

## 7. Runtime, compilation, interpretation, and JIT

“Compiled versus interpreted” is not a clean either/or classification for modern language runtimes.

- **JavaScript:** V8 parses JavaScript, may generate bytecode, and can optimize frequently executed code to machine code using JIT techniques. Node.js embeds V8 and adds APIs and runtime facilities; its event loop and I/O are supported by Node.js and libraries such as libuv.
- **Python:** CPython compiles source to Python bytecode and executes it in the CPython virtual machine. Other Python implementations can execute code differently; “Python” is the language, while CPython is its most common implementation.
- **Java:** The Java compiler typically translates source code to JVM bytecode. The JVM verifies and executes bytecode, and commonly JIT-compiles frequently used code to machine code. The JVM can also interpret bytecode.

Compilation, interpretation, and JIT are stages/strategies a runtime can combine. The language name alone does not fully specify how a particular program will execute.

## 8. Event loops, threads, and workload

An **event loop** coordinates callbacks and asynchronous work. A **thread** is an execution path scheduled by the operating system or runtime. They are related but not interchangeable.

- **Node.js:** JavaScript callbacks usually run on the main event-loop thread. Asynchronous I/O lets the event loop handle other work while I/O is pending; some operations use worker threads or the libuv thread pool. CPU-heavy JavaScript on the event-loop thread can delay unrelated requests.
- **Python:** `asyncio` uses an event loop for cooperative asynchronous I/O; a task yields at `await`. Threads and processes are also available. In standard CPython builds, the Global Interpreter Lock (GIL) generally limits simultaneous execution of Python bytecode in threads, though threads can still help with I/O, and native code/processes can enable CPU parallelism. Check the Python implementation and version for specifics.
- **Java:** Threads and executors are common concurrency tools; modern Java also provides virtual threads. The JVM supports parallel execution, while synchronization and shared mutable state require care.

An **I/O-bound** workload spends much of its time waiting for disks, networks, or other services. Asynchronous I/O or threads can keep the system productive during waits. A **CPU-bound** workload spends its time doing computation; it benefits from parallel CPU execution only when the runtime and architecture actually permit it.

## 9. A request’s journey through an application

A simplified HTTP request flow is:

1. A server accepts the connection and reads the request.
2. Routing selects a handler function or method.
3. The handler creates local variables and references to objects; it validates and transforms input.
4. The application may call a database or external API, often waiting asynchronously or using a worker/thread.
5. The handler builds a result and returns it to the server framework.
6. The framework serializes the result, sends an HTTP response, and releases request-scoped references when they are no longer needed.

Real systems add connection pools, middleware, queues, retries, caches, and error handling. A request returning does not guarantee that every object created during it is immediately collected; another reference may still exist.

## 10. Performance: measure the workload

There is no useful universal answer to “which language is fastest?” Performance depends on the workload and the whole system:

- **CPU:** algorithm, data structures, runtime optimizations, and available parallelism.
- **I/O:** network/database latency, batching, connection reuse, and asynchronous design.
- **Memory:** allocation rate, object layout, retained data, and garbage-collection costs.
- **Architecture:** queues, caches, service boundaries, serialization, and contention.

Benchmark representative workloads with realistic data and measure latency, throughput, CPU, and memory before choosing an optimization.

## 11. How long do variables and objects live?

- **Variable/name lifetime:** A local binding is ordinarily usable only within its scope, though compiler/runtime implementation details may differ. A global or module-level binding can live for the lifetime of the relevant program/module.
- **Object reachability:** An object remains available while it is reachable through live references, including references from closures, globals, collections, or active calls.
- **Collection eligibility:** Once unreachable, an object may be collected, but collection and memory release are not guaranteed to happen immediately.

Do not equate the end of a variable’s source-level scope with immediate destruction of every object it referred to.

## 12. What happens in `result = a + b`?

The precise steps depend on the values and runtime, but a useful high-level sequence is: resolve the names, obtain the values, apply that language’s addition rules, then bind or store the result. Addition may allocate a new object, produce a primitive value, or fail with a type error.

### Python

```python
result = a + b
```

Python looks up the names `a` and `b`, applies the `+` operation supported by their values (commonly via the numeric or sequence addition protocol), and binds the name `result` to the returned value. The operation may create a new object—for example, adding two lists creates a new list—or raise `TypeError` if the values do not support that combination. Python does not declare `result`’s type in this statement.

### JavaScript

```javascript
const result = a + b;
```

`const` must be followed by a binding name and initializer; `const = a + b;` is invalid syntax. JavaScript evaluates `a` and `b`, then applies the `+` operator’s rules. Depending on the values, it may perform numeric addition or string concatenation (after the language’s conversions). It binds the result to `result`, which cannot later be rebound. If the result is a mutable object, its contents may still be changed.

### Java

```java
int result = a + b;
```

The compiler checks that `a` and `b` have types for which `+` is valid and that the resulting value can be stored in an `int`. For two `int` operands, Java performs integer addition and stores the primitive result. Integer overflow wraps according to Java’s integer arithmetic rules; it does not automatically throw an overflow exception. Java also uses `+` for string concatenation when a string operand is involved. Invalid type combinations are normally rejected at compile time.