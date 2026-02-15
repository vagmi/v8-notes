# Security & Resource Limits

Running untrusted code requires strict boundaries. V8 provides some mechanisms, and we implement others in Rust.

## Memory Limits

V8 allows setting heap limits on the `Isolate`.

```rust
let params = v8::CreateParams::default()
    .heap_limits(0, 128 * 1024 * 1024); // Max 128MB
```

### The ArrayBuffer Problem

`ArrayBuffer` memory is allocated *outside* the V8 heap. To limit this, we need a custom `ArrayBuffer::Allocator`.

In `src/security/array_buffer_allocator.rs`:

```rust
pub struct CustomAllocator {
    limit: usize,
    current: AtomicUsize,
    limit_hit_flag: Arc<AtomicBool>,
}

impl v8::ArrayBufferAllocator for CustomAllocator {
    fn allocate(&mut self, length: usize) -> *mut c_void {
        if self.current + length > self.limit {
            self.limit_hit_flag.store(true, Ordering::SeqCst);
            return std::ptr::null_mut(); // Allocation failed
        }
        // ... perform allocation ...
    }
}
```

If allocation fails, V8 throws a RangeError or terminates.

## CPU Time Limits

Preventing infinite loops is critical.

### Linux: `timer_create`

On Linux, we use POSIX timers (`timer_create`) with `CLOCK_THREAD_CPUTIME_ID` to measure actual CPU time used by the thread. When the limit is reached, a signal (SIGALRM) is sent.

The signal handler calls `isolate.terminate_execution()`.

See `src/security/cpu_enforcer.rs`.

```rust
// On initialization
timer_create(CLOCK_THREAD_CPUTIME_ID, ...);
timer_settime(..., limit);

// Signal handler
extern "C" fn handler(...) {
    // This is tricky: accessing isolate from signal handler is unsafe.
    // openworkers uses a global map or a specialized mechanism
    // or checks the timer in the event loop if precision allows.
}
```

*Note: `openworkers-runtime-v8` actually uses a separate thread or mechanism depending on implementation details in `CpuEnforcer`.*

## Wall-Clock Limits

To prevent hanging on IO, we use a `TimeoutGuard`. This is a separate thread that sleeps for the limit duration. If the worker hasn't finished when it wakes up, it calls `terminate_execution()`.

```rust
pub struct TimeoutGuard {
    handle: IsolateHandle,
}

impl TimeoutGuard {
    pub fn new(handle: IsolateHandle, ms: u64) -> Self {
        thread::spawn(move || {
            sleep(ms);
            handle.terminate_execution();
        });
        // ...
    }
}
```

## Termination Handling

When `terminate_execution()` is called, V8 throws an uncatchable exception. We catch this in the Rust orchestration layer and return a `TerminationReason` (CPU limit, Wall-clock limit, etc.).
