# ES Module Loading

Modern JavaScript uses ES modules (`import`/`export`). Unlike scripts, modules require a loader to resolve and fetch dependencies. V8 provides the infrastructure, but you must implement the loading logic.

## Scripts vs Modules

**Scripts** are simple - compile and run:
```rust
let code = "console.log('hello')";
let source = v8::String::new(scope, code).unwrap();
let script = v8::Script::compile(scope, source, None).unwrap();
script.run(scope);
```

**Modules** have dependencies:
```javascript
// main.js
import { add } from './math.js';
console.log(add(2, 3));

// math.js
export function add(a, b) {
    return a + b;
}
```

V8 needs your help to:
1. Resolve `'./math.js'` to an absolute path
2. Load the file from disk
3. Compile it as a module
4. Cache it for reuse
5. Link modules together

## Module Compilation

Compile a module with `v8::script_compiler::compile_module`:

```rust
let code = "export function add(a, b) { return a + b; }";
let source_str = v8::String::new(scope, code).unwrap();

// Create ScriptOrigin with is_module=true
let origin = v8::ScriptOrigin::new(
    scope,
    v8::String::new(scope, "math.js").unwrap().into(), // filename
    0,      // line_offset
    0,      // column_offset
    false,  // is_shared_cross_origin
    123,    // script_id
    None,   // source_map_url
    false,  // is_opaque
    false,  // is_wasm
    true,   // is_module ← important!
    None,   // host_defined_options
);

let mut source = v8::script_compiler::Source::new(source_str, Some(&origin));
let module = v8::script_compiler::compile_module(scope, &mut source).unwrap();
```

## Module Resolution Callback

When a module imports another module, V8 calls your **module resolver** callback:

```rust
fn module_resolver<'a>(
    context: v8::Local<'a, v8::Context>,
    specifier: v8::Local<'a, v8::String>,      // './math.js'
    _import_attributes: v8::Local<'a, v8::FixedArray>,
    referrer: v8::Local<'a, v8::Module>,       // main.js module
) -> Option<v8::Local<'a, v8::Module>> {
    // Your loading logic here
}
```

**Your responsibilities:**
1. Convert specifier (`'./math.js'`) to absolute path
2. Check cache for already-loaded module
3. If not cached, read file and compile module
4. Store in cache
5. Return the module

## The Module Loader Pattern

Create a global module cache:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex, OnceLock};

static LOADER: OnceLock<Mutex<FsModuleLoader>> = OnceLock::new();

pub struct FsModuleLoader {
    // Map: absolute_path → Global<Module>
    modules: HashMap<String, v8::Global<v8::Module>>,
    // Map: module_hash → absolute_path (for reverse lookup)
    paths: HashMap<i32, String>,
}

impl FsModuleLoader {
    // Global singleton
    pub fn global() -> Arc<Mutex<Self>> {
        LOADER.get_or_init(|| {
            Arc::new(Mutex::new(FsModuleLoader {
                modules: HashMap::new(),
                paths: HashMap::new(),
            }))
        }).clone()
    }

    pub fn store_module(&mut self, path: String, module: v8::Global<v8::Module>, hash: i32) {
        self.modules.insert(path.clone(), module);
        self.paths.insert(hash, path);
    }

    pub fn get_module(&self, path: &str) -> Option<&v8::Global<v8::Module>> {
        self.modules.get(path)
    }

    pub fn get_path_by_hash(&self, hash: i32) -> Option<&String> {
        self.paths.get(&hash)
    }
}
```

## Path Resolution

Resolve relative specifiers to absolute paths:

```rust
impl FsModuleLoader {
    pub fn resolve_path(referrer_path: &str, specifier: &str) -> Option<String> {
        let referrer = std::path::Path::new(referrer_path);
        let referrer_dir = referrer.parent()?;

        // Resolve relative path
        let resolved = referrer_dir.join(specifier);
        let canonical = resolved.canonicalize().ok()?;

        Some(canonical.to_string_lossy().into_owned())
    }
}
```

This handles:
- `'./math.js'` - same directory
- `'../utils/helper.js'` - parent directory
- `'./sub/module.js'` - subdirectory

## The Complete Resolver

```rust
fn module_resolver<'a>(
    context: v8::Local<'a, v8::Context>,
    specifier: v8::Local<'a, v8::String>,
    _import_attributes: v8::Local<'a, v8::FixedArray>,
    referrer: v8::Local<'a, v8::Module>,
) -> Option<v8::Local<'a, v8::Module>> {
    let scope_storage = std::pin::pin!(unsafe { v8::CallbackScope::new(context) });
    let scope = &mut scope_storage.init();

    let specifier_str = specifier.to_rust_string_lossy(scope);
    let referrer_hash = referrer.get_identity_hash();

    let loader = FsModuleLoader::global();

    // 1. Find referrer's path using hash
    let referrer_path = {
        let loader_guard = loader.lock().unwrap();
        loader_guard.get_path_by_hash(referrer_hash).cloned()?
    };

    // 2. Resolve specifier relative to referrer
    let mut resolved_path = FsModuleLoader::resolve_path(&referrer_path, &specifier_str)?;

    // 3. Auto-add .js extension if missing
    if !std::path::Path::new(&resolved_path).exists() {
        resolved_path = format!("{}.js", resolved_path);
    }

    // 4. Check cache
    {
        let loader_guard = loader.lock().unwrap();
        if let Some(cached_module) = loader_guard.get_module(&resolved_path) {
            return Some(v8::Local::new(scope, cached_module));
        }
    }

    // 5. Load from filesystem
    let code = std::fs::read_to_string(&resolved_path).ok()?;

    // 6. Compile as module
    let source_str = v8::String::new(scope, &code)?;
    let origin = v8::ScriptOrigin::new(
        scope,
        v8::String::new(scope, &resolved_path)?.into(),
        0, 0, false, 123, None, false, false,
        true, // is_module
        None,
    );
    let mut source = v8::script_compiler::Source::new(source_str, Some(&origin));
    let module = v8::script_compiler::compile_module(scope, &mut source)?;

    // 7. Store in cache
    let module_hash = module.get_identity_hash();
    let global_module = v8::Global::new(scope, module);
    {
        let mut loader_guard = loader.lock().unwrap();
        loader_guard.store_module(resolved_path, global_module, module_hash);
    }

    Some(module)
}
```

## Module Instantiation and Evaluation

After compiling, modules must be **instantiated** (link imports) and **evaluated** (run code):

```rust
pub fn execute_module(&mut self, code: &str) -> String {
    let scope = &mut v8::HandleScope::new(&mut self.isolate);
    let context = v8::Local::new(scope, &self.context);
    let scope = &mut v8::ContextScope::new(scope, context);
    let tc_scope = &mut v8::TryCatch::new(scope);

    // 1. Compile
    let source_str = v8::String::new(tc_scope, code).unwrap();
    let origin = v8::ScriptOrigin::new(
        tc_scope,
        v8::String::new(tc_scope, "main.js").unwrap().into(),
        0, 0, false, 123, None, false, false,
        true, // is_module
        None,
    );
    let mut source = v8::script_compiler::Source::new(source_str, Some(&origin));
    let module = v8::script_compiler::compile_module(tc_scope, &mut source).unwrap();

    // Store main module in cache
    let module_hash = module.get_identity_hash();
    let global_module = v8::Global::new(tc_scope, module);
    let loader = FsModuleLoader::global();
    let cwd = std::env::current_dir().unwrap().to_string_lossy().to_string();
    let main_path = format!("{}/main.js", cwd);
    {
        let mut loader_guard = loader.lock().unwrap();
        loader_guard.store_module(main_path, global_module, module_hash);
    }

    // 2. Instantiate (resolve imports)
    module.instantiate_module(tc_scope, module_resolver).unwrap();

    // 3. Evaluate (run code)
    let _result = module.evaluate(tc_scope).unwrap();

    "Module executed".to_string()
}
```

## Registering the Resolver

Tell V8 to use your resolver with `set_host_import_module_dynamically_callback`:

```rust
let mut isolate = v8::Isolate::new(params);
isolate.set_host_import_module_dynamically_callback(
    import_module_dynamically_callback
);
```

This enables dynamic `import()` at runtime (beyond static imports).

## Key Insights

1. **V8 provides infrastructure, not implementation** - You load and cache modules
2. **Module identity by hash** - Use `get_identity_hash()` to track modules
3. **Global cache essential** - Avoid loading the same module twice
4. **Path resolution is tricky** - Handle relative paths, missing extensions
5. **Instantiate before evaluate** - V8 must link imports first

This is the same pattern used by Node.js and Deno, though they add features like node_modules resolution, package.json, and HTTP imports.

For the complete implementation, see `modules/mod.rs` in toyjs.
