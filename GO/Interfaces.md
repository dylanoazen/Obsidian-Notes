# Interfaces em Go

Contratos implícitos — qualquer tipo que implementar os métodos satisfaz a interface.

Related: [[GO/Structs]], [[GO/Methods]], [[GO/GO]]

---

## O que é

Em Go, uma interface é um conjunto de métodos. Qualquer tipo que tiver esses métodos satisfaz a interface — sem precisar declarar `implements`.

```go
// define o contrato
type Sender interface {
    Send(ctx context.Context, externalID string, p Product) error
}

// tray.Client satisfaz Sender — tem o método Send com a mesma assinatura
type Client struct { ... }
func (c *Client) Send(ctx context.Context, externalID string, p Product) error { ... }

// fakeSender também satisfaz — usado em testes e desenvolvimento
type fakeSender struct{}
func (f *fakeSender) Send(ctx context.Context, key string, p Product) error { ... }
```

Nenhum dos dois declara `implements Sender`. O compilador verifica automaticamente.

---

## Por que usar — Inversão de Dependência

O `Worker` não sabe de `tray.Client`. Ele só conhece `domain.Sender`:

```go
type Worker struct {
    jobSource domain.ProductSource  // interface
    sender    domain.Sender         // interface
}
```

```
main.go decide qual implementação usar:
  ├── development → &fakeSender{}   (imprime na tela)
  └── production  → tray.NewClient(...) (envia de verdade)

Worker não muda — só recebe a interface
```

Trocar a implementação é só mudar o `main.go`. O `Worker` não sabe da diferença.

---

## Interfaces do TrayGo

```go
// quem fornece os jobs
type ProductSource interface {
    ClaimBatch(ctx context.Context, n int) ([]SendJob, error)
    MarkDone(ctx context.Context, id JobID, externalID string) error
    MarkFailed(ctx context.Context, id JobID, errMsg string) error
}

// quem envia os produtos
type Sender interface {
    Send(ctx context.Context, externalID string, p Product) error
}
```

**Implementações de `ProductSource`:**
- `memory.Source` — slice em memória, para desenvolvimento e testes
- `mariadb.Source` — banco real, para produção

**Implementações de `Sender`:**
- `fakeSender` (main.go) — só imprime, para desenvolvimento
- `tray.Client` — envia para a API da Tray, para produção

---

## Interface Vazia e Type Assertion

```go
// interface vazia — qualquer tipo
var anything interface{}
anything = "texto"
anything = 42

// type assertion — extrair o tipo concreto
s := anything.(string)  // panic se não for string
s, ok := anything.(string)  // ok=false se não for, sem panic
```

---

## Nil Interface vs Nil Pointer

Um erro clássico em Go:

```go
var s *fakeSender = nil  // ponteiro nil
var i domain.Sender = s  // interface com tipo *fakeSender mas valor nil

i == nil  // FALSE — a interface tem tipo, mesmo que o valor seja nil
```

Interface só é `nil` quando **tanto o tipo quanto o valor** são nil.

---

## Interfaces Pequenas São Melhores

```go
// Go prefere interfaces com poucos métodos
type Sender interface {
    Send(ctx context.Context, externalID string, p Product) error
}
// 1 método → fácil de implementar, fácil de mockar em teste
```

A interface `io.Reader` da stdlib tem 1 método. `io.Writer` tem 1. Isso é intencional.

---

## Related

- [[GO/Structs]]
- [[GO/Methods]]
- [[GO/GO]]

#### My commentaries
-
