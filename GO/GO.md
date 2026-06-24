## Go Modules

A module is the versioned root of a Go project.

It defines:
- the project's identity
- dependency management
- versioning
- the root import namespace

Inside a module, the code is organized into packages.

### Module
- Project root
- Versioned unit
- Dependency boundary
- Import namespace

### Packages
- Logical code organization
- Group of related files
- Internal project divisions


### Packages
- [[GO/Packages/Net]]
- [[GO/Packages/Bufio]]

### Exported Identifiers

- Functions with uppercase names are exported (public) and can be imported from other packages.

## Topics

- [[GO/Semantics]]
- [[GO/Structs]]
- [[GO/Methods]]
- [[GO/Keywords]]
- [[GO/MemoryManagement]]
- [[GO/Mutex]]
- [[GO/GarbageCollector]]
- [[GO/EstruturaDeProjetoGo]]

#### My commentaries
- The main reason for doing this is, for example, to avoid breaking an older version of a project using the Go project due to incompatibility of the new version


