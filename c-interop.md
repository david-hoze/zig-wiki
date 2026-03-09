# C Interop

Zig can directly call C functions without wrappers, FFI, or bindings generators. This is one of Zig's killer features.

## @cImport — importing C headers

```zig
const c = @cImport({
    @cInclude("sqlite3.h");
});

// Now use C functions as: c.sqlite3_open, c.sqlite3_exec, etc.
```

The compiler reads the C header and generates Zig type bindings automatically.

## Setting up in build.zig

```zig
// Tell Zig to compile the C source file
exe.addCSourceFile(.{
    .file = b.path("deps/sqlite3.c"),
    .flags = &.{
        "-DSQLITE_THREADSAFE=1",   // C preprocessor defines
        "-DSQLITE_ENABLE_WAL=1",
    },
});

// Tell Zig where to find the header
exe.addIncludePath(b.path("deps"));

// Link libc (needed by most C code)
exe.linkLibC();
```

## Calling C functions

```zig
const c = @cImport({
    @cInclude("sqlite3.h");
});

// C: int sqlite3_open(const char *filename, sqlite3 **ppDb);
// Zig: fn sqlite3_open([*c]const u8, *?*c.sqlite3) c_int

var db: ?*c.sqlite3 = null;
const rc = c.sqlite3_open("test.db", &db);
if (rc != c.SQLITE_OK) {
    // handle error
}
```

## Pointer type mapping

| C type | Zig type | Meaning |
|--------|----------|---------|
| `char*` | `[*c]u8` | Nullable many-item C pointer |
| `const char*` | `[*c]const u8` | Nullable const C pointer |
| `void*` | `?*anyopaque` | Opaque pointer (like void*) |
| `int` | `c_int` | C int (platform-dependent size) |
| `size_t` | `usize` | Pointer-sized unsigned |
| `NULL` | `null` | Null pointer |

## Converting between Zig and C types

### Zig slice to C pointer

C functions expect raw pointers. Zig slices are pointer + length.

```zig
const zig_string: []const u8 = "hello";

// Get the raw pointer for C
const c_ptr: [*c]const u8 = @ptrCast(zig_string.ptr);
const c_len: c_int = @intCast(zig_string.len);

// For null-terminated C strings, use a sentinel-terminated slice:
const c_string: [*:0]const u8 = "hello";  // String literals are null-terminated
c.some_c_function(c_string);
```

### C pointer to Zig slice

```zig
const c_ptr = c.sqlite3_column_text(stmt, 0);  // [*c]const u8
if (c_ptr == null) return null;

const len = c.sqlite3_column_bytes(stmt, 0);
const zig_ptr: [*]const u8 = @ptrCast(c_ptr);
const zig_slice: []const u8 = zig_ptr[0..@intCast(len)];
```

## @intCast and @ptrCast

These are explicit type conversions. Zig never converts implicitly.

```zig
// Integer conversion
const zig_len: usize = 42;
const c_len: c_int = @intCast(zig_len);  // usize → c_int

// Pointer conversion
const c_ptr: [*c]const u8 = @ptrCast(zig_slice.ptr);  // [*]const u8 → [*c]const u8
```

## C memory management

C functions that allocate memory (malloc) must be freed with the **C allocator** (free), not Zig's allocator:

```zig
// SQLite allocates error message strings
var err_msg: [*c]u8 = null;
const rc = c.sqlite3_exec(db, sql, null, null, &err_msg);
if (err_msg) |msg| {
    std.log.err("Error: {s}", .{msg});
    c.sqlite3_free(msg);  // Free with SQLite's allocator, NOT Zig's
}
```

## C constants

C `#define` constants are available in the `c` namespace:

```zig
if (rc != c.SQLITE_OK) { ... }
if (rc == c.SQLITE_ROW) { ... }
if (rc == c.SQLITE_DONE) { ... }
```

## SQLITE_TRANSIENT

A special case — this is a C macro that casts -1 to a function pointer. Zig imports it:

```zig
c.sqlite3_bind_text(stmt, 1, ptr, len, c.SQLITE_TRANSIENT);
// SQLITE_TRANSIENT tells SQLite to copy the data immediately
```

## Complete example: SQLite wrapper

See [SQLite Integration](sqlite-integration.md) for a full working example.
