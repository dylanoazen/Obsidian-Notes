# Methods

A method is a function associated with a type.

## Receiver

A receiver is the value that a method operates on in Go.

A receiver can be passed by value or by pointer.

The main difference is:

- Value receiver:
  receives a copy of the original value.

- Pointer receiver:
  receives a reference to the original value in memory.

Pointer receivers are usually used when the method needs to modify the struct state or avoid unnecessary copies.

## Pointers vs Value Receivers

The main question is not:

```text
"Am I copying or reading?"
```

The real question is:

```text
"Should this method operate on the original object?"
```

## Examples

### Set

Set modifies the original cache state.

Because of that, it should use a pointer receiver:

```go
func (c *Cache) Set()
```

The method needs to operate on the real object stored in memory.

---

### Delete

Delete also modifies the original state.

So it should also use a pointer receiver.

---

### Get

Technically, Get could use a value receiver because it only reads data.

However, in Go it is common to keep consistency across methods of the same struct.

So if Set and Delete use pointer receivers, Get will usually also use a pointer receiver.

This keeps the API behavior more predictable and consistent.

---

## Temporary Modifications vs Original State

Sometimes you want to modify data temporarily without changing the original object.

Example:

```text
Original product price = 100
```

During a sale, you may want to:

- apply discounts
- calculate taxes
- create promotional prices

without modifying the original product stored in memory.

In this case, value semantics can make more sense.

Example:

```go
func (p Product) ApplyDiscount(percent float64) Product
```

This suggests:

```text
"Create a modified copy for temporary use"
```

instead of:

```text
"Modify the original object"
```

This reduces side effects and makes behavior more predictable.

---

## net.Conn

### Read

Part of the `io.Reader` interface. Reads data from the connection into a byte slice.

**Syntax:**
```go
Read(b []byte) (n int, err error)
```

#### My Comments
Used when you want to receive data that arrived through the connection. In practice, it's the method you call on a server to read what the client sent. It waits until data arrives and fills the buffer with it.

---

### Write

Part of the `io.Writer` interface. Writes data from a byte slice into the connection.

**Syntax:**
```go
Write(b []byte) (n int, err error)
```

#### My Comments
The opposite of Read — used to send data through the connection to the other side.

---

### Close

Part of the `io.Closer` interface. Closes the connection and releases any associated resources.

**Syntax:**
```go
Close() error
```

#### My Comments
Terminates the connection and frees its resources. Should always be called when you're done using the connection — usually with `defer` to guarantee it runs even if an error occurs.

---

### LocalAddr

Returns the local network address of the connection (your side).

**Syntax:**
```go
LocalAddr() Addr
```

#### My Comments
Returns your own address in the connection. Useful for debugging or when your server is running on multiple network interfaces and you want to know which one is being used.

---

### RemoteAddr

Returns the remote network address of the connection (the other side).

**Syntax:**
```go
RemoteAddr() Addr
```

#### My Comments
Returns the address of whoever is connected to you. Commonly used in servers to log or identify which client is talking.

---

### SetDeadline

Sets a deadline for both read and write operations. After the deadline, any operation will return an error.

**Syntax:**
```go
SetDeadline(t time.Time) error
```

#### My Comments
Sets a time limit for everything — if the operation doesn't finish in time, it returns an error. Prevents your application from hanging forever waiting for data that may never arrive.

---

### SetReadDeadline

Sets a deadline only for read operations.

**Syntax:**
```go
SetReadDeadline(t time.Time) error
```

#### My Comments
Same as SetDeadline but only for reads. Useful when you want to give more time for sending but less time for waiting on a response.

---

### SetWriteDeadline

Sets a deadline only for write operations.

**Syntax:**
```go
SetWriteDeadline(t time.Time) error
```

#### My Comments
Same as SetDeadline but only for writes. Useful to ensure that sending doesn't hang if the network is slow.

---

## net.Listener

### Accept

Part of the `net.Listener` interface. Waits for and returns the next incoming connection.

**Syntax:**
```go
Accept() (Conn, error)
```

#### My Comments
The method that blocks — meaning the program stops on this line and waits — until a new client connects. Once a client connects, it returns a Conn so you can handle it in a goroutine in parallel, keeping the server free to accept more connections.

---

### Close

Part of the `net.Listener` interface. Stops the listener from accepting new connections.

**Syntax:**
```go
Close() error
```

#### My Comments
Used to shut down the server entirely. But the function doesn't close already accepted connections — those should be closed individually.

---

### Addr

Part of the `net.Listener` interface. Returns the address the listener is bound to.

**Syntax:**
```go
Addr() Addr
```

#### My Comments
Useful for debugging or when you need to know which port the OS picked, especially when starting the listener on port `:0`.

## net.PacketConn

### ReadFrom

Reads a packet from the connection and returns the data and the sender's address.

**Syntax:**
```go
ReadFrom(b []byte) (n int, addr Addr, err error)
```

#### My Comments

---

### WriteTo

Sends a packet to a specific address. The destination is chosen on every send.

**Syntax:**
```go
WriteTo(b []byte, addr Addr) (n int, err error)
```

#### My Comments

---

### Close

Closes the PacketConn and releases any associated resources.

**Syntax:**
```go
Close() error
```

#### My Comments

---

### LocalAddr

Returns the local network address of the PacketConn.

**Syntax:**
```go
LocalAddr() Addr
```

#### My Comments

---

### SetDeadline

Sets a deadline for both read and write operations. After the deadline, any operation will return an error.

**Syntax:**
```go
SetDeadline(t time.Time) error
```

#### My Comments

