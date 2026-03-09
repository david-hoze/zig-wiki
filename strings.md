# Strings

Zig has no string type. Strings are just `[]const u8` — a byte slice.

## The basics

```zig
const greeting = "hello";  // Type: *const [5:0]u8 (pointer to 5 bytes + null terminator)
// Coerces to []const u8 (slice) in most contexts

const len = greeting.len;      // 5
const first = greeting[0];     // 'h' (u8 = 104)
```

## String comparison

**Never use `==` to compare strings.** It compares pointers, not contents.

```zig
const a = "hello";
const b = "hello";

// WRONG — compares pointers
// if (a == b) {}

// CORRECT — compares contents
if (std.mem.eql(u8, a, b)) {
    std.debug.print("equal\n", .{});
}

// Case-insensitive comparison
if (std.ascii.eqlIgnoreCase(a, "HELLO")) {
    std.debug.print("equal (case-insensitive)\n", .{});
}
```

## String operations

```zig
// Starts with / ends with
std.mem.startsWith(u8, path, "/api/")
std.mem.endsWith(u8, filename, ".zig")

// Find a substring
const idx = std.mem.indexOf(u8, haystack, "needle");  // ?usize

// Split
var it = std.mem.splitScalar(u8, "a,b,c", ',');
while (it.next()) |part| {
    std.debug.print("{s}\n", .{part});
}

// Trim
const trimmed = std.mem.trim(u8, "  hello  ", " ");  // "hello"
```

## Formatting strings

```zig
// Into a stack buffer (no allocation)
var buf: [256]u8 = undefined;
const result = std.fmt.bufPrint(&buf, "Hello, {s}! You are {d}.", .{"Alice", 30}) catch "";

// Into an allocated buffer
const result2 = try std.fmt.allocPrint(allocator, "Count: {d}", .{42});
defer allocator.free(result2);
```

## Format specifiers

```
{s}     — string ([]const u8)
{d}     — decimal integer
{d:.2}  — float with 2 decimal places
{x}     — hex lowercase
{X}     — hex uppercase
{c}     — single character (u8)
{b}     — binary
{}      — default format for the type
{any}   — debug format for any type
```

## Multi-line strings

```zig
const sql =
    \\SELECT *
    \\FROM users
    \\WHERE approved = 1
;
// Each line starts with \\, no leading whitespace issues
```

## Building strings dynamically

```zig
// ArrayList(u8) as a string builder
var buf = std.ArrayList(u8).init(allocator);
defer buf.deinit();

try buf.appendSlice("Hello");
try buf.appendSlice(", ");
try buf.appendSlice("world");
try buf.append('!');

const result = buf.items;  // "Hello, world!"
```

## Null-terminated strings for C

Zig strings (`[]const u8`) have a length and are NOT null-terminated.
C strings (`[*:0]const u8`) ARE null-terminated.

```zig
// Zig string literal is actually null-terminated behind the scenes:
const s = "hello";  // *const [5:0]u8 — 5 bytes + null sentinel

// Convert a slice to null-terminated for C functions:
const path: []const u8 = getSomePath();
const c_path = try allocator.dupeZ(u8, path);  // Adds null terminator
defer allocator.free(c_path);
c.sqlite3_open(c_path, &db);

// Or use a buffer:
var buf: [256:0]u8 = undefined;  // Sentinel-terminated buffer
@memcpy(buf[0..path.len], path);
buf[path.len] = 0;
```

## String slicing

```zig
const s = "Hello, world!";
const hello = s[0..5];     // "Hello"
const world = s[7..12];    // "world"
const rest = s[7..];       // "world!"
```

Slicing is O(1) — no copy, just pointer + length adjustment.
