# Zig Wiki

A practical Zig reference built from real experience — not theory. Every article comes from building an actual production HTTP server with SQLite, C interop, and JSON APIs using Zig 0.14.

## Articles

### Getting Started
- [Installation](installation.md) — Download, setup, verify
- [Project Setup](project-setup.md) — build.zig, build.zig.zon, directory structure
- [Hello World](hello-world.md) — Your first Zig program

### Core Concepts
- [Types and Variables](types-and-variables.md) — Integers, floats, booleans, strings, arrays, slices
- [Memory and Allocators](memory-and-allocators.md) — No GC, explicit allocation, allocator patterns
- [Error Handling](error-handling.md) — Error unions, try, catch, errdefer
- [Optionals](optionals.md) — Null safety, `?T`, `orelse`, `if` with capture
- [Control Flow](control-flow.md) — if, while, for, switch, defer
- [Structs and Methods](structs-and-methods.md) — Zig's version of OOP
- [Comptime](comptime.md) — Compile-time evaluation, generics, type reflection

### Working with Data
- [Strings](strings.md) — Slices, formatting, comparison, sentinel-terminated
- [Collections](collections.md) — ArrayList, HashMap, ArrayHashMap
- [JSON](json.md) — Parsing and serialization with std.json

### Systems Programming
- [C Interop](c-interop.md) — @cImport, calling C functions, pointer conversion
- [Pointers](pointers.md) — Single-item, many-item, C pointers, slices
- [File I/O](file-io.md) — Reading, writing, std.fs

### Networking
- [HTTP Server](http-server.md) — std.http.Server, routing, request/response
- [Networking Basics](networking.md) — std.net, TCP, address parsing

### Practical Patterns
- [SQLite Integration](sqlite-integration.md) — Full example with C interop
- [Build System](build-system.md) — build.zig in depth, C sources, linking
- [Gotchas and Pitfalls](gotchas.md) — Common mistakes and Zig 0.14 version differences

## Who is this for

Developers coming from JavaScript, Go, Python, or C who want to learn Zig by example. Each article explains Zig concepts in terms you already know, with working code you can copy.

## Zig version

All examples target **Zig 0.14.0**. The [Gotchas](gotchas.md) page documents API differences between versions.
