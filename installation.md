# Installation

Zig has **no installer, no dependencies, no runtime**. It's a single directory with a binary.

## Download

Go to [ziglang.org/download](https://ziglang.org/download/) and grab the latest stable release.

### Linux x86_64

```bash
curl -L https://ziglang.org/download/0.14.0/zig-linux-x86_64-0.14.0.tar.xz | tar -xJ
sudo mv zig-linux-x86_64-0.14.0 /opt/zig
sudo ln -s /opt/zig/zig /usr/local/bin/zig
```

### Linux ARM (Raspberry Pi, etc.)

```bash
curl -L https://ziglang.org/download/0.14.0/zig-linux-aarch64-0.14.0.tar.xz | tar -xJ
sudo mv zig-linux-aarch64-0.14.0 /opt/zig
sudo ln -s /opt/zig/zig /usr/local/bin/zig
```

### macOS (Apple Silicon)

```bash
curl -L https://ziglang.org/download/0.14.0/zig-macos-aarch64-0.14.0.tar.xz | tar -xJ
sudo mv zig-macos-aarch64-0.14.0 /opt/zig
sudo ln -s /opt/zig/zig /usr/local/bin/zig
```

### macOS (Intel)

```bash
curl -L https://ziglang.org/download/0.14.0/zig-macos-x86_64-0.14.0.tar.xz | tar -xJ
sudo mv zig-macos-x86_64-0.14.0 /opt/zig
sudo ln -s /opt/zig/zig /usr/local/bin/zig
```

### Windows

Download the zip from the website, extract it, and add to PATH. Or via curl in Git Bash:

```bash
curl -L https://ziglang.org/download/0.14.0/zig-windows-x86_64-0.14.0.zip -o zig.zip
unzip zig.zip
# Add zig-windows-x86_64-0.14.0/ to your PATH
```

## Verify

```bash
zig version
# Should print: 0.14.0
```

## That's it

No `apt`, no `brew`, no `nvm`-style version manager. Download, extract, put on PATH. One binary, zero dependencies.

## Architecture check

If you're unsure about your architecture:

```bash
uname -m
# x86_64 → use x86_64 downloads
# aarch64 or arm64 → use aarch64 downloads
```
