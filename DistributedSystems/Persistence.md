# Persistence

Persistence é a capacidade de manter os dados vivos após o processo parar. Sem ela, tudo armazenado em memória é perdido quando o servidor reinicia.

Existem duas estratégias principais:

---

## RDB — Snapshot

RDB (Redis Database) tira um **snapshot completo** de todos os dados em memória e o salva no disco em um determinado momento.

```
memory ──► [snapshot] ──► dump.rdb (file on disk)
```

**Como funciona:**
- Em um intervalo definido (ou manualmente), o Go faz um fork do processo
- O processo filho escreve uma cópia completa dos dados em um arquivo `.rdb`
- O processo principal continua rodando sem interrupção

**Exemplo de configuração:**
```
save 900 1     # snapshot if at least 1 key changed in 900s
save 300 10    # snapshot if at least 10 keys changed in 300s
save 60 10000  # snapshot if at least 10000 keys changed in 60s
```

**Prós:**
- Arquivo único compacto — fácil de fazer backup e transferir
- Reinicializações rápidas — carregar um arquivo é rápido
- Sem impacto na performance durante operação normal

**Contras:**
- Dados entre snapshots são perdidos em caso de crash
- Se o servidor travar às 4:59 e o snapshot roda a cada 5 minutos, você perde ~5 minutos de dados

**Use quando:** você consegue tolerar alguma perda de dados e quer reinicializações rápidas.

---

## AOF — Append-Only File

AOF registra **toda operação de escrita** conforme ela acontece, adicionando-a a um arquivo de log. Na reinicialização, reproduz o log para reconstruir o estado.

```
SET name Dylan  ──► appended to file
SET age 25      ──► appended to file
DEL name        ──► appended to file

restart ──► replay all entries ──► state restored
```

**Como funciona:**
- Todo comando de escrita é adicionado ao `appendonly.aof`
- Na reinicialização, o arquivo é lido linha por linha e cada comando é re-executado
- Com o tempo o arquivo cresce — uma reescrita (compactação) pode reduzi-lo

**Políticas de Fsync:**
```
appendfsync always    # write to disk on every command (safest, slowest)
appendfsync everysec  # write to disk every second (good balance)
appendfsync no        # let the OS decide (fastest, least safe)
```

**Prós:**
- No máximo 1 segundo de perda de dados (com `everysec`)
- Log legível por humanos — você pode inspecionar ou editá-lo
- Mais durável que RDB

**Contras:**
- Tamanho de arquivo maior que RDB
- Reinicializações mais lentas — precisa reproduzir todos os comandos
- Processo de reescrita necessário para manter o tamanho do arquivo gerenciável

**Use quando:** durabilidade dos dados é crítica e você não pode perder mais de 1 segundo de dados.

---

## RDB vs AOF

| | RDB | AOF |
|---|---|---|
| O que salva | Snapshot completo | Todo comando de escrita |
| Risco de perda de dados | Minutos (entre snapshots) | No máximo 1 segundo |
| Tamanho do arquivo | Pequeno | Grande (cresce com o tempo) |
| Velocidade de reinicialização | Rápida | Lenta (reproduz todos os comandos) |
| Melhor para | Backups, reinicializações rápidas | Durabilidade, log de auditoria |

---

## Usando Ambos Juntos

RDB e AOF podem ser usados simultaneamente — RDB para reinicializações rápidas e AOF para durabilidade. Na reinicialização, o AOF tem prioridade pois contém dados mais recentes.

```
RDB  ──► restauração rápida do snapshot base
AOF  ──► reproduz apenas os comandos desde o último snapshot
```

---

## Sem Persistence

Se nenhum estiver habilitado, os dados vivem apenas em memória — tudo é perdido na reinicialização. Útil para cache puro onde os dados podem ser reconstruídos a partir da fonte.

```
restart ──► empty state
```

---

## Conceito de Implementação em Go

```go
// RDB — save snapshot
func saveRDB(store map[string]string) error {
    data, _ := json.Marshal(store)
    return os.WriteFile("dump.rdb", data, 0644)
}

// RDB — load snapshot
func loadRDB() map[string]string {
    data, err := os.ReadFile("dump.rdb")
    if err != nil {
        return map[string]string{}
    }
    store := map[string]string{}
    json.Unmarshal(data, &store)
    return store
}

// AOF — append command
func appendAOF(command string) error {
    f, err := os.OpenFile("appendonly.aof", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()
    _, err = f.WriteString(command + "\n")
    return err
}
```
