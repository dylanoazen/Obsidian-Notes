# Replication

Replication é o processo de manter dados sincronizados entre múltiplos nós. Um nó primary recebe escritas e uma ou mais réplicas copiam os dados.

---

## Termos Principais

- **Primary (master):** nó que aceita escritas.
- **Replica:** nó que recebe uma cópia dos dados do primary.
- **Replication stream:** sequência ordenada de mudanças enviadas do primary para as réplicas.
- **Offset:** posição atual no replication stream.
- **Backlog:** buffer de mudanças recentes mantido pelo primary.

---

## Offsets

Um **offset** é a posição no replication stream — um único inteiro que cresce monotonicamente conforme os comandos são processados.

- O primary mantém um offset atual (avança a cada escrita)
- Cada réplica rastreia seu próprio offset (avança conforme aplica comandos)
- Se uma réplica se reconecta, ela envia seu último offset conhecido para o primary

O primary então compara os offsets:
- Gap é pequeno e ainda está no backlog → **partial resync** (envia apenas os comandos faltantes)
- Gap é muito grande ou offset não encontrado → **full resync** (envia um snapshot RDB completo)

Isso permite recuperação incremental em vez de sempre copiar tudo do zero.

### Como o offset funciona passo a passo

```
Primary offset: 100

  [write: SET name "João"]  → primary offset becomes 101
  [write: SET age 25]       → primary offset becomes 102
  [write: SET name "Maria"] → primary offset becomes 103

Réplica estava no offset 101 e se reconecta:
  → pergunta: "me dê tudo depois do 101"
  → primary envia: comandos em 102 e 103
  → réplica os aplica, agora também em offset 103 ✓
```

### Por que offset e não timestamp?

Timestamps têm problemas em sistemas distribuídos — os relógios divergem entre máquinas. Um offset é apenas um contador: determinístico, sem ambiguidade e sempre crescente. Não há discordância sobre o que "103" significa.

---

## Backlog

Um **backlog** é um buffer de mudanças recentes mantido pelo primary.

- Armazena os últimos N bytes/comandos do replication stream
- Permite que réplicas se recuperem após uma breve desconexão
- Se a réplica estiver muito atrás, um full resync é necessário

O tamanho do backlog é um trade-off: memória vs velocidade de recuperação.

---

## Full Resync vs Partial Resync

- **Partial resync:** a réplica requisita os dados faltantes a partir do seu offset
- **Full resync:** a réplica descarta o estado e copia um snapshot completo

Partial resync é mais rápido e mais barato, mas só funciona se o backlog ainda contém os dados faltantes.

---

## Fluxo Mínimo (Conceito)

1) Réplica se conecta ao primary
2) Réplica envia seu último offset conhecido
3) Primary decide:
   - Se o backlog cobre o gap -> envia os dados faltantes (partial resync)
   - Caso contrário -> envia snapshot completo (full resync)
4) Réplica aplica as mudanças e atualiza seu offset

---

## Cenários de Falha

- **Queda de rede:** réplica se reconecta usando seu último offset
- **Longa indisponibilidade:** backlog muito pequeno -> full resync
- **Lag:** réplica atrasada -> leituras podem estar desatualizadas

## Failover

Failover é quando o primary morre e uma réplica automaticamente ocupa seu lugar.

```
Antes:
Primary (alive)  →  Replica

Primary morre ↓

Depois:
Primary (dead)      Replica → promovida a novo Primary
```

### Como funciona

1. Primary para de responder
2. O sistema detecta que está fora do ar
3. A réplica com menor lag é **promovida** a primary
4. A aplicação começa a escrever nela
5. Quando o antigo primary volta, ele se torna uma réplica do novo

### Sem failover vs com failover

| | **Sem failover** | **Com failover** |
|---|---|---|
| Primary morre | App para até alguém consertar manualmente | App se recupera em segundos, automaticamente |
| Requer humano? | Sim | Não |

### Quem gerencia failover no Redis

- **Redis Sentinel** — monitora o primary, dispara failover, notifica a aplicação do novo endereço do primary
- **Redis Cluster** — failover integrado como parte do protocolo de cluster

> Replication é o mecanismo. Failover é o que replication torna possível.

---

## Quando Usar Replication

Replication faz sentido quando:

- **Perda de dados tem custo real** — carrinhos de compra, filas, estado da aplicação que não pode ser trivialmente reconstruído
- **Throughput de leitura precisa escalar** — distribua leituras entre réplicas, mantenha escritas no primary
- **Disponibilidade importa** — se o primary cair, uma réplica assume e o sistema continua funcionando

Uma única instância Redis é suficiente quando os dados são puramente um cache descartável — se morrer, você reconstrói a partir da fonte da verdade em outro lugar.

> Configure replication quando o custo de perder os dados supera o custo de rodar duas instâncias.

## Replication vs Horizontal Scaling

Frequentemente confundidos, mas resolvem problemas diferentes:

| | **Replication** | **Horizontal Scaling** |
|---|---|---|
| O que é copiado | Dados (estado) | A aplicação (lógica) |
| Exemplo | Redis primary + réplicas | 10 containers do seu backend |
| Por quê | Sobrevivência + escala de leitura | Throughput + disponibilidade |
| Precisa de sincronização? | Sim — réplicas precisam ficar em sincronia | Não — cada instância é independente |

Seu **backend** (a aplicação) você escala adicionando mais instâncias — sem replication necessária, porque é **stateless**: não mantém estado, apenas processa requisições. Qualquer instância pode lidar com qualquer requisição.

Seu **Redis** você replica porque é **stateful**: ele *é* o estado. Se morrer sem uma réplica, os dados se foram.

> Coisas stateless você escala. Coisas stateful você replica.

### Horizontal Scaling na Prática

Quando seu backend está sob carga pesada, você o escala lançando mais instâncias — mais containers, mais processos, mais máquinas. Isso funciona porque o backend é **stateless**: cada requisição carrega tudo que precisa (token, body, params), então qualquer instância pode lidar com qualquer requisição.

```
Antes:
[Container 1 - Backend]  ← sobrecarregado

Depois:
[Container 1 - Backend]
[Container 2 - Backend]  ← mesma aplicação, mais instâncias
[Container 3 - Backend]
         ↑
    Load Balancer distribui requisições entre eles
```

O estado vive **fora** dos containers:

```
[Container 1]  ↘
[Container 2]  →  Redis (sessions, cache)  +  PostgreSQL (data)
[Container 3]  ↗
```

Todos os containers leem e escrevem no mesmo Redis e no mesmo banco de dados. Nenhum deles armazena nada localmente.

É exatamente por isso que replication do Redis importa em escala — se 3 containers dependem de um único Redis e ele morre, os 3 param. Replication garante que o Redis sobreviva.

> Você escala o backend horizontalmente porque é stateless.
> Você replica o Redis porque ele carrega o estado de todos eles.

## Notas de Design

- Offsets são posições de sequência, não timestamps
- Replication costuma ser assíncrona; réplicas podem ficar atrás
- Se o backlog for menor que a taxa de escrita durante a janela de desconexão, full resync é inevitável
- Esses conceitos (offset, backlog, partial/full resync) só importam quando algo quebra — no caminho feliz, o primary apenas transmite comandos para as réplicas continuamente e os offsets ficam em sincronia

---

## Esboço Mínimo do Protocolo

- Replica -> Primary: `PSYNC <replica_id> <offset>`
- Primary -> Replica: `+CONTINUE <offset>` ou `+FULLRESYNC <offset>`
- Primary -> Replica: stream de escritas (cada escrita avança o offset)
---

## Notas Relacionadas

- [[Persistence]] — RDB, AOF, e como se conectam com replicação
- [[Concurrency]]
- [[TCP]]
