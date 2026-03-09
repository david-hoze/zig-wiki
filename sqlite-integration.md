# SQLite Integration

A complete guide to using SQLite from Zig via C interop. Based on the real implementation in the Shmirat Eynaim server.

## Setup

### 1. Download SQLite amalgamation

```bash
curl -O https://sqlite.org/2024/sqlite-amalgamation-3450000.zip
unzip sqlite-amalgamation-3450000.zip
cp sqlite-amalgamation-3450000/sqlite3.c deps/
cp sqlite-amalgamation-3450000/sqlite3.h deps/
```

You only need two files: `sqlite3.c` (the entire SQLite library) and `sqlite3.h` (the header).

### 2. Configure build.zig

```zig
exe.addCSourceFile(.{
    .file = b.path("deps/sqlite3.c"),
    .flags = &.{
        "-DSQLITE_THREADSAFE=1",
        "-DSQLITE_ENABLE_WAL=1",
    },
});
exe.addIncludePath(b.path("deps"));
exe.linkLibC();
```

### 3. Import in Zig

```zig
const c = @cImport({
    @cInclude("sqlite3.h");
});
```

## Opening and closing

```zig
var db: ?*c.sqlite3 = null;
const rc = c.sqlite3_open("myapp.db", &db);
if (rc != c.SQLITE_OK) {
    if (db) |d| _ = c.sqlite3_close(d);
    return error.SqliteOpenFailed;
}
defer _ = c.sqlite3_close(db.?);

// Enable WAL mode (better for concurrent reads)
_ = c.sqlite3_exec(db.?, "PRAGMA journal_mode=WAL;", null, null, null);
```

## Executing statements (no results)

For CREATE TABLE, INSERT, UPDATE, DELETE:

```zig
var err_msg: [*c]u8 = null;  // Must be [*c]u8, NOT ?[*:0]u8
const rc = c.sqlite3_exec(db, sql, null, null, &err_msg);
if (rc != c.SQLITE_OK) {
    if (err_msg) |msg| {
        std.log.err("SQLite error: {s}", .{msg});
        c.sqlite3_free(msg);  // Free with SQLite's allocator!
    }
    return error.ExecFailed;
}
```

**Gotcha**: The error message pointer type must be `[*c]u8`, not `?[*:0]u8`. The latter causes a type mismatch in Zig 0.14.

## Prepared statements

For queries with parameters (prevents SQL injection):

### Prepare

```zig
var stmt: ?*c.sqlite3_stmt = null;
const rc = c.sqlite3_prepare_v2(db, "SELECT * FROM users WHERE email = ?1", -1, &stmt, null);
if (rc != c.SQLITE_OK) {
    return error.PrepareFailed;
}
defer _ = c.sqlite3_finalize(stmt.?);
```

### Bind parameters

Parameters are 1-indexed (not 0):

```zig
// Bind text (string)
_ = c.sqlite3_bind_text(stmt, 1, @ptrCast(email.ptr), @intCast(email.len), c.SQLITE_TRANSIENT);

// Bind integer
_ = c.sqlite3_bind_int64(stmt, 2, 42);

// Bind float
_ = c.sqlite3_bind_double(stmt, 3, 0.95);

// Bind blob (binary data)
_ = c.sqlite3_bind_blob(stmt, 4, @ptrCast(data.ptr), @intCast(data.len), c.SQLITE_TRANSIENT);
```

`SQLITE_TRANSIENT` tells SQLite to make its own copy of the data. Without it, you'd need to keep the data alive until the statement finishes.

### Step (execute / fetch rows)

```zig
const step_rc = c.sqlite3_step(stmt);
if (step_rc == c.SQLITE_ROW) {
    // A row is available — read columns
} else if (step_rc == c.SQLITE_DONE) {
    // No more rows (or statement completed for INSERT/UPDATE)
}
```

### Read columns

Columns are 0-indexed:

```zig
// Text column
const text_ptr = c.sqlite3_column_text(stmt, 0);
if (text_ptr != null) {
    const len = c.sqlite3_column_bytes(stmt, 0);
    const zig_ptr: [*]const u8 = @ptrCast(text_ptr);
    const value: []const u8 = zig_ptr[0..@intCast(len)];
    // Use value — valid until next step() or finalize()
}

// Integer column
const int_value = c.sqlite3_column_int64(stmt, 1);

// Float column
const float_value = c.sqlite3_column_double(stmt, 2);

// Blob column
const blob_ptr = c.sqlite3_column_blob(stmt, 3);
if (blob_ptr != null) {
    const blob_len = c.sqlite3_column_bytes(stmt, 3);
    const byte_ptr: [*]const u8 = @ptrCast(blob_ptr);
    const blob: []const u8 = byte_ptr[0..@intCast(blob_len)];
}
```

**Important**: Column values point into SQLite's internal buffer. They're only valid until the next `sqlite3_step()` or `sqlite3_finalize()`. If you need to keep them, copy them.

### Reset and reuse

```zig
_ = c.sqlite3_reset(stmt);          // Reset for re-execution
_ = c.sqlite3_clear_bindings(stmt); // Clear bound parameters
// Now bind new parameters and step() again
```

## Multi-line SQL strings

Use Zig's multi-line string syntax:

```zig
const sql =
    \\CREATE TABLE IF NOT EXISTS users (
    \\    id INTEGER PRIMARY KEY AUTOINCREMENT,
    \\    email TEXT UNIQUE NOT NULL,
    \\    token TEXT UNIQUE NOT NULL,
    \\    approved INTEGER NOT NULL DEFAULT 0,
    \\    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    \\);
;
```

## Wrapping in a Zig struct

See the full `Database` and `Statement` wrappers in the server's `src/db.zig`. The key idea: wrap C functions in Zig methods that return proper Zig error unions and use Zig slices instead of raw C pointers.

```zig
pub const Database = struct {
    db: *c.sqlite3,

    pub fn open(path: [*:0]const u8) !Database { ... }
    pub fn exec(self: *Database, sql: [*:0]const u8) !void { ... }
    pub fn prepare(self: *Database, sql: [*:0]const u8) !Statement { ... }
    pub fn close(self: *Database) void { ... }
};

pub const Statement = struct {
    stmt: *c.sqlite3_stmt,

    pub fn bindText(self: *Statement, col: c_int, value: []const u8) !void { ... }
    pub fn step(self: *Statement) !bool { ... }
    pub fn columnText(self: *Statement, col: c_int) ?[]const u8 { ... }
    pub fn deinit(self: *Statement) void { ... }
};
```

## Common gotchas

1. **Error message type**: Use `[*c]u8` not `?[*:0]u8` for sqlite3_exec's error parameter
2. **Column indices are 0-based**, parameter indices are 1-based
3. **Column values are temporary** — copy if you need to keep them
4. **Always finalize** prepared statements (use `defer stmt.deinit()`)
5. **Free error messages** with `c.sqlite3_free()`, not Zig's allocator
