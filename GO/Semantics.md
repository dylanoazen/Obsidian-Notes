
Esta seção explica o significado semântico por trás da implementação do cache em Go.

O objetivo não é apenas entender a sintaxe, mas também compreender:
- ownership
- mutabilidade
- visibilidade
- comportamento de memória
- limites de componentes

---

# Cache Structure

```go
type Cache struct {
	data map[string]string
}
```

## Semantics

### `type Cache`
Cria um novo tipo na linguagem.

Em Go, tipos são cidadãos de primeira classe, não apenas classes ou objetos.

---

### `struct`
Uma struct é um grupo de dados relacionados.

Structs em Go geralmente tratam de:
- composição
- estado
- organização

e não de hierarquias profundas orientadas a objetos.

---

### `data`
Nomes em minúsculo são privados ao package.

```go
data
```

significa:
- campo interno
- não acessível fora do package

Nomes em maiúsculo são exportados/públicos:

```go
Data
```

Go usa a capitalização como controle de visibilidade em vez de keywords como:
- private
- public
- protected

---

### `map[string]string`
Representa um hash map:
- dinâmico
- mutável
- busca rápida

Maps são estruturas do tipo referência internamente.

---

# Methods and Receivers

```go
func (c *Cache) Set(key, value string)
```

## Semantics

### `func`
Define uma função.

---

### `(c *Cache)`
Isso é um receiver.

Significa:
> "Este method opera em uma instância de Cache."

---

### `*Cache`
Pointer receiver.

Significa:
> "Operar no objeto original na memória."

Usado quando:
- modificando estado
- evitando cópias desnecessárias
- trabalhando com recursos compartilhados

---

### `c`
Apenas um nome de variável local curto para o receiver.

Nomes curtos são comuns em Go quando o escopo é pequeno e óbvio.

---

# Pointer vs Value Receivers

A questão importante não é:

```text
"Am I reading or writing?"
```

A questão real é:

```text
"Should this method operate on the original object?"
```

---

## Pointer Receiver

```go
func (c *Cache) Set()
```

Usado quando:
- modificando estado compartilhado
- trabalhando no objeto original
- gerenciando componentes mutáveis

Exemplos:
- SET
- DELETE

---

## Value Receiver

```go
func (p Product) ApplyDiscount()
```

Geralmente usado para:
- transformações temporárias
- comportamento imutável
- cálculos
- evitar efeitos colaterais

Exemplo:
- aplicar descontos sem alterar o produto original

---

# GET Method

```go
func (c *Cache) Get(key string) (string, bool)
```

## Semantics

Go suporta múltiplos valores de retorno explícitos.

Em vez de:
- exceptions
- null
- estados ocultos

Go prefere:
- valores explícitos
- verificação explícita de status

---

### `bool`
Representa se a key existe.

Exemplo:

```go
value, exists := cache.Get("name")
```

Isso é muito idiomático em Go.

---

# DELETE

```go
delete(c.data, key)
```

`delete` é uma função built-in da linguagem.

Ela remove uma entrada do map.

---

# Constructor Pattern

```go
func NewCache() *Cache
```

Go não tem construtores oficiais.

A convenção idiomática é:

```text
New<Type>()
```

Exemplo:

```go
func NewCache() *Cache
```

---

# `&Cache{}`

```go
&Cache{}
```

O operador `&` retorna o endereço de memória do objeto.

Isso cria um ponteiro para a struct.

---

# `make()`

```go
make(map[string]string)
```

Usado para inicializar estruturas com suporte em runtime como:
- maps
- slices
- channels

Sem `make`, um map é `nil` e escrever nele causa um panic.

---

# Resumo Semântico Final

Esta implementação simples de cache já introduz:

- tipos customizados
- encapsulamento
- estado mutável
- APIs públicas
- pointer semantics
- ownership
- comportamento de memória
- visibilidade de package
- limites de componentes

Mesmo programas pequenos em Go já ensinam conceitos importantes de engenharia de sistemas.
