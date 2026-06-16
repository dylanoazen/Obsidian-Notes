## Go Modules

Um módulo é a raiz versionada de um projeto Go.

Ele define:
- a identidade do projeto
- gerenciamento de dependências
- versionamento
- o namespace de import raiz

Dentro de um módulo, o código é organizado em packages.

### Module
- Raiz do projeto
- Unidade versionada
- Limite de dependências
- Namespace de import

### Packages
- Organização lógica do código
- Grupo de arquivos relacionados
- Divisões internas do projeto


### Packages
- [[GO/Packages/Net]]
- [[GO/Packages/Bufio]]

### Exported Identifiers

- Funções com nomes em maiúsculo são exportadas (públicas) e podem ser importadas por outros packages.

## Topics

- [[GO/Semantics]]
- [[GO/Structs]]
- [[GO/Methods]]
- [[GO/Keywords]]
- [[GO/MemoryManagement]]
- [[GO/Mutex]]
- [[GO/GarbageCollector]]
- [[GO/Context]]
- [[GO/EstruturaDeProjetoGo]]

#### Meus comentários
- O principal motivo para isso é, por exemplo, evitar quebrar uma versão mais antiga de um projeto que usa o projeto Go por causa de incompatibilidade da nova versão


