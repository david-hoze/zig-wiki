# Error Handling

Zig uses **error unions** instead of exceptions or error codes. Every function that can fail has it baked into the return type.

## Error unions (`!T`)

A function returning `!void` can either succeed (void) or fail (error):

```zig
fn readFile(path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    return try file.readToEndAlloc(allocator, 1024 * 1024);
}
```

The `!` before the return type means "this can fail." The full type is `ErrorSet!ReturnType`.

## `try` — propagate errors

`try` is shorthand for "if this fails, return the error from my function":

```zig
// These are equivalent:
const file = try openFile(path);

const file = openFile(path) catch |err| return err;
```

`try` is like Go's `if err != nil { return err }` but in one keyword.

## `catch` — handle errors

```zig
// Provide a default value on error
const value = parseNumber(input) catch 0;

// Handle the error explicitly
const value = parseNumber(input) catch |err| {
    std.log.err("Parse failed: {}", .{err});
    return;
};

// catch unreachable — assert it can't fail (crashes if it does)
const value = parseNumber("42") catch unreachable;
```

## Custom error sets

```zig
const FileError = error{
    NotFound,
    PermissionDenied,
    DiskFull,
};

fn openConfig() FileError!Config {
    // ...
    return error.NotFound;
}
```

## Switching on errors

```zig
const result = doSomething() catch |err| switch (err) {
    error.NotFound => {
        std.log.warn("Not found, using default", .{});
        return default_value;
    },
    error.PermissionDenied => return err,  // re-throw
    else => return err,
};
```

## `errdefer` — cleanup on error only

`defer` always runs. `errdefer` only runs if the function returns an error:

```zig
fn createResource(allocator: std.mem.Allocator) !*Resource {
    const res = try allocator.create(Resource);
    errdefer allocator.destroy(res);  // Only frees if we error below

    try res.init();  // If this fails, errdefer cleans up
    return res;      // If we get here, caller owns res
}
```

## `!void` in main

If `main` returns an error, Zig prints an error trace:

```zig
pub fn main() !void {
    try riskyOperation();
    // If riskyOperation fails, Zig prints the error + stack trace
}
```

## Comparison with other languages

| Pattern | Go | JavaScript | Zig |
|---------|-----|-----------|-----|
| Can fail | `func f() error` | `async function f()` | `fn f() !void` |
| Propagate | `if err != nil { return err }` | `await f()` (throws) | `try f()` |
| Handle | `if err != nil { ... }` | `try { } catch { }` | `f() catch \|err\| { }` |
| Default | N/A | `f().catch(() => default)` | `f() catch default` |
| Cleanup | `defer` | `finally` | `defer` / `errdefer` |
