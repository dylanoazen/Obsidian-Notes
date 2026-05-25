bufio is a package that creates a buffer on top of a connection and wraps it with extra features (like `ReadString`). This makes reading much easier.

When use on the [[Net]]
- Protocols with delimiter characters
- When you need to read the byte stream step by step## Read

**Syntax:**
```go
Read([]byte)
```
Reads bytes from the buffer/byte stream.

## ReadString

**Syntax:**
```go
ReadString(delim byte)
```
Reads until it finds the delimiter character (e.g. `\n`).

## ReadBytes

**Syntax:**
```go
ReadBytes(delim byte)
```
Same as `ReadString`, but returns `[]byte`.

## ReadLine

**Syntax:**
```go
ReadLine()
```
Reads one "raw" line (it can return a fragmented line).

## Peek

**Syntax:**
```go
Peek(n int)
```
Peeks `n` bytes without consuming them.

## Discard

**Syntax:**
```go
Discard(n int)
```
Discards `n` bytes from the buffer.

## Buffered

**Syntax:**
```go
Buffered()
```
Shows how many bytes are currently in the buffer.

## Reset

**Syntax:**
```go
Reset(r io.Reader)
```
Reuses the reader with another source.
