bufio é um package que cria um buffer por cima de uma conexão e a envolve com funcionalidades extras (como `ReadString`). Isso torna a leitura muito mais fácil.

Quando usar com o [[Net]]
- Protocolos com caracteres delimitadores
- Quando você precisa ler o byte stream passo a passo## Read

**Syntax:**
```go
Read([]byte)
```
Lê bytes do buffer/byte stream.

## ReadString

**Syntax:**
```go
ReadString(delim byte)
```
Lê até encontrar o caractere delimitador (ex.: `\n`).

## ReadBytes

**Syntax:**
```go
ReadBytes(delim byte)
```
Igual a `ReadString`, mas retorna `[]byte`.

## ReadLine

**Syntax:**
```go
ReadLine()
```
Lê uma linha "crua" (pode retornar uma linha fragmentada).

## Peek

**Syntax:**
```go
Peek(n int)
```
Espia `n` bytes sem consumi-los.

## Discard

**Syntax:**
```go
Discard(n int)
```
Descarta `n` bytes do buffer.

## Buffered

**Syntax:**
```go
Buffered()
```
Mostra quantos bytes estão atualmente no buffer.

## Reset

**Syntax:**
```go
Reset(r io.Reader)
```
Reutiliza o reader com outra fonte.
