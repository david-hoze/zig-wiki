# Optionals

Zig has no `null` in the traditional sense. Instead, a type is either **definitely a value** or **explicitly optional** (`?T`).

## Optional types (`?T`)

```zig
var name: ?[]const u8 = null;  // Can be null
name = "Alice";                 // Can be a value

var count: i32 = 42;           // Cannot be null — ever
// count = null;               // COMPILE ERROR
```

## Unwrapping optionals

### `orelse` — provide a default

```zig
const name: ?[]const u8 = null;
const display_name = name orelse "Anonymous";  // "Anonymous"
```

### `if` with capture — check and use

```zig
const maybe_name: ?[]const u8 = getName();

if (maybe_name) |name| {
    // name is []const u8 here (not optional)
    std.debug.print("Hello, {s}\n", .{name});
} else {
    std.debug.print("No name\n", .{});
}
```

### `.?` — force unwrap (crashes on null)

```zig
const name: ?[]const u8 = "Alice";
const value = name.?;  // "Alice"

const empty: ?[]const u8 = null;
// const bad = empty.?;  // RUNTIME CRASH — reached unreachable
```

Only use `.?` when you've already verified it's not null.

## Optionals from C code

C functions that return pointers often return null on failure. Zig's C interop represents these as optionals:

```zig
// C: sqlite3* db = NULL; sqlite3_open(path, &db);
// In Zig, this becomes:
var db: ?*c.sqlite3 = null;  // Optional pointer — can be null
const rc = c.sqlite3_open(path, &db);
if (rc != c.SQLITE_OK) {
    if (db) |d| _ = c.sqlite3_close(d);  // If non-null, close it
    return error.OpenFailed;
}
// After checking rc, we know db is non-null:
const database = db.?;  // Safe to unwrap
```

## Optional pointers vs nullable C pointers

```zig
?*T      // Zig optional pointer — either valid or null, checked at compile time
[*c]T    // C pointer — can be null, Zig doesn't enforce checks (C compat)
```

## Chaining optionals

```zig
const result = getUser() orelse return error.NoUser;
const email = result.email orelse return error.NoEmail;
```

## Why this is better than null

In JavaScript/Java/Go, any pointer/reference can be null. You only find out at runtime. In Zig:
- If a type is `T`, it **cannot** be null. Period.
- If a type is `?T`, the compiler **forces** you to handle the null case.
- No null pointer exceptions, ever.
