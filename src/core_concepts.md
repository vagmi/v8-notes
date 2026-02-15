# Core V8 Concepts

Before building a runtime, you need to understand V8's fundamental building blocks. These concepts apply to any V8-based runtime, whether Node.js, Deno, or your own.

## Isolate

An **Isolate** is an isolated instance of the V8 engine with its own heap and garbage collector. Think of it as a completely independent JavaScript VM.

**Key characteristics:**
- Each isolate has its own memory heap - objects from one isolate cannot be used in another
- An isolate can only be accessed by one thread at a time (V8 is single-threaded per isolate)
- You can have multiple isolates in one process (each on different threads)

**When to use multiple isolates:**
- Running untrusted code in isolation (sandbox per tenant)
- Worker threads (each worker gets its own isolate)
- Microservices architecture where each service has independent JS execution

```rust
// Creating an Isolate with heap limits
let params = v8::CreateParams::default()
    .heap_limits(0, 128 * 1024 * 1024); // 128MB max heap
let mut isolate = v8::Isolate::new(params);
```

In toyjs, we create a single isolate for the entire runtime (see `runtime.rs:61`).

## Context

A **Context** is an execution environment within an isolate. It provides the global object and built-in JavaScript objects (Object, Array, etc.).

**Why contexts:**
- Multiple contexts can share one isolate
- Each context has isolated globals - variables defined in one context don't leak to another
- Useful for running untrusted code without polluting your main environment

**The global object:**
- In browsers: `window`
- In Node.js: `global`
- In your runtime: whatever you define

```rust
// Creating a Context
v8::scope!(let handle_scope, &mut isolate);
let context = v8::Context::new(handle_scope, Default::default());
```

Most simple runtimes use a single context. Advanced runtimes might use multiple contexts for sandboxing.

## Handles and Scopes

V8 uses garbage collection, so you can't hold raw pointers to JavaScript objects. Instead, V8 provides **handles** - smart pointers that the GC knows about.

### Types of Scopes

There are two main types of scopes in `rusty_v8`, with others derived from them:
- `HandleScope` - a scope to create and access `Local` handles.
- `TryCatch` - a scope to catch exceptions thrown from JavaScript.

### The Challenge of Scopes in Rust

V8 scopes have properties that are challenging to model in Rust:
- **Nesting**: `HandleScope`s can be nested, but handles are bound to the innermost scope. This means handle lifetimes are determined by the innermost `HandleScope`.
- **Immovability**: `HandleScope` and `TryCatch` cannot be moved because V8 holds direct pointers to them.
- **Inheritance**: The C++ API relies on inheritance, which is modeled using `Deref` in Rust.

### Creating and Initializing Scopes

Because scopes must be pinned to the stack, creating them involves allocation, pinning, and initialization.

The verbose way:
```rust
use v8::{HandleScope, Local, Object, Isolate, Context, ContextScope};

// 1. Allocate storage
let scope = HandleScope::new(&mut isolate);
// 2. Pin to stack
let scope = std::pin::pin!(scope);
// 3. Initialize
let mut scope = scope.init();
```

The idiomatic way using the `v8::scope!` macro:
```rust
// This expands into statements introducing `scope`
v8::scope!(let scope, &mut isolate);
```

### Scopes as Function Arguments

When passing a scope to a function, use `v8::PinScope`. This is a shorthand for `PinnedRef<'s, HandleScope<'i>>`.

```rust
fn create_number<'s>(scope: &mut v8::PinScope<'s, '_>) -> v8::Local<'s, v8::Number> {
    v8::Number::new(scope, 42.0)
}
```

### Inheritance via Deref

Scopes implement `Deref` and `DerefMut` to simulate inheritance. This allows you to pass a `ContextScope` to a function expecting a `PinScope`.

- `ContextScope` derefs to `HandleScope`
- `CallbackScope` derefs to `HandleScope`

### ContextScope

A `ContextScope` enters a context, making it the "current" context for V8 operations. Unlike `HandleScope`, it is not address-sensitive and can be moved.

```rust
let context_scope = v8::ContextScope::new(scope, context);
// Now we're "inside" the context
```

### Global Handles

If you need a handle that outlives a HandleScope (e.g., storing a callback for later), use `v8::Global`:

```rust
let global_context = v8::Global::new(scope, context);
// global_context can be stored in a struct and used later
```

## The Scope Stack Pattern

Every V8 operation follows this pattern:

```rust
pub fn execute_script(&mut self, code: &str) {
    // 1. Create HandleScope (borrows isolate mutably)
    // We create the scope, pin it to the stack, and initialize it
    let handle_scope = std::pin::pin!(v8::HandleScope::new(&mut self.isolate));
    let mut handle_scope = handle_scope.init();

    // 2. Get the context (convert Global → Local)
    let context = v8::Local::new(&mut handle_scope, &self.context);

    // 3. Enter the context
    // ContextScope wraps the HandleScope and doesn't need its own pinning
    let mut scope = v8::ContextScope::new(&mut handle_scope, context);

    // 4. Now you can work with V8
    let source = v8::String::new(&mut scope, code).unwrap();
    let script = v8::Script::compile(&mut scope, source, None).unwrap();
    script.run(&mut scope);
}
```

**Why the scope dance?**
- Rust's borrowing rules enforce V8's threading rules
- Scopes ensure handles are properly managed
- The type system prevents you from using handles after they're invalidated

This pattern appears everywhere in V8 code - you'll write it hundreds of times.
