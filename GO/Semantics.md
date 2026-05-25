
This section explains the semantic meaning behind the Go cache implementation.

The goal is not only to understand the syntax, but also to understand:
- ownership
- mutability
- visibility
- memory behavior
- component boundaries

---

# Cache Structure

```go
type Cache struct {
	data map[string]string
}
```

## Semantics

### `type Cache`
Creates a new type in the language.

In Go, types are first-class citizens, not just classes or objects.

---

### `struct`
A struct is a group of related data.

Go structs are usually about:
- composition
- state
- organization

not deep object-oriented hierarchies.

---

### `data`
Lowercase names are private to the package.

```go
data
```

means:
- internal field
- not accessible outside the package

Uppercase names are exported/public:

```go
Data
```

Go uses capitalization as visibility control instead of keywords like:
- private
- public
- protected

---

### `map[string]string`
Represents a hash map:
- dynamic
- mutable
- fast lookup

Maps are reference-like structures internally.

---

# Methods and Receivers

```go
func (c *Cache) Set(key, value string)
```

## Semantics

### `func`
Defines a function.

---

### `(c *Cache)`
This is a receiver.

It means:
> "This method operates on a Cache instance."

---

### `*Cache`
Pointer receiver.

It means:
> "Operate on the original object in memory."

Used when:
- modifying state
- avoiding unnecessary copies
- working with shared resources

---

### `c`
Just a short local variable name for the receiver.

Small names are common in Go when the scope is small and obvious.

---

# Pointer vs Value Receivers

The important question is not:

```text
"Am I reading or writing?"
```

The real question is:

```text
"Should this method operate on the original object?"
```

---

## Pointer Receiver

```go
func (c *Cache) Set()
```

Used when:
- modifying shared state
- working on the original object
- managing mutable components

Examples:
- SET
- DELETE

---

## Value Receiver

```go
func (p Product) ApplyDiscount()
```

Usually used for:
- temporary transformations
- immutable behavior
- calculations
- avoiding side effects

Example:
- applying discounts without changing the original product

---

# GET Method

```go
func (c *Cache) Get(key string) (string, bool)
```

## Semantics

Go supports multiple explicit return values.

Instead of:
- exceptions
- null
- hidden states

Go prefers:
- explicit values
- explicit status checking

---

### `bool`
Represents whether the key exists.

Example:

```go
value, exists := cache.Get("name")
```

This is very idiomatic in Go.

---

# DELETE

```go
delete(c.data, key)
```

`delete` is a built-in language function.

It removes an entry from the map.

---

# Constructor Pattern

```go
func NewCache() *Cache
```

Go does not have official constructors.

The idiomatic convention is:

```text
New<Type>()
```

Example:

```go
func NewCache() *Cache
```

---

# `&Cache{}`

```go
&Cache{}
```

The `&` operator returns the memory address of the object.

This creates a pointer to the struct.

---

# `make()`

```go
make(map[string]string)
```

Used to initialize runtime-backed structures like:
- maps
- slices
- channels

Without `make`, a map is `nil` and writing to it causes a panic.

---

# Final Semantic Summary

This simple cache implementation already introduces:

- custom types
- encapsulation
- mutable state
- public APIs
- pointer semantics
- ownership
- memory behavior
- package visibility
- component boundaries

Even small Go programs already teach important systems engineering concepts.