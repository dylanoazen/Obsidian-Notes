 
O protocolo TCP permite que dois processos troquem bytes pela rede.

TCP NÃO é como:
- REST APIs
- JSON
- abstrações de request/response

TCP só entende:
# byte streams.

Tudo que é enviado pelo TCP eventualmente se torna bytes, incluindo:
- strings
- JSON
- structs
- files

O servidor que recebe os bytes deve interpretar e fazer o parse dos dados corretamente.

---

# Listener

Um listener é a porta aberta onde o servidor aguarda conexões TCP de entrada.

Exemplo:

```text
localhost:8080
```

O servidor fica escutando por clients que tentam se conectar.

---

# Connection (`conn`)

Exemplo de conexão de cliente:

```bash
nc localhost 8080
```

Quando um client conecta, o servidor recebe um objeto de conexão TCP (`conn`).

Essa conexão representa um canal de comunicação entre:
- client
- servidor

---

# Stream

Uma conexão TCP é um fluxo contínuo de bytes.

TCP NÃO conhece:
- mensagens
- comandos
- estruturas JSON
- limites de requisição

Por causa disso, a própria aplicação deve definir:
# como as mensagens são separadas e interpretadas.

Exemplo:

```text
SET name Dylan\n
```

A aplicação pode decidir que:
- cada linha
- ou cada delimitador

representa um comando completo.

---

# Framing

Como o TCP é um byte stream sem limites de mensagem, a aplicação deve definir como separar as mensagens. Isso se chama framing.

Veja [[Framing]] para o detalhamento completo dos tipos de framing e como implementá-los.

---

# Meus Comentários:

TCP fornece um byte stream confiável e ordenado. Uma conexão é estabelecida com um three-way handshake: o client envia SYN, o servidor responde com SYN-ACK, e o client envia ACK (que pode incluir dados). Depois disso, os dois lados trocam bytes em ordem; o TCP não preserva limites de mensagem. 
O byte stream é a forma como as máquinas enviam e recebem dados e funciona assim:
TCP **não** envia primeiro o comprimento total do stream. Em vez disso, divide os dados em segmentos, cada um com um **número de sequência**. O receptor confirma o maior número de sequência contínuo recebido (ACK). Segmentos perdidos são retransmitidos. É assim que o TCP garante entrega confiável e ordenada sobre uma rede não confiável.