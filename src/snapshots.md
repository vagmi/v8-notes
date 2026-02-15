# V8 Snapshots and Performance

Building a JavaScript runtime is easy. Building a *fast* JavaScript runtime requires understanding V8's most powerful optimization: snapshots. This chapter will teach you how to achieve 50-80% startup time improvements using V8's snapshot feature.

## What Are V8 Snapshots?

A V8 snapshot is a serialized representation of the JavaScript heap at a specific point in time. Think of it as a "save game" for your JavaScript runtime:

- All JavaScript code is already parsed and compiled
- Objects and prototypes are pre-allocated  
- The heap layout is optimized and ready to use

Instead of starting from scratch every time, V8 can deserialize this snapshot and have a fully initialized JavaScript environment in milliseconds.

```
Without Snapshot:                    With Snapshot:
┌──────────────┐                    ┌──────────────┐
│ Start V8     │ 5ms                │ Start V8     │ 5ms
├──────────────┤                    ├──────────────┤
│ Parse JS     │ 40ms               │ Load Snapshot│ 10ms
├──────────────┤                    ├──────────────┤
│ Compile      │ 30ms               │ (Ready!)     │
├──────────────┤                    └──────────────┘
│ Initialize   │ 25ms                    Total: 15ms
├──────────────┤
│ Setup APIs   │ 20ms
└──────────────┘
    Total: 120ms
```

## The JavaScript Startup Problem

Every time you start a JavaScript runtime without snapshots, V8 must:

1. **Parse Source Code**: Convert JavaScript text into an Abstract Syntax Tree (AST)
2. **Compile to Bytecode**: Transform AST into executable bytecode
3. **Initialize Built-ins**: Create `Object`, `Array`, `Function` prototypes
4. **Set Up Globals**: Create `console`, `setTimeout`, and other APIs
5. **Load Polyfills**: Parse and execute Web API implementations

Let's measure the real cost:

```rust
// Measuring startup overhead
let start = std::time::Instant::now();

// Without snapshot
let isolate = v8::Isolate::new(Default::default());
let runtime = JsRuntime::new(isolate);
runtime.execute_script("console.log('Hello')");

println!("Startup time: {:?}", start.elapsed());
// Output: Startup time: 123.45ms

// With snapshot
let isolate = v8::Isolate::new(CreateParams::default()
    .snapshot_blob(SNAPSHOT_BLOB));
let runtime = JsRuntime::new(isolate);
runtime.execute_script("console.log('Hello')");

println!("Startup time: {:?}", start.elapsed());  
// Output: Startup time: 15.23ms (87% improvement!)
```

### Real-World Impact

Consider these scenarios:

**Serverless Functions**:
- Cold start: 120ms (without snapshot) vs 15ms (with snapshot)
- 10,000 cold starts/day = 17 minutes saved daily
- AWS Lambda bills per 1ms = Direct cost savings

**CLI Tools**:
```bash
# Without snapshot
$ time mytool --version
0.124s user

# With snapshot  
$ time mytool --version
0.015s user
```

**Embedded Runtimes**:
- IoT devices with limited CPU
- Mobile apps with JavaScript engines
- Every millisecond affects battery life

## How V8 Snapshots Work

### The Serialization Process

V8 snapshots work by freezing the JavaScript heap at a specific moment and serializing it to a binary blob:

```
JavaScript Heap                    Serialization                 Snapshot Blob
┌─────────────────┐               ┌─────────────┐              ┌────────────┐
│ Global Object   │──────────────>│             │              │ Header     │
├─────────────────┤               │  Traverse   │              ├────────────┤
│ Object.prototype│──────────────>│  Object     │──────────>   │ Objects    │
├─────────────────┤               │  Graph      │              ├────────────┤
│ Array.prototype │──────────────>│             │              │ Code       │
├─────────────────┤               │  Serialize  │              ├────────────┤
│ Custom Objects  │──────────────>│  to Binary  │              │ Strings    │
└─────────────────┘               └─────────────┘              └────────────┘
```

### Key Concepts

**1. Heap Serialization**:
V8 walks the entire object graph starting from the global object, serializing:
- Object properties and prototypes
- Function bytecode and metadata
- String contents and symbols
- Internal V8 structures

**2. Pointer Fixup**:
Since memory addresses change between runs, V8 converts all pointers to relative offsets:
```
Before: Object* ptr = 0x7fff12345678
After:  uint32_t offset = 0x1234  // Relative to heap base
```

**3. Lazy Deserialization**:
Not everything is deserialized immediately. V8 can defer some objects until first access.

### Context vs Isolate Snapshots

V8 supports two types of snapshots:

**Isolate Snapshot** (Built into V8):
- Contains JavaScript built-ins
- Shared by all contexts
- About 1.5MB compressed
- Includes: `Object`, `Array`, `Function`, etc.

**Context Snapshot** (Custom):
- Your application code
- Per-context initialization
- Typically 2-10MB
- Includes: Your APIs, polyfills, libraries

```rust
// Creating a context snapshot
let snapshot_creator = v8::SnapshotCreator::new(None);
let isolate = unsafe { snapshot_creator.get_owned_isolate() };

{
    let scope = v8::HandleScope::new(isolate);
    let context = v8::Context::new(&scope);
    let scope = v8::ContextScope::new(&scope, context);
    
    // Add your JavaScript code
    execute_script(&scope, include_str!("web_apis.js"));
    
    // Set as default context
    snapshot_creator.set_default_context(context);
}

// Extract the snapshot
let blob = snapshot_creator.create_blob(
    v8::FunctionCodeHandling::Keep  // Keep compiled code!
);
```

## Implementation Guide

Let's implement a production-ready snapshot system based on openworkers-runtime-v8:

### Step 1: Project Structure

```
my-runtime/
├── src/
│   ├── main.rs           # Runtime entry point
│   ├── snapshot.rs       # Snapshot creation logic
│   └── js/
│       ├── core.js       # Core utilities
│       ├── console.js    # Console implementation
│       └── web_apis.js   # Web API polyfills
├── build.rs              # Build script
└── Cargo.toml
```

### Step 2: Create the Snapshot Generator

```rust
// src/snapshot.rs
use v8;

pub struct SnapshotCreator {
    files: Vec<(&'static str, &'static str)>,
}

impl SnapshotCreator {
    pub fn new() -> Self {
        Self {
            files: vec![
                ("core.js", include_str!("js/core.js")),
                ("console.js", include_str!("js/console.js")),
                ("web_apis.js", include_str!("js/web_apis.js")),
            ],
        }
    }

    pub fn create_snapshot(&self) -> Vec<u8> {
        // Initialize V8
        let platform = v8::new_default_platform(0, false).make_shared();
        v8::V8::initialize_platform(platform);
        v8::V8::initialize();

        let snapshot_creator = v8::SnapshotCreator::new(None);
        let isolate = unsafe { snapshot_creator.get_owned_isolate() };

        {
            let scope = &mut v8::HandleScope::new(isolate);
            let context = v8::Context::new(scope);
            let scope = &mut v8::ContextScope::new(scope, context);

            // Execute all JavaScript files
            for (name, source) in &self.files {
                self.execute_script(scope, name, source)
                    .expect("Failed to execute script");
            }

            // IMPORTANT: Set the context as default
            snapshot_creator.set_default_context(context);
        }

        // Create blob with compiled code
        let startup_data = snapshot_creator.create_blob(
            v8::FunctionCodeHandling::Keep
        ).unwrap();
        
        startup_data.to_vec()
    }

    fn execute_script(
        &self,
        scope: &mut v8::HandleScope,
        name: &str,
        source: &str,
    ) -> Result<(), String> {
        let source = v8::String::new(scope, source).unwrap();
        let origin = self.create_origin(scope, name);
        
        let script = v8::Script::compile(scope, source, Some(&origin))
            .ok_or_else(|| "Failed to compile script")?;
            
        script.run(scope)
            .ok_or_else(|| "Failed to execute script")?;
            
        Ok(())
    }

    fn create_origin<'s>(
        &self,
        scope: &mut v8::HandleScope<'s>,
        name: &str,
    ) -> v8::ScriptOrigin<'s> {
        let name = v8::String::new(scope, name).unwrap();
        v8::ScriptOrigin::new(scope, name.into(), 0, 0, false, 0, None, false, false, false)
    }
}
```

### Step 3: Build-Time Snapshot Generation

```rust
// build.rs
use std::env;
use std::fs;
use std::path::PathBuf;

fn main() {
    println!("cargo:rerun-if-changed=src/js/");
    
    // Only generate snapshot for release builds
    if env::var("PROFILE").unwrap() == "release" {
        generate_snapshot();
    }
}

fn generate_snapshot() {
    // Create snapshot
    let creator = snapshot::SnapshotCreator::new();
    let blob = creator.create_snapshot();
    
    // Write to file
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());
    let snapshot_path = out_dir.join("snapshot.bin");
    fs::write(&snapshot_path, blob).unwrap();
    
    println!("cargo:rustc-env=SNAPSHOT_PATH={}", snapshot_path.display());
}
```

### Step 4: Loading Snapshots at Runtime

```rust
// src/main.rs
use once_cell::sync::Lazy;
use std::sync::Arc;

// The snapshot is loaded once and cached
static SNAPSHOT: Lazy<Option<Arc<[u8]>>> = Lazy::new(|| {
    // Try to load snapshot from compile-time path
    if let Some(path) = option_env!("SNAPSHOT_PATH") {
        match std::fs::read(path) {
            Ok(data) => {
                println!("Loaded snapshot: {} bytes", data.len());
                return Some(data.into());
            }
            Err(e) => {
                eprintln!("Failed to load snapshot: {}", e);
            }
        }
    }
    None
});

pub struct Runtime {
    isolate: v8::OwnedIsolate,
}

impl Runtime {
    pub fn new() -> Self {
        let mut params = v8::CreateParams::default();
        
        // Use snapshot if available
        if let Some(snapshot) = &*SNAPSHOT {
            params = params.snapshot_blob(snapshot.as_ref());
        }
        
        let isolate = v8::Isolate::new(params);
        
        Self { isolate }
    }
}
```

### Step 5: The Memory Management Pattern

A critical pattern in V8 snapshot usage is the "leak to static" pattern:

```rust
// Why we leak snapshot data:
pub fn load_snapshot() -> &'static [u8] {
    let data = std::fs::read("snapshot.bin").unwrap();
    
    // This is SAFE and REQUIRED because:
    // 1. V8 needs the snapshot data to live for the entire program
    // 2. The data is read-only after loading
    // 3. We only load it once
    Box::leak(data.into_boxed_slice())
}

// DON'T do this - snapshot will be freed!
pub fn bad_load_snapshot() -> Vec<u8> {
    std::fs::read("snapshot.bin").unwrap()
    // Vec is dropped here, V8 will crash!
}
```

## What to Include in Snapshots

### ✅ DO Include: Pure JavaScript

Pure JavaScript code is perfect for snapshots:

```javascript
// js/text_encoding.js - Great for snapshots!
class TextEncoder {
    encode(string) {
        // Pure JS implementation
        const bytes = [];
        for (let i = 0; i < string.length; i++) {
            const code = string.charCodeAt(i);
            if (code < 0x80) {
                bytes.push(code);
            } else if (code < 0x800) {
                bytes.push(0xc0 | (code >> 6));
                bytes.push(0x80 | (code & 0x3f));
            } else {
                // ... more UTF-8 encoding
            }
        }
        return new Uint8Array(bytes);
    }
}

// js/url.js - Also great!
class URL {
    constructor(url, base) {
        // Pure JS URL parsing
        this.#parse(url, base);
    }
    
    #parse(url, base) {
        // Regular expressions and string manipulation
        // No native dependencies!
    }
}
```

### ✅ DO Include: Polyfills and Utilities

```javascript
// js/polyfills.js
if (!Array.prototype.at) {
    Array.prototype.at = function(index) {
        if (index < 0) index += this.length;
        return this[index];
    };
}

// Utility functions
function debounce(fn, delay) {
    let timeout;
    return function(...args) {
        clearTimeout(timeout);
        timeout = setTimeout(() => fn.apply(this, args), delay);
    };
}
```

### ❌ DON'T Include: Native Bindings

Native function callbacks cannot be snapshotted:

```rust
// This CANNOT go in a snapshot!
fn setup_console(scope: &mut v8::HandleScope) {
    let console = v8::Object::new(scope);
    
    // Function::new creates a native callback - not snapshottable!
    let log_fn = v8::Function::new(scope, console_log_callback).unwrap();
    let key = v8::String::new(scope, "log").unwrap();
    console.set(scope, key.into(), log_fn.into());
}

// Instead, use a two-phase approach:
// 1. Snapshot phase: Create the object structure
// 2. Runtime phase: Bind native functions
```

### ❌ DON'T Include: Runtime-Specific State

```javascript
// BAD: Don't include configuration
const API_KEY = process.env.API_KEY;  // Different each run!

// BAD: Don't include mutable global state
let requestCounter = 0;  // Will be wrong after restore

// BAD: Don't include time-dependent values
const startTime = Date.now();  // Frozen in snapshot!

// GOOD: Initialize these at runtime
let API_KEY;
let requestCounter;
let startTime;

function initializeRuntime(config) {
    API_KEY = config.apiKey;
    requestCounter = 0;
    startTime = Date.now();
}
```

### Real Example: Web API Organization

Here's how openworkers-runtime-v8 organizes Web APIs:

```javascript
// In snapshot: Pure JS implementations
class Headers {
    #headers = new Map();
    
    constructor(init) {
        if (init) this.#fill(init);
    }
    
    append(name, value) {
        // Pure JS logic
        name = this.#normalizeName(name);
        value = this.#normalizeValue(value);
        
        const existing = this.#headers.get(name);
        if (existing) {
            this.#headers.set(name, existing + ', ' + value);
        } else {
            this.#headers.set(name, value);
        }
    }
    
    // More methods...
}

// At runtime: Native bindings added separately
function fetch(url, init) {
    // This calls into native code
    return __nativeFetch(url, init);
}
```

## Performance Analysis

### Measuring Snapshot Benefits

Let's create a comprehensive benchmark:

```rust
// benches/startup.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_startup_no_snapshot(c: &mut Criterion) {
    c.bench_function("startup_no_snapshot", |b| {
        b.iter(|| {
            let isolate = v8::Isolate::new(Default::default());
            let runtime = Runtime::new_no_snapshot(isolate);
            black_box(runtime.execute("1 + 1"));
        });
    });
}

fn bench_startup_with_snapshot(c: &mut Criterion) {
    c.bench_function("startup_with_snapshot", |b| {
        b.iter(|| {
            let params = v8::CreateParams::default()
                .snapshot_blob(&*SNAPSHOT);
            let isolate = v8::Isolate::new(params);
            let runtime = Runtime::new(isolate);
            black_box(runtime.execute("1 + 1"));
        });
    });
}
```

### Typical Results

```
Benchmark Results:
┌─────────────────────────┬──────────┬────────────┬─────────┐
│ Test                    │ Time     │ vs Baseline│ Speedup │
├─────────────────────────┼──────────┼────────────┼─────────┤
│ No Snapshot             │ 124.3ms  │ baseline   │ 1.0x    │
│ With Snapshot           │  18.7ms  │ -105.6ms   │ 6.6x    │
│ With Snapshot (cached)  │  15.2ms  │ -109.1ms   │ 8.2x    │
└─────────────────────────┴──────────┴────────────┴─────────┘

Memory Usage:
┌─────────────────────────┬──────────┬─────────────┐
│ Metric                  │ Size     │ Notes       │
├─────────────────────────┼──────────┼─────────────┤
│ Snapshot file size      │ 3.2MB    │ Compressed  │
│ Runtime memory          │ 8.4MB    │ Decompressed│
│ Additional per context  │ 0.8MB    │ Shared data │
└─────────────────────────┴──────────┴─────────────┘
```

### First Execution Benefits

Snapshots also improve first execution:

```javascript
// Without snapshot: Must compile on first run
function fibonacci(n) {  // +5ms to compile
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// With snapshot: Already compiled!
fibonacci(30);  // Runs immediately
```

## Advanced Topics

### Multiple Contexts from One Snapshot

Create isolated contexts efficiently:

```rust
pub struct ContextPool {
    isolate: v8::OwnedIsolate,
    snapshot: Arc<[u8]>,
}

impl ContextPool {
    pub fn create_context(&mut self) -> v8::Global<v8::Context> {
        let scope = &mut v8::HandleScope::new(&mut self.isolate);
        
        // Each context is independent but shares snapshot
        let context = v8::Context::new(scope);
        
        // Add runtime-specific bindings
        self.setup_native_bindings(scope, context);
        
        v8::Global::new(scope, context)
    }
}
```

### Debugging Snapshot Issues

Common problems and solutions:

**1. "Invalid snapshot" Error**:
```rust
// Add version checking
const SNAPSHOT_VERSION: u32 = 1;

struct SnapshotHeader {
    magic: [u8; 4],  // "V8SS"
    version: u32,
    v8_version: [u8; 16],
    checksum: u32,
}

// Validate before use
fn validate_snapshot(data: &[u8]) -> Result<(), Error> {
    let header = parse_header(data)?;
    if header.version != SNAPSHOT_VERSION {
        return Err(Error::VersionMismatch);
    }
    Ok(())
}
```

**2. Missing Functions After Snapshot**:
```javascript
// Debug helper
function verifySnapshot() {
    const required = [
        'TextEncoder',
        'URL',
        'Headers',
        'Request',
        'Response'
    ];
    
    const missing = required.filter(name => 
        !(name in globalThis)
    );
    
    if (missing.length > 0) {
        throw new Error(`Snapshot missing: ${missing.join(', ')}`);
    }
}
```

### Cross-Platform Considerations

Snapshots are architecture-specific:

```rust
// Generate per-platform snapshots
#[cfg(target_arch = "x86_64")]
const SNAPSHOT: &[u8] = include_bytes!("snapshot_x64.bin");

#[cfg(target_arch = "aarch64")]
const SNAPSHOT: &[u8] = include_bytes!("snapshot_arm64.bin");

// Or use runtime detection
fn load_platform_snapshot() -> Vec<u8> {
    let arch = if cfg!(target_arch = "x86_64") {
        "x64"
    } else if cfg!(target_arch = "aarch64") {
        "arm64"
    } else {
        panic!("Unsupported architecture");
    };
    
    let path = format!("snapshot_{}.bin", arch);
    std::fs::read(path).unwrap()
}
```

### Security Implications

Snapshots require trust:

```rust
// Add integrity checking
use sha2::{Sha256, Digest};

fn verify_snapshot_integrity(data: &[u8], expected_hash: &str) -> bool {
    let mut hasher = Sha256::new();
    hasher.update(data);
    let result = hasher.finalize();
    let hash = format!("{:x}", result);
    hash == expected_hash
}

// Sign snapshots in production
fn create_signed_snapshot() -> SignedSnapshot {
    let data = create_snapshot();
    let signature = sign_with_private_key(&data);
    SignedSnapshot { data, signature }
}
```

## Best Practices for Runtime Authors

### 1. Architecture Guidelines

**Separate Layers**:
```
┌─────────────────────────┐
│   Native Bindings       │ <- Runtime setup
├─────────────────────────┤
│   JavaScript APIs       │ <- In snapshot
├─────────────────────────┤
│   Pure JS Polyfills     │ <- In snapshot
├─────────────────────────┤
│   V8 Built-ins          │ <- V8's snapshot
└─────────────────────────┘
```

**Dependency Injection Pattern**:
```javascript
// snapshot_init.js
let __fetch, __crypto, __fs;

class Request {
    async arrayBuffer() {
        // Uses injected fetch
        const response = await __fetch(this.url);
        return response.arrayBuffer();
    }
}

// runtime_init.js
function injectNatives(natives) {
    __fetch = natives.fetch;
    __crypto = natives.crypto;
    __fs = natives.fs;
}
```

### 2. Build Pipeline Integration

```yaml
# .github/workflows/release.yml
name: Build Release

jobs:
  build:
    strategy:
      matrix:
        platform: [ubuntu-latest, macos-latest, windows-latest]
        arch: [x64, arm64]
    
    steps:
      - name: Generate Snapshot
        run: |
          cargo build --bin generate-snapshot --release
          ./target/release/generate-snapshot
          
      - name: Verify Snapshot
        run: cargo test --test snapshot_verification
        
      - name: Bundle Snapshot
        run: |
          mkdir -p dist
          cp snapshot.bin dist/snapshot_${PLATFORM}_${ARCH}.bin
          sha256sum dist/snapshot_*.bin > dist/checksums.txt
```

### 3. Version Management

```rust
// Embed version info in snapshot
const SNAPSHOT_METADATA: &str = r#"{
    "version": "1.0.0",
    "buildTime": "2024-01-15T10:30:00Z",
    "v8Version": "11.8.172.13",
    "features": ["textencoder", "url", "fetch"]
}"#;

// Runtime compatibility check
fn check_snapshot_compatibility(metadata: &SnapshotMetadata) -> Result<()> {
    let runtime_v8 = v8::V8::get_version();
    let snapshot_v8 = &metadata.v8_version;
    
    // Major.minor must match
    if !versions_compatible(runtime_v8, snapshot_v8) {
        return Err("V8 version mismatch");
    }
    Ok(())
}
```

### 4. Testing Snapshot Correctness

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_snapshot_equivalence() {
        // Create runtime without snapshot
        let mut runtime1 = Runtime::new_no_snapshot();
        runtime1.execute_file("js/web_apis.js");
        
        // Create runtime with snapshot
        let mut runtime2 = Runtime::new_with_snapshot();
        
        // Test same behavior
        let test_cases = vec![
            "new TextEncoder().encode('hello')",
            "new URL('https://example.com/path').pathname",
            "new Headers({'content-type': 'text/plain'}).get('content-type')",
        ];
        
        for test in test_cases {
            let result1 = runtime1.execute(test);
            let result2 = runtime2.execute(test);
            assert_eq!(result1, result2, "Snapshot divergence in: {}", test);
        }
    }
}
```

### 5. Performance Monitoring

```rust
// Track snapshot performance over time
struct StartupMetrics {
    snapshot_load_time: Duration,
    context_creation_time: Duration,
    first_execution_time: Duration,
    total_startup_time: Duration,
}

impl Runtime {
    pub fn new_with_metrics() -> (Self, StartupMetrics) {
        let start = Instant::now();
        
        let snapshot_start = Instant::now();
        let snapshot = load_snapshot();
        let snapshot_load_time = snapshot_start.elapsed();
        
        let context_start = Instant::now();
        let runtime = Self::create_with_snapshot(snapshot);
        let context_creation_time = context_start.elapsed();
        
        let exec_start = Instant::now();
        runtime.execute("1");  // Warm up
        let first_execution_time = exec_start.elapsed();
        
        let metrics = StartupMetrics {
            snapshot_load_time,
            context_creation_time,
            first_execution_time,
            total_startup_time: start.elapsed(),
        };
        
        (runtime, metrics)
    }
}
```

## Common Pitfalls and Solutions

### 1. The Native Function Trap

**Problem**: Trying to snapshot native function bindings
```javascript
// This breaks!
globalThis.readFile = createNativeFunction('fs_read');
```

**Solution**: Two-phase initialization
```javascript
// Phase 1: In snapshot
globalThis.readFile = function(...args) {
    return globalThis.__native_readFile(...args);
};

// Phase 2: At runtime
globalThis.__native_readFile = createNativeBinding('fs_read');
```

### 2. Mutable Global State

**Problem**: State frozen at snapshot time
```javascript
// BAD: This will always be the same!
const startupTime = Date.now();
const randomId = Math.random();
```

**Solution**: Lazy initialization
```javascript
// GOOD: Initialize at runtime
let _startupTime;
let _randomId;

function getStartupTime() {
    if (!_startupTime) _startupTime = Date.now();
    return _startupTime;
}
```

### 3. External Dependencies

**Problem**: Dynamic imports not available
```javascript
// This won't work in snapshot
import { something } from './dynamic.js';
```

**Solution**: Bundle or defer
```javascript
// Option 1: Bundle everything
// Use a bundler to include all dependencies

// Option 2: Runtime loading
async function loadDependencies() {
    const module = await import('./dynamic.js');
    return module;
}
```

## Decision Framework

### When to Use Snapshots

✅ **Definitely use snapshots for:**
- Production deployments
- Serverless/edge functions  
- CLI tools
- Embedded runtimes
- Any latency-sensitive application

❌ **Avoid snapshots for:**
- Development builds (slower iteration)
- Highly dynamic code that changes frequently
- When debugging V8 internals
- Prototyping phase

### Snapshot Size Guidelines

```
┌──────────────────┬────────────┬─────────────────┐
│ Snapshot Content │ Size       │ Recommendation  │
├──────────────────┼────────────┼─────────────────┤
│ Minimal (core)   │ 1-2 MB     │ Always include  │
│ Web APIs         │ 2-5 MB     │ Recommended     │
│ Framework code   │ 5-10 MB    │ Consider impact │
│ Application code │ 10+ MB     │ Carefully eval  │
└──────────────────┴────────────┴─────────────────┘
```

## Conclusion

V8 snapshots are the key to building fast JavaScript runtimes. By pre-compiling and serializing your JavaScript environment, you can achieve:

- **50-80% faster startup times**
- **Better cold start performance**  
- **Reduced memory pressure**
- **Consistent initialization**

Remember these key principles:

1. **Separate pure JavaScript from native bindings**
2. **Use the "leak to static" pattern for snapshot lifetime**
3. **Test snapshot equivalence thoroughly**
4. **Monitor performance continuously**
5. **Plan your architecture for snapshottability**

The complexity of implementing snapshots pays off in production performance. Whether you're building a serverless runtime, a CLI tool, or an embedded JavaScript engine, snapshots should be part of your optimization toolkit.

Happy snapshotting!