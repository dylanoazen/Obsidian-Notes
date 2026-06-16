# Security & Auth

Segurança em sistemas distribuídos trata de duas perguntas fundamentais:

- **Quem é você?** → Authentication
- **O que você pode fazer?** → Authorization

---

## Authentication

Authentication é o ato de provar sua identidade antes de ser permitido entrar.

As formas mais comuns:

| Método | Como funciona |
|---|---|
| Senha | Você conhece uma string secreta |
| Token (JWT) | Você possui um token assinado emitido pelo servidor |
| API Key | Você possui uma chave vinculada à sua identidade |
| SSH Key | Você possui uma chave privada que corresponde a uma chave pública no servidor |
| Certificate (mTLS) | Sua máquina possui um certificado criptográfico |

**Analogia:** o segurança na porta — você só entra se puder provar quem é.

Authentication responde: **"você é quem diz ser?"**

### SSH Keys

SSH keys funcionam como um par de chaves:

```
Private key  ← fica na sua máquina, nunca sai
Public key   ← você dá ao servidor antecipadamente
```

Quando você tenta se conectar:

```
Você        →  "Quero entrar"
Servidor    →  envia um desafio criptografado
Você        →  decifra com sua chave privada e responde
Servidor    →  somente o dono da chave privada poderia fazer isso → permitido ✓
```

Você nunca envia uma senha pela rede — prova sua identidade resolvendo um desafio criptográfico.

**Analogia:** um cadeado com duas chaves. Você dá o cadeado aberto ao servidor. Apenas você tem a chave para abrí-lo. Se você conseguir abrí-lo, provou que é você.

| | **Senha** | **SSH Key** |
|---|---|---|
| O que trafega pela rede | A senha (mesmo que criptografada) | Nada sensível — apenas a prova |
| Pode ser roubada remotamente | Sim | Não — a chave privada nunca sai da sua máquina |
| Força bruta possível | Sim | Praticamente impossível |

---

## Authorization

Authorization acontece após authentication — ela define o que uma identidade autenticada está autorizada a fazer.

Você pode estar autenticado (ter provado quem é) e ainda não estar autorizado a fazer certas coisas.

**Analogia:** você tem um crachá para entrar no prédio *(authentication)*, mas seu crachá só abre a porta principal e o seu andar — não a sala de servidores *(authorization)*.

Authorization responde: **"você tem permissão para fazer isso?"**

---

## ACL (Access Control List)

Uma ACL é a forma mais comum de implementar authorization — uma lista que mapeia identidades a permissões.

```
user: analytics  → pode ler chaves user:*
user: api        → pode ler e escrever chaves user:*
user: admin      → pode fazer tudo
```

Cada entrada diz: *esta identidade pode fazer essas ações nesses recursos.*

**Princípio do menor privilégio:** dê a cada usuário o acesso mínimo necessário para fazer seu trabalho — nada mais. Se um serviço for comprometido, o raio de impacto é contido.

---

## Roles

Em vez de atribuir permissões a cada usuário individualmente, você define **roles** — conjuntos nomeados de permissões — e atribui roles aos usuários.

```
Role: readonly   → GET, LIST
Role: readwrite  → GET, LIST, SET, DELETE
Role: admin      → everything

User: analytics  → role: readonly
User: api        → role: readwrite
User: dylan      → role: admin
```

Isso facilita gerenciar permissões em escala — mude a role, todos os usuários com aquela role são atualizados.

**Analogia:** cargos em uma empresa. Um estagiário tem acesso de estagiário. Um gerente tem acesso de gerente. Você define o que cada cargo pode fazer, depois atribui cargos às pessoas.

---

## Authentication vs Authorization

Uma fonte comum de confusão:

| | **Authentication** | **Authorization** |
|---|---|---|
| Pergunta | Quem é você? | O que você pode fazer? |
| Acontece | Primeiro | Após authentication |
| Exemplo | Login com senha | Você pode acessar este endpoint? |
| Se falhar | "Não te conheço" | "Te conheço, mas você não pode fazer isso" |

Você não pode ter authorization sem authentication — você precisa saber quem alguém é antes de decidir o que pode fazer.

---

## TLS — Criptografando a Conexão

Authentication prova quem você é. TLS criptografa o que trafega entre cliente e servidor.

Sem TLS, senhas e dados trafegam em texto plano — qualquer roteador no caminho pode ler tudo. Com TLS, um túnel criptografado é criado e qualquer um que intercepte o tráfego vê apenas ruído.

> Veja [[TLS]] para o detalhamento completo — handshake, troca de chaves Diffie-Hellman, certificados e como se sobrepõe ao TCP.

---

a## No Redis

Redis implementa esses conceitos nativamente:

- **Authentication** → `AUTH password` ou username + password
- **Authorization** → sistema ACL com permissões de comandos e chaves por usuário
- **TLS** → conexões criptografadas entre clientes e servidor

```bash
# create a user with limited access
ACL SETUSER analytics on >secret ~user:* +GET

# user can only GET keys starting with user:
# cannot SET, DEL, FLUSHALL, or access other keys
```

Redis não tem roles nomeadas integradas — você as simula criando usuários com conjuntos de permissões equivalentes.

---

## Notas de Design

- Sempre autentique em produção — nunca deixe sistemas abertos
- Use roles para gerenciar permissões em escala em vez de regras por usuário
- O princípio do menor privilégio reduz os danos quando algo é comprometido
- TLS é obrigatório quando o tráfego cruza uma rede que você não controla completamente
- Audite regularmente quem tem acesso — remova o que não é mais necessário

---

## Notas Relacionadas

- [[Replication]]
- [[Kubernetes]]
- [[TransactionsPipeline]]
