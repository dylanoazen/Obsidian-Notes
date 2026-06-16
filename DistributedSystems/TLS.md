# TLS (Transport Layer Security)

TLS é o protocolo que cria um **túnel criptografado** entre duas partes em uma rede. Ninguém no meio consegue ler ou alterar o que trafega por ele.

Ele roda em cima do TCP — o TCP abre a conexão, o TLS a protege.

---

## Três Objetivos

1. **Identidade** — verificar que o servidor é quem diz ser (certificado)
2. **Troca de chave** — concordar em uma chave secreta sem nunca enviá-la pela rede
3. **Criptografia** — criptografar tudo a partir desse ponto

---

## O TLS Handshake Passo a Passo

### Passo 1 — ClientHello

Você envia ao servidor:
> "Quero me comunicar criptografado. Suporto esses algoritmos. Aqui está um número aleatório meu."

O número aleatório será usado depois para gerar a chave de sessão.

### Passo 2 — ServerHello + Certificado

O servidor responde:
> "Ok, vamos usar este algoritmo. Aqui está meu certificado. Aqui está meu número aleatório."

O certificado é o documento de identidade do servidor — assinado por uma CA (Certificate Authority) em que seu browser já confia.

### Passo 3 — Você Verifica o Certificado

Seu browser verifica:
> "Este certificado foi assinado por alguém em quem confio?"

- Sim → continue
- Não → tela vermelha: "Sua conexão não é privada"

### Passo 4 — Troca de Chave (Diffie-Hellman)

Essa é a parte elegante. Ambos os lados precisam chegar à mesma chave secreta **sem nunca enviá-la pela rede**.

**Analogia das tintas:**

```
Ambos concordam em uma cor base pública: amarelo
                👀 qualquer um pode ver isso

Você escolhe uma cor secreta: azul       (nunca enviada pela rede)
Servidor escolhe uma cor secreta: vermelho    (nunca enviada pela rede)

Você envia:    amarelo + azul   = verde
Servidor envia: amarelo + vermelho   = laranja
                👀 qualquer um pode ver verde e laranja

Você pega laranja e mistura com seu azul secreto   = marrom
Servidor pega verde e mistura com seu vermelho secreto = marrom

Ambos chegaram ao marrom — sem nunca enviar suas cores secretas.
```

**Analogia numérica:**

```
Base pública: 2

Você escolhe secreto:    3    (nunca enviado)
Servidor escolhe secreto: 1   (nunca enviado)

Você envia:    2 + 3 = 5  →
             ← 2 + 1 = 3  :Servidor envia

Você:    3 + seu secreto 3 = 6... espera não

Na verdade:
Você envia ao servidor: base + seu secreto
Servidor envia a você: base + secreto dele
Ambos somam o valor do outro com seu próprio secreto → mesmo resultado
```

Em criptografia, a matemática torna impossível reverter o engenheiro do segredo a partir do que foi enviado publicamente. Um atacante vê os valores públicos mas não consegue derivar o segredo.

### Passo 5 — Tudo Criptografado

Ambos os lados agora compartilham a mesma chave secreta. Toda comunicação daqui em diante é criptografada com ela usando **criptografia simétrica** — rápida e leve.

---

## Simétrico vs Assimétrico no TLS

### Criptografia Simétrica

**Apenas uma chave** — quem criptografa e quem descriptografa usam a mesma chave.

```
Você      →  encrypt with key 🔑  →  "█▓▒░"
Servidor  →  decrypt with key 🔑  →  "hello"
```

**Problema:** como você concorda com o servidor sobre a chave sem enviá-la pela rede? Se alguém interceptar, acabou.

**Vantagem:** muito rápida — ótima para criptografar grandes volumes de dados.

### Criptografia Assimétrica

**Duas chaves** — uma pública, uma privada. O que uma criptografa, apenas a outra pode descriptografar.

```
Chave pública  🔓  → você compartilha com todos
Chave privada  🔑  → fica apenas com você, nunca sai
```

Qualquer um pode criptografar uma mensagem com sua chave pública. Apenas você pode descriptografá-la com sua chave privada.

**Vantagem:** não precisa concordar em nada previamente — a chave pública pode ser distribuída abertamente.

**Problema:** muito mais lenta que a simétrica.

### Por que TLS Usa Ambas

TLS combina o melhor das duas:

```
1. Handshake  →  usa assimétrico
               → resolve "como concordamos em uma chave sem enviá-la pela rede?"
               → Diffie-Hellman faz isso com chaves pública/privada

2. Sessão    →  usa simétrico
               → agora que ambos têm a mesma chave secreta, use-a para tudo
               → muito mais rápido para criptografar dados de verdade
```

**Analogia:** você usa correio seguro *(assimétrico)* para combinar um código secreto com alguém. Uma vez que ambos têm o código, vocês se comunicam por rádio usando esse código *(simétrico)* — muito mais rápido.

| | **Assimétrico** | **Simétrico** |
|---|---|---|
| Quando | Troca de chave (handshake) | Criptografia de dados (sessão) |
| Como | Par de chaves pública/privada | Uma chave compartilhada |
| Por quê | Seguro para trocar segredos | Muito mais rápido |

> Assimétrico resolve o **problema de acordo de chave** com segurança.
> Simétrico resolve o **problema de performance** durante a troca de dados.
> TLS usa assimétrico para inicializar, simétrico para rodar.

---

## O Fluxo Completo do Handshake

```
[TCP connection already open]

Você        →   ClientHello (algorithms, random number)
            ←   ServerHello + Certificate + random number
Verifica certificado ✓
            →   Key exchange (Diffie-Hellman)
            ←   Key exchange (Diffie-Hellman)
Ambos derivam a mesma chave secreta independentemente
            →   Finished (encrypted)
            ←   Finished (encrypted)

[Encrypted tunnel established — HTTP starts flowing]
```

---

## HTTPS = HTTP + TLS

Quando você acessa `https://`, é exatamente isso — HTTP rodando dentro de um túnel TLS.

```
Camadas:
┌─────────────┐
│    HTTP     │  ← sua requisição
├─────────────┤
│    TLS      │  ← criptografia
├─────────────┤
│    TCP      │  ← move bytes, não sabe o que são
├─────────────┤
│     IP      │  ← roteamento
└─────────────┘
```

TCP não sabe que está carregando TLS — para o TCP são apenas bytes.

---

## O Custo em RTTs

RTT (Round Trip Time) = tempo para enviar algo e receber uma resposta.

```
TCP handshake:   1 RTT   ← conexão abre
TLS handshake:   1 RTT   ← túnel estabelecido  (TLS 1.3)
Primeiros dados HTTP: 1 RTT   ← sua requisição + resposta
```

TLS 1.3 (versão atual) reduziu o handshake para **1 RTT**.
Também tem um modo **0-RTT** para clientes que já se conectaram antes — o túnel é estabelecido com o primeiro pacote de dados.

---

## Certificate Authorities (CA)

O certificado só é confiável porque foi assinado por uma CA em que seu browser já confia.

| CA | Notas |
|---|---|
| **Let's Encrypt** | Gratuito, automatizado, amplamente usado para aplicações web |
| **DigiCert** | Pago, usado por grandes empresas |
| **Sectigo** | Pago, comum em sites comerciais |
| **AWS Certificate Manager** | Gratuito para serviços AWS, gerenciado automaticamente |
| **Google Trust Services** | Usado internamente pelos produtos Google |
| **Cloudflare** | Emite certificados automaticamente para sites proxiados |

---

## Notas de Design

- TLS 1.0 e 1.1 estão obsoletos — use TLS 1.2 no mínimo, 1.3 preferencialmente
- Um certificado tem data de validade — certificado expirado = aviso no browser
- Certificados Let's Encrypt expiram em 90 dias mas são renovados automaticamente
- mTLS (mutual TLS) = ambos os lados apresentam certificados, não apenas o servidor

---

## Notas Relacionadas

- [[SecurityAuth]]
- [[TCP]]
