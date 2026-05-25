Go isn't a hierarchy-oriented language.

Basically, you have components that collaborate with each other instead of being extensions of one another.

Objects "have" other objects, they don't "become" them.

For example, a Client has a Person.
The Client can use parts of Person, but it isn't a Person itself.

This creates a looser coupling between components.

So if I change something inside Person, it doesn't necessarily affect Client directly.
## Composition vs Inheritance in Go

Go prefers composition over inheritance.

Instead of creating deep hierarchies like:

Person
 └── Client
 └── Seller

Go usually models relationships using composition.

## Example

Instead of:

```text
Seller extends Person
```

Go uses:

```go
type Person struct {
	CPF   string
	RG    string
	Phone string
}

type Address struct {
	Street string
	City   string
}

type Client struct {
	Person
	Address
}

type Seller struct {
	Person
}
```

## Why is this important?

With inheritance, if you add a required field like Address to Person:

```text
Person
 ├── CPF
 ├── RG
 ├── Phone
 └── Address
```

every child automatically inherits it.

That means Seller would also require Address, even if it doesn't make sense for the business rules.

This creates strong coupling between entities.

## Composition gives flexibility

With composition:

- Client can have Address
- Seller can avoid Address
- components stay reusable
- entities stay independent
- changes become more localized

Instead of saying:

```text
Client IS A Person
```

Go usually prefers:

```text
Client HAS A Person
```

This reduces hierarchy complexity and creates more maintainable systems.