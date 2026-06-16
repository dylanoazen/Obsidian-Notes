Go não é uma linguagem orientada a hierarquias.

Basicamente, você tem componentes que colaboram entre si em vez de serem extensões uns dos outros.

Objetos "têm" outros objetos, eles não "se tornam" eles.

Por exemplo, um Client tem uma Person.
O Client pode usar partes de Person, mas não é uma Person em si.

Isso cria um acoplamento mais fraco entre os componentes.

Portanto, se eu mudar algo dentro de Person, isso não necessariamente afeta Client diretamente.
## Composition vs Inheritance em Go

Go prefere composição a herança.

Em vez de criar hierarquias profundas como:

Person
 └── Client
 └── Seller

Go geralmente modela relacionamentos usando composição.

## Exemplo

Em vez de:

```text
Seller extends Person
```

Go usa:

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

## Por que isso é importante?

Com herança, se você adicionar um campo obrigatório como Address a Person:

```text
Person
 ├── CPF
 ├── RG
 ├── Phone
 └── Address
```

todo filho automaticamente o herda.

Isso significa que Seller também exigiria Address, mesmo que não faça sentido para as regras de negócio.

Isso cria um forte acoplamento entre as entidades.

## Composition dá flexibilidade

Com composição:

- Client pode ter Address
- Seller pode não ter Address
- componentes permanecem reutilizáveis
- entidades permanecem independentes
- mudanças se tornam mais localizadas

Em vez de dizer:

```text
Client IS A Person
```

Go geralmente prefere:

```text
Client HAS A Person
```

Isso reduz a complexidade da hierarquia e cria sistemas mais fáceis de manter.
