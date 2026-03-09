# JSON

Zig's standard library has built-in JSON parsing and serialization via `std.json`.

## Parsing into a struct

Define a struct matching the JSON shape, then parse:

```zig
const Request = struct {
    name: []const u8,
    age: i64,
    active: bool,
};

const json_str = '{"name":"Alice","age":30,"active":true}';

const parsed = try std.json.parseFromSlice(
    Request,
    allocator,
    json_str,
    .{ .ignore_unknown_fields = true },  // Don't error on extra keys
);
defer parsed.deinit();  // Frees all memory allocated during parsing

const req = parsed.value;
std.debug.print("Name: {s}, Age: {d}\n", .{ req.name, req.age });
```

**Important**: `parsed.deinit()` frees ALL memory including the string slices. Don't use `req.name` after deinit.

## Parsing options

```zig
.{
    .ignore_unknown_fields = true,  // Skip JSON keys not in the struct
    .allocate = .alloc_always,      // Control allocation behavior
}
```

## Serialization

### Stringify a struct

```zig
const response = Response{
    .status = "ok",
    .count = 42,
};

var buf = std.ArrayList(u8).init(allocator);
defer buf.deinit();

try std.json.stringify(response, .{}, buf.writer());
// buf.items contains: {"status":"ok","count":42}
```

### Manual JSON building

For complex or dynamic JSON, build it manually:

```zig
var buf = std.ArrayList(u8).init(allocator);
defer buf.deinit();

try buf.appendSlice("{\"results\":{");

var first = true;
for (items) |item| {
    if (!first) try buf.appendSlice(",");
    first = false;

    try std.fmt.format(buf.writer(),
        \\"{s}":{{"value":{d}}}
    , .{ item.key, item.value });
}

try buf.appendSlice("}}");
```

## Nested structs

```zig
const Address = struct {
    city: []const u8,
    zip: []const u8,
};

const Person = struct {
    name: []const u8,
    address: Address,
};

// Parses: {"name":"Alice","address":{"city":"NYC","zip":"10001"}}
const parsed = try std.json.parseFromSlice(Person, allocator, json_str, .{});
```

## Arrays in JSON

```zig
const BatchRequest = struct {
    hashes: []const []const u8,  // JSON array of strings
};

// Parses: {"hashes":["abc","def","ghi"]}
const parsed = try std.json.parseFromSlice(BatchRequest, allocator, json_str, .{});
const hashes = parsed.value.hashes;  // slice of string slices
```

## Optional fields

```zig
const Config = struct {
    name: []const u8,
    port: ?u16 = null,       // Optional — defaults to null if missing from JSON
};

// Both valid: {"name":"app"} and {"name":"app","port":8080}
```

## Dynamic JSON (no struct)

When you don't know the shape ahead of time:

```zig
const parsed = try std.json.parseFromSlice(
    std.json.Value,  // Generic JSON value type
    allocator,
    json_str,
    .{},
);
defer parsed.deinit();

const root = parsed.value;
switch (root) {
    .object => |obj| {
        if (obj.get("name")) |name_val| {
            switch (name_val) {
                .string => |s| std.debug.print("Name: {s}\n", .{s}),
                else => {},
            }
        }
    },
    else => {},
}
```

## Common gotcha

If `std.json.stringify` fails on your struct, it's usually because of:
- `?T` optional fields with no default
- Custom types that `std.json` doesn't know how to serialize
- Nested pointers

Fallback: build the JSON manually with `std.fmt` into an `ArrayList(u8)`.
