# Pointers

Zig has several pointer types, each with specific semantics. This is confusing at first but makes code safer.

## The four pointer types

| Type | What it is | Nullable? | Length known? | Example use |
|------|-----------|-----------|---------------|-------------|
| `*T` | Single-item pointer | No | N/A (one item) | `*Database` |
| `[*]T` | Many-item pointer | No | No | Raw buffer pointer |
| `[]T` | Slice | No | Yes (ptr + len) | `[]const u8` (strings) |
| `[*c]T` | C pointer | Yes | No | C interop |

## Single-item pointer (`*T`)

Points to exactly one value. Cannot be null.

```zig
var x: i32 = 42;
const ptr: *i32 = &x;  // Take address of x
ptr.* = 100;            // Dereference and modify
std.debug.print("{d}\n", .{ptr.*});  // 100
```

Used for: function parameters that modify the caller's value.

```zig
fn increment(value: *i32) void {
    value.* += 1;
}

var x: i32 = 0;
increment(&x);  // x is now 1
```

## Slice (`[]T`)

A pointer + length pair. The most common pointer type in Zig.

```zig
const arr = [_]i32{ 10, 20, 30, 40, 50 };
const slice: []const i32 = arr[1..4];  // {20, 30, 40}

slice.len  // 3
slice.ptr  // pointer to the second element
slice[0]   // 20
```

Strings are slices: `[]const u8`.

### Mutable vs const slices

```zig
[]const u8    // Read-only slice (like string view)
[]u8          // Mutable slice (can modify contents)
```

## Many-item pointer (`[*]T`)

A raw pointer with no length info. Rare in Zig code — mostly for C interop.

```zig
const arr = [_]i32{ 1, 2, 3 };
const ptr: [*]const i32 = &arr;
const first = ptr[0];   // 1
const second = ptr[1];  // 2
// No bounds checking — you must know the length yourself
```

## C pointer (`[*c]T`)

Nullable pointer for C interop. Can be `null`.

```zig
var err_msg: [*c]u8 = null;  // Nullable C pointer

// Check for null before using
if (err_msg) |msg| {
    // msg is [*]u8 (non-null guaranteed inside the if)
    std.debug.print("{s}\n", .{msg});
}
```

## Sentinel-terminated pointers (`[*:0]const u8`)

A pointer with a sentinel value (like C's null-terminated strings):

```zig
const c_string: [*:0]const u8 = "hello";  // Null-terminated
// Zig string literals are secretly null-terminated, so this works

// Convert to slice:
const slice: []const u8 = std.mem.span(c_string);  // "hello", len=5
```

## Pointer arithmetic

Zig doesn't allow pointer arithmetic on `*T`. Use slices instead:

```zig
// WRONG — no pointer arithmetic
// ptr += 1;

// CORRECT — use slice indexing
const slice = arr[1..];  // Skip first element
```

## Converting between pointer types

```zig
// Slice to raw pointer
const slice: []const u8 = "hello";
const ptr: [*]const u8 = slice.ptr;

// Raw pointer to C pointer
const c_ptr: [*c]const u8 = @ptrCast(ptr);

// Array to slice
const arr = [_]i32{ 1, 2, 3 };
const slice2: []const i32 = &arr;

// Pointer to single item to many-item pointer
var x: i32 = 42;
const single: *i32 = &x;
const many: [*]i32 = @ptrCast(single);
_ = many;
```

## Common patterns

### Passing to functions

```zig
fn processItems(items: []const Item) void { ... }  // Read-only
fn modifyItems(items: []Item) void { ... }          // Mutable
fn modifyOne(item: *Item) void { ... }              // Single mutable item
```

### Returning slices

```zig
fn getItems(allocator: std.mem.Allocator) ![]Item {
    const items = try allocator.alloc(Item, 10);
    // Caller must free with allocator.free(items)
    return items;
}
```
