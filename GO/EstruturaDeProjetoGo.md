---
tags: [go, arquitetura, project-layout]
---

# Estrutura de Projeto em Go: `cmd/` e `internal/`

> [!info] Fundamento que segura tudo
> Em Go, **pasta = pacote**. Toda pasta é um pacote, todo arquivo `.go` dentro
> dela declara o mesmo `package`, e o `import` traz a **pasta inteira**, nunca um
> arquivo isolado. Diferente do PHP, onde namespace é declarado por arquivo e o
> PSR-4 só mapeia por convenção (dá pra burlar), aqui o **compilador obriga**.
> É isso que faz o *pacote* ser a unidade real de encapsulamento da linguagem.

## `cmd/` — os executáveis

- Onde moram os `package main` (os binários). É a **ponta**: onde o programa
  começa e onde tudo é montado/injetado.
- Regra mental: **`cmd/` não tem lógica de negócio, só ligação** (composition root).
- Vários binários convivem aqui, cada um numa subpasta, reusando o mesmo `internal/`:
  - `cmd/tray-sync/` → serviço principal
  - `cmd/tray-backfill/` → (hipotético) recarga histórica
- Análogo PHP: o `public/index.php` / o entrypoint do console — o lugar que só amarra.

## `internal/` — privacidade imposta pelo compilador

- Regra **da linguagem**, não convenção: nada de fora do módulo pode importar o
  que está sob `internal/`. O compilador **recusa**.
- É encapsulamento de verdade: as tripas da aplicação ficam aqui e ninguém
  externo consegue se acoplar nelas.
- Não tem equivalente direto em PHP — o mais perto é "classe interna por
  disciplina", mas aqui é garantido, não combinado.

## Como isso se combina

```
projeto/
├── go.mod                  # raiz do módulo (≈ composer.json + raiz de namespace)
├── cmd/<bin>/main.go       # ponta: só monta e liga
└── internal/               # tripas privadas ao módulo
    └── <pacotes>/          # o split aqui dentro é escolha de arquitetura
```

> [!warning] O split DENTRO de `internal/` não é regra do Go
> Separar em `domain/`, `worker/`, etc. é decisão de arquitetura (ex.: Hexagonal).
> O Go não liga; quem dá sentido a essa divisão é o projeto.

## Pra fixar (pergunta de revisão)

- Por que `import` de um arquivo específico não existe em Go? → porque a unidade é o pacote/pasta.
- O que `internal/` garante que uma convenção de PHP não garante? → proibição imposta pelo compilador.
- O que pode (e não pode) morar em `cmd/`? → só ligação; nada de regra de negócio.

---
Relacionado: [[GO/GO]] · [[GO/Packages]] · [[GO/GO]] · [[Arquitetura Hexagonal]] · [[Ports e Adapters]]
