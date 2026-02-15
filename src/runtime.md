# Runtime Architecture

A JavaScript runtime is more than just the V8 engine - it's the orchestration layer that connects V8 to the outside world. Let's understand the key architectural decisions.

## The Runtime Struct

At the core, a runtime needs to hold:
1. **The V8 Isolate** - The JavaScript engine instance
2. **A Global Context** - The execution environment
3. **Communication Channels** - For async operations (timers, I/O, etc.)

```rust
pub struct JsRuntime {
    isolate: v8::OwnedIsolate,
    context: v8::Global<v8::Context>,
    // Channels for event loop communication
    scheduler_tx: mpsc::UnboundedSender<SchedulerMessage>,
    callback_rx: mpsc::UnboundedReceiver<CallbackMessage>,
}
```

**Why `v8::Global` for context?**
- The context needs to outlive individual function scopes
- `v8::Global` is a handle that can be stored in structs and moved between functions
- It's converted to `v8::Local` when you need to use it (with a HandleScope)

## Initialization Sequence

Setting up a runtime involves several steps in a specific order:

### 1. Initialize the V8 Platform (Once Per Process)

V8 requires platform initialization before creating isolates. Use `std::sync::Once` to ensure it happens exactly once:

```rust
use std::sync::Once;

static INIT: Once = Once::new();

pub fn init_v8() {
    INIT.call_once(|| {
        let platform = v8::new_default_platform(0, false).make_shared();
        v8::V8::initialize_platform(platform);
        v8::V8::initialize();
    });
}
```

**Why once?** V8 maintains global state. Multiple initializations will crash your program.

### 2. Create the Isolate

```rust
init_v8(); // Ensure V8 is initialized first

let params = v8::CreateParams::default();
let mut isolate = v8::Isolate::new(params);
```

**Optional: Set heap limits**
```rust
let params = v8::CreateParams::default()
    .heap_limits(0, 128 * 1024 * 1024); // 128MB
```

### 3. Create the Context

The context must be created inside a HandleScope and immediately converted to a Global:

```rust
let context = {
    let scope = &mut v8::HandleScope::new(&mut isolate);
    let context = v8::Context::new(scope, Default::default());
    v8::Global::new(scope, context) // Convert to Global before scope ends
};
```

**Why the extra scope block?** We need the HandleScope to create the context, but we can't store the HandleScope in our struct. We convert to a Global handle which can outlive the scope.

### 4. Setup Native Bindings

With the context created, you can now attach native functions to the global object:

```rust
let scope = &mut v8::HandleScope::new(&mut isolate);
let context = v8::Local::new(scope, &context);
let scope = &mut v8::ContextScope::new(scope, context);

// Get the global object
let global = context.global(scope);

// Create a native function
let print_fn = v8::FunctionTemplate::new(scope, print_callback);
let print_fn = print_fn.get_function(scope).unwrap();

// Attach to global
let name = v8::String::new(scope, "print").unwrap();
global.set(scope, name.into(), print_fn.into());
```

Now JavaScript can call `print("hello")`!

## Pinned Scopes

Modern `rusty_v8` uses a **pinned scope pattern** for better safety:

```rust
let handle_scope = std::pin::pin!(v8::HandleScope::new(&mut isolate));
let mut scope = handle_scope.init();
// use scope...
```

This prevents accidentally moving scopes, which can cause memory safety issues. You'll see this pattern in toyjs (see `runtime.rs:65`).

## Startup Optimization: Snapshots

V8 supports **heap snapshots** - serialized state of an isolate that can be quickly loaded instead of initializing from scratch.

**How it works:**
1. At build time: Create an isolate, run initialization code, serialize the heap
2. At runtime: Load the blob directly

```rust
let snapshot_data: &'static [u8] = include_bytes!("snapshot.bin");
let params = v8::CreateParams::default()
    .snapshot_blob(snapshot_data);
let isolate = v8::Isolate::new(params);
```

**Benefits:**
- Faster startup (10-100x for large standard libraries)
- Smaller runtime binary (move JS code into snapshot)
- Used by Node.js, Deno for built-in modules

**Tradeoffs:**
- Build-time complexity
- Snapshot must be regenerated when code changes
- Only helps if you have significant initialization code

For a simple runtime like toyjs, snapshots aren't necessary. But production runtimes (Node, Deno) use them extensively.
