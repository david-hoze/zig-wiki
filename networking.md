# Networking Basics

## Listening on a port

```zig
const std = @import("std");

const address = std.net.Address.parseIp("0.0.0.0", 8080) catch unreachable;

var server = try address.listen(.{
    .reuse_address = true,  // Allow quick restart without TIME_WAIT
});
defer server.deinit();
```

`0.0.0.0` means all interfaces. Use `127.0.0.1` for localhost only.

## Accepting connections

```zig
while (true) {
    const connection = try server.accept();
    defer connection.stream.close();

    // connection.stream is a std.net.Stream — read/write TCP data
    var buf: [1024]u8 = undefined;
    const n = try connection.stream.read(&buf);
    const data = buf[0..n];
    _ = data;

    try connection.stream.writeAll("HTTP/1.1 200 OK\r\n\r\nHello");
}
```

## Address types

```zig
// Parse an IP address + port
const addr = std.net.Address.parseIp("192.168.1.1", 3000) catch unreachable;

// IPv4
const addr4 = std.net.Address.initIp4(.{ 127, 0, 0, 1 }, 8080);

// From a hostname (DNS resolution)
const list = try std.net.getAddressList(allocator, "example.com", 80);
defer list.deinit();
```

## Reading and writing

```zig
// Read into a buffer
var buf: [4096]u8 = undefined;
const bytes_read = try stream.read(&buf);
const data = buf[0..bytes_read];

// Write all data (retries partial writes)
try stream.writeAll("complete message");

// Write a slice
try stream.writeAll(response_bytes);
```

## Timeouts

Currently, Zig's std.net doesn't have built-in timeouts. For the HTTP server, the `busy_timeout` on SQLite and the connection lifecycle handle this. For production use, consider OS-level socket options.
