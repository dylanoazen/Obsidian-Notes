# Methods

Um method é uma função associada a um tipo.

## Receiver

Um receiver é o valor sobre o qual um method opera em Go.

Um receiver pode ser passado por valor ou por ponteiro.

A principal diferença é:

- Value receiver:
  recebe uma cópia do valor original.

- Pointer receiver:
  recebe uma referência ao valor original na memória.

Pointer receivers são geralmente usados quando o method precisa modificar o estado da struct ou evitar cópias desnecessárias.

## Pointers vs Value Receivers

A principal questão não é:

```text
"Am I copying or reading?"
```

A questão real é:

```text
"Should this method operate on the original object?"
```

## Exemplos

### Set

Set modifica o estado original do cache.

Por isso, deve usar um pointer receiver:

```go
func (c *Cache) Set()
```

O method precisa operar no objeto real armazenado na memória.

---

### Delete

Delete também modifica o estado original.

Portanto, também deve usar um pointer receiver.

---

### Get

Tecnicamente, Get poderia usar um value receiver porque apenas lê dados.

No entanto, em Go é comum manter consistência entre os methods da mesma struct.

Portanto, se Set e Delete usam pointer receivers, Get geralmente também usará um pointer receiver.

Isso mantém o comportamento da API mais previsível e consistente.

---

## Modificações Temporárias vs Estado Original

Às vezes você quer modificar dados temporariamente sem alterar o objeto original.

Exemplo:

```text
Original product price = 100
```

Durante uma promoção, você pode querer:

- aplicar descontos
- calcular impostos
- criar preços promocionais

sem modificar o produto original armazenado na memória.

Nesse caso, value semantics pode fazer mais sentido.

Exemplo:

```go
func (p Product) ApplyDiscount(percent float64) Product
```

Isso sugere:

```text
"Create a modified copy for temporary use"
```

em vez de:

```text
"Modify the original object"
```

Isso reduz efeitos colaterais e torna o comportamento mais previsível.

---

## net.Conn

### Read

Parte da interface `io.Reader`. Lê dados da conexão em um byte slice.

**Syntax:**
```go
Read(b []byte) (n int, err error)
```

#### Meus Comentários
Usado quando você quer receber dados que chegaram pela conexão. Na prática, é o method que você chama em um servidor para ler o que o cliente enviou. Ele aguarda até que os dados cheguem e preenche o buffer com eles.

---

### Write

Parte da interface `io.Writer`. Escreve dados de um byte slice na conexão.

**Syntax:**
```go
Write(b []byte) (n int, err error)
```

#### Meus Comentários
O oposto de Read — usado para enviar dados pela conexão para o outro lado.

---

### Close

Parte da interface `io.Closer`. Fecha a conexão e libera quaisquer recursos associados.

**Syntax:**
```go
Close() error
```

#### Meus Comentários
Encerra a conexão e libera seus recursos. Deve sempre ser chamado quando você terminar de usar a conexão — geralmente com `defer` para garantir que rode mesmo se ocorrer um erro.

---

### LocalAddr

Retorna o endereço de rede local da conexão (seu lado).

**Syntax:**
```go
LocalAddr() Addr
```

#### Meus Comentários
Retorna seu próprio endereço na conexão. Útil para debug ou quando seu servidor está rodando em múltiplas interfaces de rede e você quer saber qual está sendo usada.

---

### RemoteAddr

Retorna o endereço de rede remoto da conexão (o outro lado).

**Syntax:**
```go
RemoteAddr() Addr
```

#### Meus Comentários
Retorna o endereço de quem está conectado a você. Comumente usado em servidores para registrar em log ou identificar qual cliente está se comunicando.

---

### SetDeadline

Define um deadline para operações de leitura e escrita. Após o deadline, qualquer operação retornará um erro.

**Syntax:**
```go
SetDeadline(t time.Time) error
```

#### Meus Comentários
Define um limite de tempo para tudo — se a operação não terminar a tempo, retorna um erro. Evita que sua aplicação fique travada para sempre aguardando dados que talvez nunca cheguem.

---

### SetReadDeadline

Define um deadline apenas para operações de leitura.

**Syntax:**
```go
SetReadDeadline(t time.Time) error
```

#### Meus Comentários
Igual a SetDeadline mas apenas para leituras. Útil quando você quer dar mais tempo para enviar, mas menos tempo para aguardar uma resposta.

---

### SetWriteDeadline

Define um deadline apenas para operações de escrita.

**Syntax:**
```go
SetWriteDeadline(t time.Time) error
```

#### Meus Comentários
Igual a SetDeadline mas apenas para escritas. Útil para garantir que o envio não trave se a rede estiver lenta.

---

## net.Listener

### Accept

Parte da interface `net.Listener`. Aguarda e retorna a próxima conexão de entrada.

**Syntax:**
```go
Accept() (Conn, error)
```

#### Meus Comentários
O method que bloqueia — ou seja, o programa para nessa linha e aguarda — até que um novo cliente se conecte. Quando um cliente se conecta, retorna um Conn para que você possa tratá-lo em uma goroutine em paralelo, mantendo o servidor livre para aceitar mais conexões.

---

### Close

Parte da interface `net.Listener`. Para o listener de aceitar novas conexões.

**Syntax:**
```go
Close() error
```

#### Meus Comentários
Usado para desligar o servidor completamente. Mas a função não fecha conexões já aceitas — essas devem ser fechadas individualmente.

---

### Addr

Parte da interface `net.Listener`. Retorna o endereço ao qual o listener está vinculado.

**Syntax:**
```go
Addr() Addr
```

#### Meus Comentários
Útil para debug ou quando você precisa saber qual porta o OS escolheu, especialmente ao iniciar o listener na porta `:0`.

## net.PacketConn

### ReadFrom

Lê um pacote da conexão e retorna os dados e o endereço do remetente.

**Syntax:**
```go
ReadFrom(b []byte) (n int, addr Addr, err error)
```

#### Meus Comentários

---

### WriteTo

Envia um pacote para um endereço específico. O destino é escolhido a cada envio.

**Syntax:**
```go
WriteTo(b []byte, addr Addr) (n int, err error)
```

#### Meus Comentários

---

### Close

Fecha o PacketConn e libera quaisquer recursos associados.

**Syntax:**
```go
Close() error
```

#### Meus Comentários

---

### LocalAddr

Retorna o endereço de rede local do PacketConn.

**Syntax:**
```go
LocalAddr() Addr
```

#### Meus Comentários

---

### SetDeadline

Define um deadline para operações de leitura e escrita. Após o deadline, qualquer operação retornará um erro.

**Syntax:**
```go
SetDeadline(t time.Time) error
```

#### Meus Comentários
