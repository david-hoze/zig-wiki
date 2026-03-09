# File I/O

## Reading a file

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Open
    const file = try std.fs.cwd().openFile("data.txt", .{});
    defer file.close();

    // Read entire file into memory
    const contents = try file.readToEndAlloc(allocator, 1024 * 1024);  // 1MB max
    defer allocator.free(contents);

    std.debug.print("File contents: {s}\n", .{contents});
}
```

## Reading line by line

```zig
const file = try std.fs.cwd().openFile("data.txt", .{});
defer file.close();

var buf: [1024]u8 = undefined;
const reader = file.reader();

while (try reader.readUntilDelimiterOrEof(&buf, '\n')) |line| {
    std.debug.print("Line: {s}\n", .{line});
}
```

## Writing a file

```zig
// Create or overwrite
const file = try std.fs.cwd().createFile("output.txt", .{});
defer file.close();

try file.writeAll("Hello, world!\n");

// Formatted writing
const writer = file.writer();
try writer.print("Count: {d}\n", .{42});
```

## Appending to a file

```zig
const file = try std.fs.cwd().openFile("log.txt", .{
    .mode = .write_only,
});
defer file.close();

try file.seekFromEnd(0);  // Seek to end
try file.writeAll("New log entry\n");
```

## Check if file exists

```zig
const stat = std.fs.cwd().statFile("config.json") catch |err| {
    if (err == error.FileNotFound) {
        std.debug.print("File not found\n", .{});
        return;
    }
    return err;
};
std.debug.print("Size: {d} bytes\n", .{stat.size});
```

## Create directories

```zig
try std.fs.cwd().makePath("path/to/dir");  // Creates intermediate dirs
```

## Delete a file

```zig
try std.fs.cwd().deleteFile("temp.txt");
```

## List directory contents

```zig
var dir = try std.fs.cwd().openDir("src", .{ .iterate = true });
defer dir.close();

var it = dir.iterate();
while (try it.next()) |entry| {
    std.debug.print("{s} ({s})\n", .{ entry.name, @tagName(entry.kind) });
}
```

## Absolute paths

```zig
// Get absolute path from relative
var buf: [std.fs.max_path_bytes]u8 = undefined;
const abs_path = try std.fs.cwd().realpath("src/main.zig", &buf);
```
