# Gotchas and Pitfalls

Hard-won lessons from building real Zig projects. Save yourself time by reading these first.

## Zig version differences

The Zig stdlib API changes between versions. **Always check your Zig version** when example code doesn't compile.

### `std.http.Server.Request.respond` (0.14)

```zig
// Zig 0.14: content FIRST, options SECOND
try request.respond(body, .{
    .status = .ok,
    .extra_headers = &.{ ... },
});

// Older Zig: options first, body second (REVERSED)
// try request.respond(.{ .status = .ok, ... }, body);
```

If you see `error: type '[]const u8' does not support struct initialization syntax`, the argument order is wrong.

### `std.http.Server.Request.reader` (0.14)

```zig
// Zig 0.14: reader() returns an error union
const reader = try request.reader();

// Older Zig: reader() returns directly
// const reader = request.reader();
```

### `build.zig.zon` format (0.14)

```zig
// Zig 0.14: .name is an enum literal, requires .fingerprint
.name = .myapp,
.fingerprint = 0x...,

// Older Zig: .name was a string
// .name = "myapp",
```

## C interop gotchas

### sqlite3_exec error message type

```zig
// CORRECT — matches C's `char**`
var err_msg: [*c]u8 = null;

// WRONG — type mismatch
// var err_msg: ?[*:0]u8 = null;
```

### C string null termination

Zig slices are NOT null-terminated. C strings ARE.

```zig
const zig_str: []const u8 = "hello";  // No guaranteed null terminator

// To pass to C:
const c_str: [*:0]const u8 = "hello";  // String literals ARE null-terminated
// Or allocate a null-terminated copy:
const c_str2 = try allocator.dupeZ(u8, zig_str);
defer allocator.free(c_str2);
```

### @ptrCast and @intCast

Zig never converts types implicitly. Common conversions:

```zig
// usize to c_int
const c_len: c_int = @intCast(slice.len);

// Zig pointer to C pointer
const c_ptr: [*c]const u8 = @ptrCast(slice.ptr);

// i64 to usize (only safe if non-negative)
const idx: usize = @intCast(some_i64);
```

## String comparison

```zig
// WRONG — compares pointers, not contents
if (a == b) { }

// CORRECT
if (std.mem.eql(u8, a, b)) { }
```

## `undefined` vs `null`

```zig
var buf: [100]u8 = undefined;   // Uninitialized memory — using it is UB
var maybe: ?i32 = null;          // Optional with no value — safe to check
```

**Never read from `undefined`** memory. It's for buffers you'll fill before reading.

## Forgetting `defer`

```zig
// MEMORY LEAK — no free
const buf = try allocator.alloc(u8, 1024);
// ... use buf ...
// Oops, forgot to free

// CORRECT — defer immediately after alloc
const buf2 = try allocator.alloc(u8, 1024);
defer allocator.free(buf2);
```

## Pointer lifetime with SQLite

```zig
var stmt = try db.prepare("SELECT name FROM users");
defer stmt.deinit();

if (try stmt.step()) {
    const name = stmt.columnText(0);
    // name is valid HERE

    // After another step() or deinit(), name is INVALID
    _ = try stmt.step();
    // name is now dangling! Don't use it.
}
```

Copy the value if you need it to live longer:

```zig
if (try stmt.step()) {
    const name = stmt.columnText(0) orelse "";
    const name_copy = try allocator.dupe(u8, name);
    defer allocator.free(name_copy);
    // name_copy is safe to use after step()/deinit()
}
```

## Exhaustive switch

Zig requires switch to handle ALL cases:

```zig
// COMPILE ERROR — missing cases
switch (method) {
    .GET => handleGet(),
    .POST => handlePost(),
}

// CORRECT — use `else` for remaining cases
switch (method) {
    .GET => handleGet(),
    .POST => handlePost(),
    else => handleNotFound(),
}
```

## Format string type safety

```zig
// COMPILE ERROR — {d} can't format a string
std.debug.print("{d}\n", .{"hello"});

// {s} for strings, {d} for numbers, {} for default
std.debug.print("{s}\n", .{"hello"});   // string
std.debug.print("{d}\n", .{42});        // number
std.debug.print("{}\n", .{some_enum});  // default
```

## `var` requires explicit type

```zig
// COMPILE ERROR — type can't be inferred for var
// var x = 42;

// CORRECT
var x: i32 = 42;

// const CAN infer types
const y = 42;  // OK — comptime_int
```

## Debugging tips

1. **Use `std.log` liberally** — trace execution flow
2. **Check the exact compiler error** — Zig errors are very specific
3. **If it's a type error**, the error message tells you both the expected and actual types
4. **If C interop fails**, compare the Zig-generated C binding with the C header
5. **Build with debug** (default) — you get stack traces on errors
6. **Use `@compileError`** to debug comptime code: `@compileError("got here")`
