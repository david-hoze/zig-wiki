# Control Flow

## if / else

```zig
if (condition) {
    // ...
} else if (other_condition) {
    // ...
} else {
    // ...
}
```

`if` as an expression (like ternary):

```zig
const value = if (x > 0) x else -x;
```

### `if` with optional capture

```zig
if (maybe_value) |value| {
    // value is the unwrapped non-null value
} else {
    // maybe_value was null
}
```

### `if` with error capture

```zig
if (riskyFunction()) |result| {
    // success — result is the return value
} else |err| {
    // error — err is the error value
}
```

## while

```zig
var i: usize = 0;
while (i < 10) {
    std.debug.print("{d}\n", .{i});
    i += 1;
}

// With continue expression (like C's for-loop increment)
var j: usize = 0;
while (j < 10) : (j += 1) {
    std.debug.print("{d}\n", .{j});
}

// With optional capture (loop until null)
while (iterator.next()) |item| {
    processItem(item);
}
```

## for

`for` loops iterate over slices and ranges:

```zig
const items = [_]i32{ 10, 20, 30 };

// Iterate with value
for (items) |item| {
    std.debug.print("{d}\n", .{item});
}

// Iterate with index
for (items, 0..) |item, i| {
    std.debug.print("[{d}] = {d}\n", .{i, item});
}

// Iterate over a range
for (0..10) |i| {
    std.debug.print("{d}\n", .{i});
}
```

## switch

Zig's switch is exhaustive — you must handle every case (or use `else`):

```zig
const x: u8 = 42;
switch (x) {
    0 => std.debug.print("zero\n", .{}),
    1...9 => std.debug.print("single digit\n", .{}),
    10, 20, 30 => std.debug.print("round number\n", .{}),
    else => std.debug.print("other: {d}\n", .{x}),
}
```

Switch on enums (must be exhaustive, no `else` needed):

```zig
const method = request.head.method;
switch (method) {
    .GET => handleGet(),
    .POST => handlePost(),
    .OPTIONS => handleOptions(),
    else => handleNotFound(),
}
```

Switch as an expression:

```zig
const name = switch (color) {
    .red => "red",
    .green => "green",
    .blue => "blue",
};
```

## defer

Runs a statement when the current scope exits:

```zig
{
    const file = try openFile("data.txt");
    defer file.close();  // Runs when this scope exits

    // Use file...
    // Even if an error happens, file.close() still runs
}
```

Multiple defers run in **reverse order** (LIFO):

```zig
defer std.debug.print("1\n", .{});
defer std.debug.print("2\n", .{});
defer std.debug.print("3\n", .{});
// Prints: 3, 2, 1
```

## break and continue

```zig
for (items) |item| {
    if (item == 0) continue;  // Skip this iteration
    if (item < 0) break;      // Exit the loop
}
```

Labeled blocks and break with values:

```zig
const result = blk: {
    if (condition) break :blk 42;
    break :blk 0;
};
```
