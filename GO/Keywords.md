# Keywords

Go tem exatamente **25 keywords reservadas**. Elas não podem ser usadas como nomes de variáveis, funções ou identificadores.

---

## Declarações

### var
Declara uma variável.
```go
var name string = "Dylan"
var age int // zero value = 0
```

### const
Declara uma constante — um valor que não pode mudar.
```go
const Pi = 3.14
const MaxSize = 100
```

### type
Declara um novo tipo.
```go
type User struct {
    Name string
    Age  int
}
type ID int // alias
```

### func
Declara uma função.
```go
func add(a int, b int) int {
    return a + b
}
```

### package
Declara a qual package o arquivo pertence.
```go
package main
```

### import
Importa packages externos.
```go
import "fmt"
import (
    "fmt"
    "net"
)
```

---

## Tipos de Dados

### struct
Define uma estrutura — um grupo de campos.
```go
type Point struct {
    X int
    Y int
}
```

### interface
Define um conjunto de métodos que um tipo deve implementar.
```go
type Animal interface {
    Sound() string
}
```

### map
Declara uma coleção chave/valor.
```go
m := map[string]int{
    "apples": 5,
}
```

### chan
Declara um channel — usado para comunicação entre goroutines.
```go
ch := make(chan int)
ch <- 42  // send
v := <-ch // receive
```

---

## Controle de Fluxo

### if / else
Execução condicional.
```go
if age > 18 {
    fmt.Println("adult")
} else {
    fmt.Println("minor")
}
```

### for
O único loop do Go — funciona como while, for e foreach.
```go
for i := 0; i < 10; i++ {}       // classic
for i < 10 {}                     // while-style
for range slice {}                // foreach-style
```

### range
Itera sobre arrays, slices, maps, strings e channels.
```go
for i, v := range []int{1, 2, 3} {
    fmt.Println(i, v)
}
```

### switch / case / default
Condicional com múltiplos ramos.
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
Força a execução continuar para o próximo case (incomum em Go).
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
Como switch, mas para channels — espera pelo channel que ficar pronto primeiro.
```go
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case msg := <-ch2:
    fmt.Println("from ch2:", msg)
}
```

---

## Controle de Execução

### return
Sai de uma função e opcionalmente retorna valores.
```go
func double(n int) int {
    return n * 2
}
```

### break
Sai de um loop ou switch antecipadamente.
```go
for i := 0; i < 10; i++ {
    if i == 5 {
        break // stops at 5
    }
}
```

### continue
Pula para a próxima iteração de um loop.
```go
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue // skip even numbers
    }
    fmt.Println(i)
}
```

### goto
Salta para uma linha marcada com label. Raramente usado — considerado má prática na maioria dos casos.
```go
goto end
fmt.Println("this is skipped")
end:
fmt.Println("jumped here")
```

---

## Concorrência

### go
Inicia uma goroutine — executa uma função concorrentemente.
```go
go doSomething()
go func() {
    fmt.Println("running in parallel")
}()
```

### defer
Agenda uma função para rodar quando a função ao redor retornar. Executa em ordem LIFO.
```go
func readFile() {
    f, _ := os.Open("file.txt")
    defer f.Close() // guaranteed to run at the end
    // work with f...
}
```

---

## Tabela Resumo

| Keyword | Category | Purpose |
|---------|----------|---------|
| `var` | Declaration | Declarar variável |
| `const` | Declaration | Declarar constante |
| `type` | Declaration | Declarar novo tipo |
| `func` | Declaration | Declarar função |
| `package` | Declaration | Declarar package |
| `import` | Declaration | Importar packages |
| `struct` | Type | Grupo de campos |
| `interface` | Type | Conjunto de métodos |
| `map` | Type | Coleção chave/valor |
| `chan` | Type | Channel de comunicação |
| `if` | Control | Condicional |
| `else` | Control | Alternativa condicional |
| `for` | Control | Loop |
| `range` | Control | Iterar sobre coleção |
| `switch` | Control | Condicional com múltiplos ramos |
| `case` | Control | Ramo do switch |
| `default` | Control | Fallback do switch |
| `fallthrough` | Control | Continuar para o próximo case |
| `select` | Control | Aguardar channels |
| `return` | Flow | Sair da função |
| `break` | Flow | Sair do loop ou switch |
| `continue` | Flow | Próxima iteração do loop |
| `goto` | Flow | Saltar para label |
| `go` | Concurrency | Iniciar goroutine |
| `defer` | Concurrency | Atrasar chamada de função |
