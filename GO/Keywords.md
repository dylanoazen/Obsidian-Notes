# Keywords

Go has exactly **25 reserved keywords**. They cannot be used as variable names, function names, or identifiers.

---

## Declarations

### var
Declares a variable.
```go
var name string = "Dylan"
var age int // zero value = 0
```

### const
Declares a constant — a value that cannot change.
```go
const Pi = 3.14
const MaxSize = 100
```

### type
Declares a new type.
```go
type User struct {
    Name string
    Age  int
}
type ID int // alias
```

### func
Declares a function.
```go
func add(a int, b int) int {
    return a + b
}
```

### package
Declares which package the file belongs to.
```go
package main
```

### import
Imports external packages.
```go
import "fmt"
import (
    "fmt"
    "net"
)
```

---

## Data Types

### struct
Defines a structure — a group of fields.
```go
type Point struct {
    X int
    Y int
}
```

### interface
Defines a set of methods a type must implement.
```go
type Animal interface {
    Sound() string
}
```

### map
Declares a key/value collection.
```go
m := map[string]int{
    "apples": 5,
}
```

### chan
Declares a channel — used to communicate between goroutines.
```go
ch := make(chan int)
ch <- 42  // send
v := <-ch // receive
```

---

## Control Flow

### if / else
Conditional execution.
```go
if age > 18 {
    fmt.Println("adult")
} else {
    fmt.Println("minor")
}
```

### for
The only loop in Go — works as while, for, and foreach.
```go
for i := 0; i < 10; i++ {}       // classic
for i < 10 {}                     // while-style
for range slice {}                // foreach-style
```

### range
Iterates over arrays, slices, maps, strings, and channels.
```go
for i, v := range []int{1, 2, 3} {
    fmt.Println(i, v)
}
```

### switch / case / default
Multi-branch conditional.
```go
switch status {
case "ok":
    fmt.Println("all good")
case "error":
    fmt.Println("something failed")
default:
    fmt.Println("unknown")
}
```

### fallthrough
Forces execution to continue into the next case (unusual in Go).
```go
switch x {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two") // runs even if x == 1
}
```

### select
Like switch, but for channels — waits for whichever channel is ready first.
```go
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case msg := <-ch2:
    fmt.Println("from ch2:", msg)
}
```

---

## Flow Control

### return
Exits a function and optionally returns values.
```go
func double(n int) int {
    return n * 2
}
```

### break
Exits a loop or switch early.
```go
for i := 0; i < 10; i++ {
    if i == 5 {
        break // stops at 5
    }
}
```

### continue
Skips to the next iteration of a loop.
```go
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue // skip even numbers
    }
    fmt.Println(i)
}
```

### goto
Jumps to a labeled line. Rarely used — considered bad practice in most cases.
```go
goto end
fmt.Println("this is skipped")
end:
fmt.Println("jumped here")
```

---

## Concurrency

### go
Starts a goroutine — runs a function concurrently.
```go
go doSomething()
go func() {
    fmt.Println("running in parallel")
}()
```

### defer
Schedules a function to run when the surrounding function returns. Runs in LIFO order.
```go
func readFile() {
    f, _ := os.Open("file.txt")
    defer f.Close() // guaranteed to run at the end
    // work with f...
}
```

---

## Summary Table

| Keyword | Category | Purpose |
|---------|----------|---------|
| `var` | Declaration | Declare variable |
| `const` | Declaration | Declare constant |
| `type` | Declaration | Declare new type |
| `func` | Declaration | Declare function |
| `package` | Declaration | Declare package |
| `import` | Declaration | Import packages |
| `struct` | Type | Group of fields |
| `interface` | Type | Set of methods |
| `map` | Type | Key/value collection |
| `chan` | Type | Communication channel |
| `if` | Control | Conditional |
| `else` | Control | Conditional fallback |
| `for` | Control | Loop |
| `range` | Control | Iterate over collection |
| `switch` | Control | Multi-branch conditional |
| `case` | Control | Branch of switch |
| `default` | Control | Fallback of switch |
| `fallthrough` | Control | Continue to next case |
| `select` | Control | Wait on channels |
| `return` | Flow | Exit function |
| `break` | Flow | Exit loop or switch |
| `continue` | Flow | Next loop iteration |
| `goto` | Flow | Jump to label |
| `go` | Concurrency | Start goroutine |
| `defer` | Concurrency | Delay function call |
