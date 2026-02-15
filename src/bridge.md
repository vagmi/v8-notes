# Bridging Rust and JavaScript

The power of a custom runtime comes from exposing Rust capabilities to JavaScript. This bridge is where Rust functions become JavaScript APIs.

## Native Function Callbacks

V8 function callbacks have a specific signature using `rusty_v8`'s pinned scope pattern:

```rust
fn print_callback(
    scope: &mut v8::PinScope,
    args: v8::FunctionCallbackArguments,
    mut _retval: v8::ReturnValue,
) {
    // Get first argument
    if args.length() > 0 {
        let arg = args.get(0);
        let s = arg.to_string(scope).unwrap();
        let rust_string = s.to_rust_string_lossy(scope);
        println!("JS: {}", rust_string);
    }
}
```

**Key components:**
- **scope**: The HandleScope for creating V8 values
- **args**: Access function arguments with `.get(index)`
- **retval**: Set the return value with `.set(value)`

## Registering Native Functions

To make a Rust function callable from JavaScript, attach it to the global object:

```rust
pub fn setup_bindings(scope: &mut v8::PinScope) {
    let global = scope.get_current_context().global(scope);

    // Create function from callback
    let func = v8::FunctionTemplate::new(scope, print_callback);
    let func = func.get_function(scope).unwrap();

    // Attach to global with name "print"
    let name = v8::String::new(scope, "print").unwrap();
    global.set(scope, name.into(), func.into());
}
```

Now JavaScript can call `print("Hello from JS!")`

## Type Conversion

Converting between Rust and JavaScript types:

**Rust → JavaScript:**
```rust
// Strings
let js_str = v8::String::new(scope, "hello").unwrap();

// Numbers
let js_num = v8::Number::new(scope, 42.5);

// Booleans
let js_bool = v8::Boolean::new(scope, true);

// Objects
let obj = v8::Object::new(scope);
let key = v8::String::new(scope, "foo").unwrap();
let val = v8::String::new(scope, "bar").unwrap();
obj.set(scope, key.into(), val.into());
```

**JavaScript → Rust:**
```rust
// Check type
if value.is_string() {
    let s = value.to_string(scope).unwrap();
    let rust_str = s.to_rust_string_lossy(scope);
}

if value.is_number() {
    let num = value.number_value(scope).unwrap(); // f64
}

if value.is_boolean() {
    let b = value.boolean_value(scope); // bool
}
```

## The Closure Problem

Consider this async function:

```rust
fn setup_fetch(scope: &mut v8::PinScope, scheduler_tx: mpsc::UnboundedSender<SchedulerMessage>) {
    // ❌ This doesn't work - can't capture non-Copy types in callback
    let func = v8::Function::new(scope, |scope, args, _retval| {
        let url = args.get(0).to_rust_string_lossy(scope);
        scheduler_tx.send(SchedulerMessage::Fetch(1, url)); // ❌ Can't capture scheduler_tx
    });
}
```

**Problem:** Function callbacks can't capture non-`Copy` types like channels.

## Solution: External State Pattern

Store state in V8's heap using `v8::External`:

```rust
// 1. Define state struct
struct FetchState {
    scheduler_tx: mpsc::UnboundedSender<SchedulerMessage>,
}

// 2. Store state in V8 heap
pub fn setup_fetch(scope: &mut v8::PinScope, scheduler_tx: mpsc::UnboundedSender<SchedulerMessage>) {
    let global = scope.get_current_context().global(scope);

    // Box the state and convert to raw pointer
    let state = FetchState { scheduler_tx };
    let state_ptr = Box::into_raw(Box::new(state)) as *mut std::ffi::c_void;

    // Wrap in v8::External
    let external = v8::External::new(scope, state_ptr);
    let key = v8::String::new(scope, "__fetchState").unwrap();
    global.set(scope, key.into(), external.into());

    // Create native function
    let native_fetch = v8::Function::new(scope, native_fetch_callback).unwrap();
    let name = v8::String::new(scope, "__nativeFetch").unwrap();
    global.set(scope, name.into(), native_fetch.into());
}

// 3. Retrieve state in callback
fn native_fetch_callback(
    scope: &mut v8::PinScope,
    args: v8::FunctionCallbackArguments,
    mut _retval: v8::ReturnValue,
) {
    // Get the state from global
    let global = scope.get_current_context().global(scope);
    let key = v8::String::new(scope, "__fetchState").unwrap();
    let external_val = global.get(scope, key.into()).unwrap();

    if let Ok(external) = v8::Local::<v8::External>::try_from(external_val) {
        let state_ptr = external.value() as *const FetchState;
        let state = unsafe { &*state_ptr };

        // Now we can use the channel!
        let id = args.get(0).number_value(scope).unwrap_or(0.0) as u64;
        let url = args.get(1).to_rust_string_lossy(scope);
        let _ = state.scheduler_tx.send(SchedulerMessage::Fetch(id, url));
    }
}
```

This pattern is used in toyjs for timers and fetch (see `runtime/timers.rs` and `runtime/fetch.rs`).

## JavaScript Wrapper Pattern

Native functions are low-level. Wrap them with JavaScript for a clean API:

```rust
// Compile JavaScript wrapper
let js_code = r#"
    globalThis.fetchCallbacks = new Map();
    globalThis.nextFetchId = 1;

    globalThis.fetch = function(url) {
        return new Promise((resolve, reject) => {
            const id = globalThis.nextFetchId++;
            globalThis.fetchCallbacks.set(id, { resolve, reject });
            __nativeFetch(id, url); // Call native function
        });
    };

    globalThis.__executeFetchSuccess = function(id, body) {
        const callbacks = globalThis.fetchCallbacks.get(id);
        if (callbacks) {
            globalThis.fetchCallbacks.delete(id);
            callbacks.resolve({ text: () => Promise.resolve(body) });
        }
    };
"#;

let code_str = v8::String::new(scope, js_code).unwrap();
let script = v8::Script::compile(scope, code_str, None).unwrap();
script.run(scope).unwrap();
```

**Benefits:**
- Clean JavaScript API (`fetch(url)` instead of `__nativeFetch(id, url)`)
- Promise support
- Callback management in JavaScript (easier than Rust `HashMap<u64, v8::Global<v8::Function>>`)

## The Complete Flow

1. **JavaScript calls** `fetch("url")`
2. **JS wrapper** generates ID, stores callbacks, calls `__nativeFetch(id, url)`
3. **Rust callback** extracts state, sends `SchedulerMessage::Fetch` to event loop
4. **Event loop** performs async HTTP request
5. **Event loop** sends `CallbackMessage::FetchSuccess(id, body)` back
6. **Rust** calls `__executeFetchSuccess(id, body)` in V8
7. **JS wrapper** retrieves stored callbacks, resolves Promise

This pattern powers all async operations in JavaScript runtimes.
