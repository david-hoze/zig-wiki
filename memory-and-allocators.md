# Memory and Allocators

This is the most important concept in Zig. There is **no garbage collector**. Every allocation is explicit, and every allocation needs a corresponding free.

## The key idea

In most languages, memory allocation is hidden:

```javascript
// JavaScript — allocates silently, GC cleans up
const arr = [1, 2, 3];  // heap allocation hidden
const str = "hello " + "world";  // another hidden allocation
```

In Zig, **nothing allocates behind your back**:

```zig
// Zig — allocation is explicit
var list = std.ArrayList(u8).init(allocator);  // you see the allocator
defer list.deinit();  // you clean up explicitly
try list.appendSlice("hello");  // you see that this might allocate
```

## Allocators

An allocator is an interface that provides `alloc`, `free`, `resize`. You pass it to any function that needs to allocate memory.

### GeneralPurposeAllocator (GPA)

The standard heap allocator. In debug builds, it detects memory leaks and double-frees.

```zig
pub fn main() !void {
    // Create the allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();  // Reports leaks in debug builds
    const allocator = gpa.allocator();

    // Pass it to things that need memory
    var list = std.ArrayList(u8).init(allocator);
    defer list.deinit();

    try list.appendSlice("hello");
}
```

### Page allocator

Low-level, allocates directly from the OS. No tracking, no overhead. Good for large one-time allocations.

```zig
const allocator = std.heap.page_allocator;
const buf = try allocator.alloc(u8, 4096);
defer allocator.free(buf);
```

### Arena allocator

Allocates from a pool, frees everything at once. Great for request-scoped memory (allocate per request, free all when done).

```zig
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();  // Frees ALL allocations at once
const allocator = arena.allocator();

// Allocate freely — no need to free individual allocations
const a = try allocator.alloc(u8, 100);
const b = try allocator.alloc(u8, 200);
// When arena.deinit() runs, both a and b are freed
_ = a;
_ = b;
```

### Fixed buffer allocator

Allocates from a stack buffer. Zero heap usage. Fails if you exceed the buffer.

```zig
var buf: [1024]u8 = undefined;
var fba = std.heap.FixedBufferAllocator.init(&buf);
const allocator = fba.allocator();

const slice = try allocator.alloc(u8, 100);  // From the stack buffer
_ = slice;
```

## Common patterns

### Allocate + defer free

```zig
const buf = try allocator.alloc(u8, 1024);
defer allocator.free(buf);
// Use buf...
```

### errdefer — free only on error

```zig
const buf = try allocator.alloc(u8, 1024);
errdefer allocator.free(buf);  // Only frees if we return an error
// Do stuff that might fail...
return buf;  // Caller now owns the memory
```

### ArrayList — dynamic array

```zig
var list = std.ArrayList(u8).init(allocator);
defer list.deinit();

try list.append(42);
try list.appendSlice(&[_]u8{ 1, 2, 3 });

const items = list.items;  // []u8 slice
```

## Rules of thumb

1. **Use `const` by default** — if you don't need to mutate, don't
2. **`defer` immediately after allocation** — so you never forget to free
3. **Pass allocators down** — create in main, pass to everything
4. **Prefer stack allocation** — arrays with known sizes go on the stack for free
5. **Use arena allocators** for short-lived, request-scoped work
6. **Every `alloc` needs a `free`** — or a `defer deinit()` on the container
