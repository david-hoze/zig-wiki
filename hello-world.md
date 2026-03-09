# Hello World

## Minimal program

```zig
// main.zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, world!\n", .{});
}
```

Run it:
```bash
zig run main.zig
```

## With proper error handling

```zig
const std = @import("std");

pub fn main() !void {
    // Get a writer to stdout
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello, {s}!\n", .{"world"});
}
```

The `!void` return type means "this function can return an error OR void." `try` propagates errors upward. If `main` returns an error, Zig prints a stack trace.

## Formatted printing

```zig
std.debug.print("Number: {d}\n", .{42});           // "Number: 42"
std.debug.print("Float: {d:.2}\n", .{3.14159});     // "Float: 3.14"
std.debug.print("String: {s}\n", .{"hello"});       // "String: hello"
std.debug.print("Hex: {x}\n", .{255});              // "Hex: ff"
std.debug.print("Bool: {}\n", .{true});              // "Bool: true"
std.debug.print("Multiple: {d} + {d} = {d}\n", .{1, 2, 3}); // "Multiple: 1 + 2 = 3"
```

**Key difference from other languages**: The format arguments go in an anonymous struct literal `.{...}`, not as separate arguments. This is because Zig uses comptime (compile-time) type checking on format strings.

## Logging

For application logging, use `std.log` instead of `std.debug.print`:

```zig
const std = @import("std");

pub fn main() !void {
    std.log.info("Server starting on port {d}", .{8080});
    std.log.warn("Disk space low: {d}MB remaining", .{42});
    std.log.err("Failed to connect: {s}", .{"timeout"});
}
```

`std.log` writes to stderr with a severity prefix. `std.debug.print` also writes to stderr but with no prefix.
