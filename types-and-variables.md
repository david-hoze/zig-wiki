# Types and Variables

## Variable declarations

```zig
// Mutable variable — can be reassigned
var x: i32 = 42;
x = 100;  // OK

// Immutable binding — cannot be reassigned
const y: i32 = 42;
// y = 100;  // COMPILE ERROR

// Type inference — Zig figures out the type
const z = 42;  // inferred as comptime_int
var w: i32 = 42;  // explicit type needed for var
```

**Rule of thumb**: Use `const` by default. Only use `var` when you need to mutate.

## Integer types

Zig has fixed-width integers with explicit signedness:

```zig
const a: u8 = 255;      // Unsigned 8-bit (0 to 255)
const b: i8 = -128;     // Signed 8-bit (-128 to 127)
const c: u16 = 65535;   // Unsigned 16-bit
const d: i32 = -42;     // Signed 32-bit
const e: u64 = 0;       // Unsigned 64-bit
const f: i64 = 0;       // Signed 64-bit
const g: usize = 0;     // Pointer-sized unsigned (like size_t in C)

// You can even have arbitrary-width integers
const h: u7 = 127;      // 7-bit unsigned!
const i: u1 = 1;        // 1-bit unsigned (0 or 1)
```

No implicit conversions between integer types:

```zig
const a: u8 = 42;
// const b: u16 = a;  // COMPILE ERROR — must be explicit
const b: u16 = @intCast(a);  // OK — explicit widening
```

## Floating point

```zig
const a: f32 = 3.14;    // 32-bit float
const b: f64 = 3.14;    // 64-bit float (default for literals)
```

## Booleans

```zig
const t: bool = true;
const f: bool = false;

// No truthy/falsy — only bool works in conditions
// if (1) {}  // COMPILE ERROR
if (t) {}     // OK
```

## Strings

Strings are just byte slices — `[]const u8`:

```zig
const greeting: []const u8 = "hello";
const name = "world";  // inferred as *const [5:0]u8, coerces to []const u8

// String length
const len = greeting.len;  // 5

// Individual bytes
const first_byte = greeting[0];  // 'h' (u8 value 104)

// Multi-line strings (backslash-continued)
const multi =
    \\Line one
    \\Line two
    \\Line three
;
```

See [Strings](strings.md) for more.

## Arrays

Fixed-size, stack-allocated:

```zig
// Array of 5 i32s
const arr = [5]i32{ 1, 2, 3, 4, 5 };

// Array with inferred length
const arr2 = [_]i32{ 1, 2, 3 };  // [3]i32

// Initialize all to same value
const zeros = [_]u8{0} ** 100;  // 100 zeros

// Access
const first = arr[0];
const length = arr.len;  // 5, known at comptime

// Undefined (uninitialized) — useful for buffers
var buf: [1024]u8 = undefined;
```

## Slices

A pointer + length pair. Like arrays but the length is runtime-known:

```zig
const arr = [5]i32{ 10, 20, 30, 40, 50 };

// Slice from an array (no copy, just a view)
const slice: []const i32 = arr[1..4];  // {20, 30, 40}
const slice_len = slice.len;  // 3

// Slices don't own memory — they point into something else
```

See [Pointers](pointers.md) for the full pointer/slice story.

## Enums

```zig
const Color = enum {
    red,
    green,
    blue,
};

const c = Color.red;

// Switch on enums (must be exhaustive)
switch (c) {
    .red => std.debug.print("Red\n", .{}),
    .green => std.debug.print("Green\n", .{}),
    .blue => std.debug.print("Blue\n", .{}),
}
```

## Structs

```zig
const Point = struct {
    x: f64,
    y: f64,
};

const p = Point{ .x = 1.0, .y = 2.0 };
std.debug.print("({d}, {d})\n", .{ p.x, p.y });
```

See [Structs and Methods](structs-and-methods.md).

## Undefined vs Null

These are different things in Zig:

```zig
// undefined — uninitialized memory. Using it is undefined behavior.
// Used for buffers you'll fill later.
var buf: [100]u8 = undefined;

// null — a valid value for optional types
var maybe: ?i32 = null;
maybe = 42;
```

See [Optionals](optionals.md).
