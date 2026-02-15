# Data Exchange

Passing data between Rust and V8 requires converting between Rust types and V8 handles.

## Primitives

### Strings
V8 strings are UTF-16, but `rusty_v8` handles UTF-8 conversion.

```rust
// Rust -> JS
let v8_str = v8::String::new(scope, "Hello").unwrap();

// JS -> Rust
let v8_str = args.get(0).to_string(scope).unwrap();
let rust_str = v8_str.to_rust_string_lossy(scope);
```

### Numbers
```rust
// Rust -> JS
let v8_num = v8::Number::new(scope, 42.0);

// JS -> Rust
let val = args.get(0).number_value(scope).unwrap();
```

## Objects

```rust
// Create object
let obj = v8::Object::new(scope);

// Set property
let key = v8::String::new(scope, "foo").unwrap();
let val = v8::String::new(scope, "bar").unwrap();
obj.set(scope, key.into(), val.into());
```

## Typed Arrays (Binary Data)

For high-performance binary data (like request bodies), we use `Uint8Array` backed by an `ArrayBuffer`.

### Zero-Copy (ish) Transfer

To move data from Rust to JS without full copy logic in JS:

1.  Create an `ArrayBuffer` with a `BackingStore` from a Rust `Vec`.
2.  Create a `Uint8Array` view over it.

```rust
let data: Vec<u8> = vec![1, 2, 3];
let len = data.len();

// Create backing store (takes ownership of Vec)
let backing_store = v8::ArrayBuffer::new_backing_store_from_vec(data);

// Create ArrayBuffer
let buffer = v8::ArrayBuffer::with_backing_store(scope, &backing_store.make_shared());

// Create Uint8Array
let uint8_array = v8::Uint8Array::new(scope, buffer, 0, len).unwrap();
```

Note: `new_backing_store_from_vec` might involve a copy depending on alignment, but it's efficient. The `Vec` memory is now managed by V8 GC.

## JSON

For complex structures, serialization via JSON is often easiest, though slower than direct object manipulation.

```rust
// Rust -> JS via JSON
let json = serde_json::to_string(&my_struct).unwrap();
let json_str = v8::String::new(scope, &json).unwrap();
let parsed = v8::json::parse(scope, json_str.into()).unwrap();
```
