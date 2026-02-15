# Wrapper Scripts

Raw native bindings are often low-level. We use JavaScript wrapper scripts to provide a friendly Web API (like `fetch`, `Request`, `Response`).

## Polyfilling `fetch`

Instead of exposing `fetch` directly from Rust, we can expose a low-level `__nativeFetch` and wrap it.

```javascript
globalThis.fetch = function(url, options) {
    return new Promise((resolve, reject) => {
        __nativeFetch(url, options, (responseMeta) => {
            // Create Response object
            const response = new Response(null, responseMeta);
            resolve(response);
        }, (error) => {
            reject(new Error(error));
        });
    });
};
```

This allows us to implement the standard `Promise` API in JS, while the heavy lifting happens in Rust.
