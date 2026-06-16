# Framing

Framing é uma técnica para marcar onde uma mensagem começa e onde ela termina dentro de um byte stream do [[TCP]].

Como o [[TCP]] não preserva limites de mensagem, a própria aplicação deve definir como separar as mensagens. Sem framing, o receptor não tem como saber se recebeu uma mensagem completa, metade de uma mensagem ou duas mensagens ao mesmo tempo.

## Tipos de Framing

### Delimiter-based

Usa um caractere especial para sinalizar o fim de uma mensagem. O delimitador mais comum é `\n` (newline).

```text
SET name Dylan\n   ← message 1
GET name\n         ← message 2
```

O receptor lê byte a byte até encontrar o delimitador, então processa a mensagem completa.

**Usado em:** Redis protocol, HTTP headers, servidores TCP simples.

**Desvantagem:** O caractere delimitador não pode aparecer dentro dos dados da mensagem.

---

### Length-prefix

Envia o tamanho da mensagem antes da própria mensagem. O receptor lê o comprimento primeiro, depois lê exatamente aquela quantidade de bytes.

```text
[0 0 0 11] [SET name Dylan]
 ↑ 4 bytes    ↑ 11 bytes
  (length)    (message)
```

```go
// sending
binary.Write(conn, binary.BigEndian, uint32(len(msg)))
conn.Write(msg)

// receiving
var length uint32
binary.Read(conn, binary.BigEndian, &length)
buf := make([]byte, length)
io.ReadFull(conn, buf)
```

**Usado em:** gRPC, databases, a maioria dos protocolos binários.

**Vantagem:** Funciona com qualquer dado, incluindo binário.

---

### Fixed-size

Cada mensagem sempre tem o mesmo tamanho. O receptor sempre lê exatamente N bytes.

```text
[msg 1 - 64 bytes][msg 2 - 64 bytes][msg 3 - 64 bytes]
```

**Usado em:** protocolos de hardware de baixo nível, alguns game engines.

**Desvantagem:** Desperdiça espaço se as mensagens forem menores que o tamanho fixo.

---

## Resumo

| Tipo | Como funciona | Melhor para |
|------|-------------|----------|
| Delimiter | Caractere especial marca o fim | Protocolos de texto, servidores simples |
| Length-prefix | Tamanho enviado antes da mensagem | Dados binários, sistemas em produção |
| Fixed-size | Cada mensagem tem o mesmo tamanho | Tamanhos de mensagem conhecidos e previsíveis |
