# Collections

## ArrayList — dynamic array

The most common collection. Like `std::vector` in C++ or `[]T` in Go.

```zig
var list = std.ArrayList(i32).init(allocator);
defer list.deinit();  // Frees the underlying buffer

try list.append(1);
try list.append(2);
try list.append(3);

// Access items as a slice
const items = list.items;  // []i32
std.debug.print("Length: {d}\n", .{items.len});  // 3
std.debug.print("First: {d}\n", .{items[0]});    // 1

// Iterate
for (list.items) |item| {
    std.debug.print("{d}\n", .{item});
}

// Remove
_ = list.orderedRemove(0);  // Remove first element, shift others
_ = list.swapRemove(0);      // Remove first, swap with last (faster)
```

### ArrayList(u8) as string builder

```zig
var buf = std.ArrayList(u8).init(allocator);
defer buf.deinit();

try buf.appendSlice("Hello");
try buf.appendSlice(", ");

// Use as a writer for formatted output
try std.fmt.format(buf.writer(), "{s}!", .{"world"});

const result = buf.items;  // "Hello, world!"
```

## HashMap — key-value store

```zig
var map = std.StringHashMap(i32).init(allocator);
defer map.deinit();

try map.put("alice", 30);
try map.put("bob", 25);

// Lookup
if (map.get("alice")) |age| {
    std.debug.print("Alice is {d}\n", .{age});
}

// Check existence
const exists = map.contains("charlie");  // false

// Remove
_ = map.remove("bob");

// Iterate
var it = map.iterator();
while (it.next()) |entry| {
    std.debug.print("{s}: {d}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
}
```

### HashMap with non-string keys

```zig
// AutoHashMap uses automatic hash/equality for any key type
var map = std.AutoHashMap(u64, []const u8).init(allocator);
defer map.deinit();

try map.put(42, "the answer");
```

## ArrayHashMap — insertion-ordered map

Like HashMap but preserves insertion order:

```zig
var map = std.StringArrayHashMap(i32).init(allocator);
defer map.deinit();

try map.put("first", 1);
try map.put("second", 2);
try map.put("third", 3);

// Iteration is in insertion order
var it = map.iterator();
while (it.next()) |entry| {
    // Prints: first, second, third (guaranteed order)
}
```

## BoundedArray — fixed-capacity, no allocation

Like ArrayList but backed by a comptime-known fixed buffer. No heap allocation.

```zig
var arr = std.BoundedArray(u8, 256){};  // Max 256 elements, on the stack

try arr.append('h');
try arr.append('i');

const slice = arr.slice();  // []u8
```

## Sorting

```zig
var items = [_]i32{ 5, 3, 1, 4, 2 };
std.mem.sort(i32, &items, {}, std.sort.asc(i32));
// items is now { 1, 2, 3, 4, 5 }

// Custom comparator
std.mem.sort(Person, people.items, {}, struct {
    fn lessThan(_: void, a: Person, b: Person) bool {
        return std.mem.lessThan(u8, a.name, b.name);
    }
}.lessThan);
```
