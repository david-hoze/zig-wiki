# Comptime

`comptime` means "evaluated at compile time." It's Zig's replacement for generics, macros, and template metaprogramming — but simpler.

## comptime variables

```zig
// This is computed at compile time — no runtime cost
comptime var x = 0;
comptime {
    x += 1;
    x += 1;
}
// x is 2, known at compile time
```

## comptime parameters (generics)

Zig has no `<T>` generic syntax. Instead, you pass types as comptime parameters:

```zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

const result = max(i32, 10, 20);    // Works with i32
const result2 = max(f64, 1.5, 2.5); // Works with f64
```

This is how `ArrayList` works:

```zig
// std.ArrayList is a function that returns a type
// ArrayList(u8) returns a struct type specialized for u8
var list = std.ArrayList(u8).init(allocator);
```

## comptime if

```zig
fn doThing(comptime debug: bool) void {
    if (debug) {
        // This branch is removed entirely in non-debug builds
        std.debug.print("Debug info\n", .{});
    }
}
```

## comptime loops

```zig
// Unroll a loop at compile time
inline for ([_][]const u8{ "a", "b", "c" }) |name| {
    std.debug.print("{s}\n", .{name});
}
```

## Format strings are comptime

This is why `std.debug.print` catches format errors at compile time:

```zig
std.debug.print("{d}\n", .{42});        // OK
// std.debug.print("{d}\n", .{"hello"}); // COMPILE ERROR — {d} can't format a string
```

The format string is a `comptime` parameter, so Zig checks it against the argument types during compilation.

## @typeInfo — reflect on types at comptime

```zig
fn isOptional(comptime T: type) bool {
    return @typeInfo(T) == .optional;
}

const yes = isOptional(?i32);  // true
const no = isOptional(i32);    // false
```

## When to use comptime

- **Generics**: Functions that work with multiple types
- **Compile-time assertions**: `comptime { assert(condition); }`
- **Code generation**: Build types or lookup tables at compile time
- **Zero-cost abstractions**: Code that looks abstract but compiles to efficient machine code

## comptime vs runtime

```zig
// comptime — known at compile time, no runtime cost
const SIZE = 1024;
var buf: [SIZE]u8 = undefined;

// runtime — computed at runtime
var size: usize = getUserInput();
const buf2 = try allocator.alloc(u8, size);  // Must use allocator for runtime sizes
```

Key insight: Array sizes must be comptime-known. If the size is runtime, you need a slice + allocator.
