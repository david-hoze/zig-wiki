# Structs and Methods

Zig has no classes, no inheritance, no interfaces. Just structs with functions.

## Basic struct

```zig
const Point = struct {
    x: f64,
    y: f64,
};

const p = Point{ .x = 1.0, .y = 2.0 };
std.debug.print("({d}, {d})\n", .{ p.x, p.y });
```

## Methods

A "method" is just a function inside a struct that takes `self`:

```zig
const Point = struct {
    x: f64,
    y: f64,

    // Method — takes self by value (can't modify)
    pub fn distanceTo(self: Point, other: Point) f64 {
        const dx = self.x - other.x;
        const dy = self.y - other.y;
        return @sqrt(dx * dx + dy * dy);
    }

    // Mutating method — takes self by pointer
    pub fn translate(self: *Point, dx: f64, dy: f64) void {
        self.x += dx;
        self.y += dy;
    }

    // "Static" method — no self parameter
    pub fn origin() Point {
        return Point{ .x = 0, .y = 0 };
    }
};

// Usage
var p = Point{ .x = 3.0, .y = 4.0 };
const dist = p.distanceTo(Point.origin());  // Method call
p.translate(1.0, 1.0);                       // Mutating method
const o = Point.origin();                     // Static method
```

## Self parameter rules

```zig
pub fn read(self: MyStruct) void     // Read-only, by value (copy)
pub fn write(self: *MyStruct) void   // Mutable, by pointer
pub fn create() MyStruct             // No self = "static" / constructor
```

## Default values

```zig
const Config = struct {
    port: u16 = 8080,
    host: []const u8 = "0.0.0.0",
    debug: bool = false,
};

const config = Config{};  // All defaults
const custom = Config{ .port = 3000 };  // Override just port
```

## Nested structs (namespacing)

```zig
const Database = struct {
    db: *c.sqlite3,

    pub const Statement = struct {
        stmt: *c.sqlite3_stmt,

        pub fn step(self: *Statement) !bool {
            // ...
        }
    };

    pub fn prepare(self: *Database, sql: [*:0]const u8) !Statement {
        // ...
    }
};
```

## Structs as namespaces

A file IS a struct in Zig. When you `@import("db.zig")`, you get the struct that the file defines:

```zig
// db.zig — this file IS a struct
const std = @import("std");

pub const Database = struct { ... };

pub fn helperFunction() void { ... }
```

```zig
// main.zig
const db_mod = @import("db.zig");
var database = try db_mod.Database.open("test.db");
```

## Anonymous structs (tuples)

```zig
// Anonymous struct literal — used for format args, etc.
std.debug.print("{d} + {d} = {d}\n", .{ 1, 2, 3 });
//                                    ^^^^^^^^^^^^^ anonymous struct

// The .{} syntax creates an anonymous struct with inferred field types
```

## No inheritance

Zig doesn't have inheritance or interfaces. Instead:
- **Composition**: Embed one struct inside another
- **Generics via comptime**: Pass types as comptime parameters
- **Duck typing via comptime**: Check if a type has certain methods at compile time

```zig
// Composition over inheritance
const HttpServer = struct {
    database: *Database,    // has-a, not is-a
    config: Config,

    pub fn handleRequest(self: *HttpServer, ...) !void {
        // Use self.database, self.config
    }
};
```
