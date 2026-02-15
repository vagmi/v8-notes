# Building a JavaScript Runtime with V8 and Rust

Welcome to this guide on understanding V8 and building custom JavaScript runtimes using Rust. These notes focus on core V8 concepts and runtime architecture patterns.

## What is V8?

V8 is Google's open source high-performance JavaScript and WebAssembly engine, written in C++. It's the same engine that powers Chrome, Node.js, and Deno. V8 implements the ECMAScript specification and compiles JavaScript directly to native machine code using just-in-time (JIT) compilation.

## Why Build a Runtime?

Building a runtime helps you:
1. **Understand Runtime Internals** - Learn how Node.js and Deno work under the hood
2. **Embed Scripting** - Add JavaScript capabilities to your Rust applications
3. **Control the Environment** - Define exactly what JavaScript can access (security, resource limits)
4. **Expose Native Functionality** - Bridge your Rust code to JavaScript

## Prerequisites

- Rust programming language
- Basic understanding of JavaScript event loops and async programming
- `rusty_v8` crate (safe Rust bindings to V8)

## What You'll Learn

This book covers the fundamental concepts for building a JavaScript runtime:
1. Core V8 concepts (Isolates, Contexts, Handles, Scopes)
2. Runtime architecture and initialization
3. Event loop design with message passing
4. Bridging Rust and JavaScript with native bindings
5. ES Module loading and resolution
6. Async operations (timers, fetch, promises)
7. Data exchange between Rust and JS
8. Security considerations and resource limits

Throughout this book, we'll reference [toyjs](https://github.com/vagmi/toyjs) - a minimal teaching runtime that demonstrates these concepts in practice. The code examples show how these V8 concepts translate into working Rust code.
