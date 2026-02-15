# The Event Loop

JavaScript's power comes from its non-blocking async model. But V8 itself is synchronous - it can't wait for network requests or timers. We must build an event loop in Rust to provide async capabilities.

## Why V8 Needs an Event Loop

V8 executes JavaScript synchronously:
```javascript
console.log("A");
// V8 blocks here if this were synchronous
doExpensiveWork();
console.log("B");
```

But JavaScript expects async operations:
```javascript
console.log("A");
setTimeout(() => console.log("C"), 1000); // Don't block!
console.log("B");
// Output: A, B, C (after 1 second)
```

The runtime (not V8) must provide:
- **Timers** (setTimeout, setInterval)
- **I/O** (fetch, file operations)
- **Promises** (microtask queue)

## The Two-Channel Architecture

The standard pattern uses message passing between V8 (single-threaded) and an async event loop:

```
┌─────────────┐              ┌──────────────────┐              ┌──────────────┐
│ JavaScript  │              │  Rust Runtime    │              │  Event Loop  │
│   (V8)      │              │  (Main Thread)   │              │   (Tokio)    │
└─────────────┘              └──────────────────┘              └──────────────┘
      │                               │                                │
      │  fetch("url")                 │                                │
      │------------------------------>│                                │
      │                               │  SchedulerMessage::Fetch       │
      │                               │------------------------------->│
      │                               │                                │
      │                               │                         (spawn async task)
      │                               │                         (perform HTTP)
      │                               │                                │
      │                               │    CallbackMessage::Success    │
      │                               │<-------------------------------│
      │  resolve(response)            │                                │
      │<------------------------------│                                │
      │                               │                                │
```

**Two channels:**
1. **Scheduler Channel** (JS → Event Loop): Sends async work requests
2. **Callback Channel** (Event Loop → JS): Sends completion notifications

## Message Types

Define enums for communication:

```rust
pub type CallbackId = u64;

// JavaScript → Event Loop
pub enum SchedulerMessage {
    ScheduleTimeout(CallbackId, u64),  // id, delay_ms
    Fetch(CallbackId, String),          // id, url
    Shutdown,
}

// Event Loop → JavaScript
pub enum CallbackMessage {
    ExecuteTimeout(CallbackId),
    FetchSuccess(CallbackId, String),   // id, body
    FetchError(CallbackId, String),     // id, error
}
```

## The Event Loop (Rust Async Side)

The event loop runs in a Tokio task and spawns async operations:

```rust
pub async fn run_event_loop(
    mut scheduler_rx: mpsc::UnboundedReceiver<SchedulerMessage>,
    callback_tx: mpsc::UnboundedSender<CallbackMessage>,
) {
    while let Some(msg) = scheduler_rx.recv().await {
        match msg {
            SchedulerMessage::ScheduleTimeout(id, delay_ms) => {
                let tx = callback_tx.clone();
                tokio::spawn(async move {
                    tokio::time::sleep(Duration::from_millis(delay_ms)).await;
                    let _ = tx.send(CallbackMessage::ExecuteTimeout(id));
                });
            }
            SchedulerMessage::Fetch(id, url) => {
                let tx = callback_tx.clone();
                tokio::spawn(async move {
                    match reqwest::get(&url).await {
                        Ok(resp) => match resp.text().await {
                            Ok(body) => {
                                let _ = tx.send(CallbackMessage::FetchSuccess(id, body));
                            }
                            Err(e) => {
                                let _ = tx.send(CallbackMessage::FetchError(id, e.to_string()));
                            }
                        },
                        Err(e) => {
                            let _ = tx.send(CallbackMessage::FetchError(id, e.to_string()));
                        }
                    }
                });
            }
            SchedulerMessage::Shutdown => break,
        }
    }
}
```

See `runtime/event_loop.rs` in toyjs for the full implementation.

## The V8 Pump (Sync Side)

The main thread periodically checks for completed callbacks and executes them in V8:

```rust
pub fn process_callbacks(&mut self) {
    let scope = std::pin::pin!(v8::HandleScope::new(&mut self.isolate));
    let mut scope = scope.init();
    let context = v8::Local::new(&scope, &self.context);
    let scope = &mut v8::ContextScope::new(&mut scope, context);

    // Process all pending callbacks
    while let Ok(msg) = self.callback_rx.try_recv() {
        match msg {
            CallbackMessage::ExecuteTimeout(id) => {
                // Call JS: __executeTimer(id)
                let global = context.global(scope);
                let key = v8::String::new(scope, "__executeTimer").unwrap();
                if let Some(func) = global.get(scope, key.into()) {
                    if func.is_function() {
                        let func: v8::Local<v8::Function> = func.try_into().unwrap();
                        let id_val = v8::Number::new(scope, id as f64);
                        func.call(scope, global.into(), &[id_val.into()]);
                    }
                }
            }
            // ... handle other callbacks
        }
    }

    // CRITICAL: Process microtasks (Promises!)
    let tc_scope = std::pin::pin!(v8::TryCatch::new(scope));
    let mut tc_scope = tc_scope.init();
    tc_scope.perform_microtask_checkpoint();
}
```

## Microtasks and Promises

V8 has an internal **microtask queue** for Promises. After processing callbacks, you **must** call `perform_microtask_checkpoint()`:

```rust
tc_scope.perform_microtask_checkpoint();
```

**Why?** When JavaScript uses Promises:
```javascript
fetch("url").then(data => console.log(data));
```

The `.then()` handler is queued as a microtask. Without the checkpoint, Promise handlers never execute!

## The Main Loop

In your main function, continuously pump callbacks:

```rust
#[tokio::main]
async fn main() {
    let mut runtime = JsRuntime::new();
    let event_loop = runtime.run_event_loop();

    runtime.execute_script_module("fetch('https://httpbin.org/get')");

    // Keep processing callbacks
    for _ in 0..100 {
        runtime.process_callbacks();
        tokio::time::sleep(Duration::from_millis(100)).await;
    }

    runtime.shutdown();
    event_loop.await.unwrap();
}
```

## Points to note

1. **V8 is synchronous** - The event loop is your responsibility
2. **Message passing** - Tokio and V8 communicate via channels
3. **Callback IDs** - Track which JS callback to invoke when async work completes
4. **Microtask checkpoint** - Essential for Promise resolution
5. **try_recv()** - Non-blocking, V8 doesn't wait for callbacks

This architecture is used by Node.js (libuv), Deno (Tokio), and most V8-based runtimes.

## Limitations of the Message-Passing Design

While our two-channel architecture is clear and educational, it has several limitations in production:

### 1. **Performance Overhead**

Every async operation requires:
- Creating and sending a message through a channel
- Channel synchronization costs
- Polling the callback channel (`try_recv()`)
- Looking up callbacks in JavaScript Maps

```javascript
// JavaScript side maintains callback maps
const timeoutCallbacks = new Map();
const fetchResolvers = new Map();

// Every async op needs manual bookkeeping
function fetch(url) {
    const id = nextId++;
    return new Promise((resolve, reject) => {
        fetchResolvers.set(id, { resolve, reject });
        __scheduleFetch(id, url);
    });
}
```

### 2. **Polling vs Event-Driven**

Our main loop polls for callbacks every 100ms:
```rust
for _ in 0..100 {
    runtime.process_callbacks();
    tokio::time::sleep(Duration::from_millis(100)).await;
}
```

This means:
- 100ms latency for fast operations
- Wasted CPU cycles when idle
- No backpressure control

### 3. **Manual Type Conversions**

Each binding requires manual argument handling:
```rust
fn fetch_binding(
    scope: &mut v8::HandleScope,
    args: v8::FunctionCallbackArguments,
    mut retval: v8::ReturnValue,
) {
    // Manual extraction and validation
    let id = args.get(0).number_value(scope).unwrap() as u64;
    let url = args.get(1).to_rust_string_lossy(scope);
    // ... send message
}
```

### 4. **Limited Error Handling**

Errors become simple strings:
```rust
CallbackMessage::FetchError(id, e.to_string())
```

No stack traces, error types, or proper error propagation.

### 5. **No Resource Management**

No way to:
- Track open connections
- Cancel in-flight operations
- Limit concurrent operations
- Clean up resources on shutdown

## How Deno Solves These Problems

Deno uses a sophisticated **op (operation) system** that addresses all these limitations:

### 1. **Direct Op Integration**

Instead of message passing, Deno ops are directly callable:

```rust
#[op2(async)]
pub async fn op_fetch(
    state: Rc<RefCell<OpState>>,
    #[string] url: String,
) -> Result<Response, AnyError> {
    // Direct async Rust code
    let client = state.borrow().borrow::<HttpClient>();
    let resp = client.get(url).await?;
    Ok(Response::from(resp))
}
```

The `#[op2]` macro generates:
- V8 function bindings
- Type conversions (automatic)
- Fast call paths (when possible)
- Error handling

### 2. **Promise-Based Architecture**

Deno tracks promises internally with a clever system:

```rust
// In Rust
pub struct JsRuntime {
    pending_ops: FuturesUnorderedDriver<OpResult>,
    promise_ring: Vec<Option<(PromiseId, OpResult)>>,
    // ...
}

// In JavaScript (core/01_core.js)
const promiseRing = new Array(RING_SIZE);
const promiseMap = new SafeMap();

// Promises get tagged with IDs internally
function opAsync(name, ...args) {
    const promise = new Promise((resolve, reject) => {
        // ...
    });
    promise[promiseIdSymbol] = promiseId;
    return promise;
}
```

### 3. **Integrated Event Loop**

Instead of polling, Deno integrates async ops into the main event loop:

```rust
pub fn poll_event_loop(&mut self, cx: &mut Context) -> Poll<Result<(), Error>> {
    // Poll pending ops
    loop {
        match self.pending_ops.poll_next_unpin(cx) {
            Poll::Ready(Some((promise_id, result))) => {
                // Collect results
                results.push((promise_id, result));
            }
            Poll::Ready(None) | Poll::Pending => break,
        }
    }
    
    // Single JavaScript call with all results
    if !results.is_empty() {
        self.resolve_promises_in_js(results);
    }
    
    // Run microtasks
    self.perform_microtask_checkpoint();
    
    // Determine if more work pending
    if self.has_pending_work() {
        Poll::Pending
    } else {
        Poll::Ready(Ok(()))
    }
}
```

### 4. **OpState for Resource Management**

Deno provides `OpState` - a type-safe container for runtime state:

```rust
pub struct OpState {
    resource_table: ResourceTable,
    extensions: HashMap<TypeId, Box<dyn Any>>,
    // ...
}

// Ops can access shared state
#[op2]
fn op_read(
    state: &mut OpState,
    #[smi] rid: ResourceId,
    #[buffer] buf: &mut [u8],
) -> Result<usize, AnyError> {
    let resource = state.resource_table.get::<FileResource>(rid)?;
    resource.read(buf)
}
```

### 5. **Performance Optimizations**

Deno uses several techniques for performance:

- **Fast ops**: Direct V8 fast API calls for simple ops
- **Zero-copy buffers**: Pass ArrayBuffers without copying
- **Lazy/Eager scheduling**: Control when ops are polled
- **Ring buffer**: Recent promises in array, older in Map

```rust
// Fast op - no promise overhead for sync operations
#[op2(fast)]
fn op_add(#[smi] a: i32, #[smi] b: i32) -> i32 {
    a + b
}
```
