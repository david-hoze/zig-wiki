# HTTP Server

Zig's standard library includes `std.http.Server` — a built-in HTTP server. It's low-level: you handle connections, parse requests, and write responses yourself.

## Minimal server

```zig
const std = @import("std");

pub fn main() !void {
    const address = std.net.Address.parseIp("0.0.0.0", 8080) catch unreachable;

    var server = try address.listen(.{
        .reuse_address = true,
    });
    defer server.deinit();

    std.debug.print("Listening on :8080\n", .{});

    while (true) {
        const connection = try server.accept();
        handleConnection(connection) catch |err| {
            std.log.err("Error: {}", .{err});
        };
    }
}

fn handleConnection(connection: std.net.Server.Connection) !void {
    defer connection.stream.close();

    var buf: [8192]u8 = undefined;
    var http = std.http.Server.init(connection, &buf);

    var request = try http.receiveHead();

    const path = request.head.target;
    const method = request.head.method;

    std.log.info("{s} {s}", .{ @tagName(method), path });

    try request.respond("Hello, World!", .{
        .status = .ok,
        .extra_headers = &.{
            .{ .name = "Content-Type", .value = "text/plain" },
        },
    });
}
```

## Request handling

### Reading the request

```zig
var request = try http.receiveHead();

// Method
const method = request.head.method;  // .GET, .POST, .OPTIONS, etc.

// Path
const path = request.head.target;  // "/api/users?id=42"

// Headers
var it = request.iterateHeaders();
while (it.next()) |header| {
    std.debug.print("{s}: {s}\n", .{ header.name, header.value });
}
```

### Reading the body (Zig 0.14)

```zig
fn readBody(allocator: std.mem.Allocator, request: *std.http.Server.Request) !?[]u8 {
    const content_length = request.head.content_length orelse return null;
    if (content_length > 1_048_576) return error.BodyTooLarge;  // 1MB limit

    const buf = try allocator.alloc(u8, @intCast(content_length));
    errdefer allocator.free(buf);

    const reader = try request.reader();  // Note: `try` needed in 0.14
    var total: usize = 0;
    while (total < buf.len) {
        const n = try reader.read(buf[total..]);
        if (n == 0) break;
        total += n;
    }
    return buf[0..total];
}
```

### Sending a response (Zig 0.14)

```zig
// Zig 0.14: respond(content, options) — content FIRST, options SECOND
try request.respond(body, .{
    .status = .ok,
    .extra_headers = &.{
        .{ .name = "Content-Type", .value = "application/json" },
    },
});
```

**Version note**: In older Zig versions, the argument order was reversed (options first, body second). If you see `error: type '[]const u8' does not support struct initialization syntax`, you have the arguments in the wrong order.

## Routing

Zig has no built-in router. Use `std.mem.eql` and `std.mem.startsWith`:

```zig
const path = request.head.target;
const method = request.head.method;

if (method == .GET and std.mem.eql(u8, path, "/api/stats")) {
    try handleStats(allocator, &request);
} else if (method == .GET and std.mem.startsWith(u8, path, "/api/users/")) {
    const id = path["/api/users/".len..];  // Extract path parameter
    try handleGetUser(allocator, &request, id);
} else if (method == .POST and std.mem.eql(u8, path, "/api/users")) {
    try handleCreateUser(allocator, &request, body);
} else if (method == .OPTIONS) {
    try handleCors(&request);
} else {
    try request.respond("{\"error\":\"not found\"}", .{ .status = .not_found });
}
```

## CORS

```zig
fn handleCors(request: *std.http.Server.Request) !void {
    try request.respond("", .{
        .status = .no_content,
        .extra_headers = &.{
            .{ .name = "Access-Control-Allow-Origin", .value = "*" },
            .{ .name = "Access-Control-Allow-Headers", .value = "Authorization, Content-Type" },
            .{ .name = "Access-Control-Allow-Methods", .value = "GET, POST, OPTIONS" },
            .{ .name = "Access-Control-Max-Age", .value = "86400" },
        },
    });
}
```

## JSON responses

```zig
fn handleStats(allocator: std.mem.Allocator, request: *std.http.Server.Request) !void {
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    try std.fmt.format(buf.writer(),
        \\{{"users":{d},"active":{d}}}
    , .{ user_count, active_count });

    try request.respond(buf.items, .{
        .status = .ok,
        .extra_headers = &.{
            .{ .name = "Content-Type", .value = "application/json" },
        },
    });
}
```

## POST endpoint with JSON parsing

A complete pattern: parse JSON body, validate, write to DB, return JSON:

```zig
const RegisterRequest = struct {
    email: []const u8,
};

fn handleRegister(
    allocator: std.mem.Allocator,
    database: *Database,
    request: *std.http.Server.Request,
    body: ?[]const u8,
) !void {
    const request_body = body orelse {
        try sendError(request, .bad_request, "missing request body");
        return;
    };

    // Parse JSON into struct — catch turns parse error into a handled response
    const parsed = std.json.parseFromSlice(RegisterRequest, allocator, request_body, .{
        .ignore_unknown_fields = true,
    }) catch {
        try sendError(request, .bad_request, "invalid JSON");
        return;
    };
    defer parsed.deinit();
    const req = parsed.value;

    // Validate
    if (req.email.len == 0) {
        try sendError(request, .bad_request, "email is required");
        return;
    }

    // Use the parsed data (e.g., insert into DB)
    var stmt = try database.prepare("INSERT INTO users (email) VALUES (?1)");
    defer stmt.deinit();
    try stmt.bindText(1, req.email);
    _ = stmt.step() catch {
        try sendError(request, .bad_request, "user already exists");
        return;
    };

    // Build dynamic JSON response
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();
    try std.fmt.format(buf.writer(), \\{{"email":"{s}"}}, .{req.email});
    try sendResponse(request, .ok, buf.items);
}
```

**Key pattern**: `catch` after `parseFromSlice` or `stmt.step()` lets you return a proper HTTP error instead of crashing.

## Extracting headers

```zig
fn getHeader(request: *std.http.Server.Request, name: []const u8) ?[]const u8 {
    var it = request.iterateHeaders();
    while (it.next()) |header| {
        if (std.ascii.eqlIgnoreCase(header.name, name)) {
            return header.value;
        }
    }
    return null;
}

// Usage
const auth = getHeader(&request, "authorization");
if (auth) |token| {
    // Use token
}
```

## Status codes

```zig
.ok                  // 200
.created             // 201
.no_content          // 204
.bad_request         // 400
.unauthorized        // 401
.not_found           // 404
.too_many_requests   // 429
.internal_server_error  // 500
```

## Threading note

The example above is single-threaded (handles one connection at a time). For concurrent connections, you'd use `std.Thread.spawn` per connection. For most internal/low-traffic servers, single-threaded is fine.
