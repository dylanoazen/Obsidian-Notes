# ReactPHP — Cards de Entrevista

Decisões técnicas, perguntas de sênior e pontos de melhoria em projetos com ReactPHP.

Related: [[PHP/ReactPHP]], [[PHP/EventLoop]], [[PHP/Concurrency]]

---

## Decisões Técnicas — "Por que você fez dessa forma?"

---

**Por que ReactPHP e não PHP tradicional (FPM/Apache)?**

> O PHP-FPM descarta todo o estado ao fim de cada request — variáveis, arrays, objetos, tudo é perdido. Para manter estado em memória entre requests sem banco de dados, preciso de um processo que fique vivo. ReactPHP é exatamente isso: um processo de longa duração com event loop. Assim o repositório persiste na memória enquanto o servidor está rodando.

---

**Por que `int` para valores monetários e não `float`?**

> `Float` tem problema de representação binária — `0.1 + 0.2` em ponto flutuante não é exatamente `0.3`. Em dinheiro isso é inaceitável. Usar `int` com centavos (10 reais = 1000) elimina o problema. A alternativa correta para produção seria `bcmath` para aritmética de precisão arbitrária, ou uma classe `Money` com `amount` + `currency`.

---

**Como a transferência é atômica sem usar `MULTI/EXEC` ou transação de banco?**

> O event loop do ReactPHP é single-threaded — só um callback executa por vez. Não existem duas transferências acontecendo ao mesmo tempo no mesmo processo. Por isso debitar antes de creditar é seguro: se `withdraw` lançar `InsufficientFundsException`, o `deposit` nunca acontece e o estado fica consistente. Sem possibilidade de race condition nesse modelo.

---

**Por que exceções de domínio tipadas e não retornar `null` ou `false`?**

> O domínio não sabe nada de HTTP — não conhece status code, não conhece `Response`. Ele só sabe que a operação falhou e por quê. O controller captura a exceção e traduz para HTTP. Essa separação mantém o domínio puro e testável sem precisar subir o servidor.

---

**Por que `Repository` separado e não salvar diretamente no service?**

> Separar o repositório isola a camada de persistência. Hoje o estado é um array em memória. Para tornar o projeto durável, bastaria criar uma `PdoRepository` implementando a mesma interface e trocá-la no entry point — o service não muda nada. É o princípio de inversão de dependência na prática.

---

## Perguntas de Sênior

---

**Como escalar para múltiplos processos/servidores?**

> O modelo atual não escala horizontalmente — cada processo tem seu próprio array em memória. Se subir dois containers, cada um tem estado diferente. Para escalar seria necessário estado compartilhado: Redis ou banco relacional. O `Repository` seria trocado por uma implementação com `Predis` ou `PDO` sem alterar o restante do código.

---

**Por que ReactPHP e não Swoole ou RoadRunner?**

> Simplicidade e menos dependências. Swoole é mais performático e tem corrotinas nativas, mas exige extensão C compilada. RoadRunner é uma boa opção para produção, mas adiciona complexidade de configuração. ReactPHP resolve o problema com uma única dependência via `composer require`.

---

**Como tornaria o estado durável (persistente entre restarts)?**

> Trocaria a implementação do `Repository`. Criaria uma `PdoRepository` com os mesmos métodos, mas usando banco. O service e o controller não saberiam da diferença — injetaria a nova implementação no entry point. Como solução intermediária, um snapshot em JSON no disco a cada escrita também funcionaria.

---

**Por que não testou o controller?**

> A lógica de negócio está no domínio — entidade e service são testados unitariamente. Testar o controller exigiria mockar a interface do ReactPHP sem agregar cobertura real de lógica. O valor está em testar onde as regras residem. Um teste de integração com cliente HTTP real contra o servidor no ar faria mais sentido.

---

## O Que Poderia Ter Sido Melhor

---

**`setUp()` no PHPUnit para evitar repetição**

Cada teste repete `new Repository()` e `new Service()`. O correto:

```php
protected function setUp(): void
{
    $this->repository = new AccountRepository();
    $this->service    = new AccountService($this->repository);
}
```

---

**Nome de arquivo de teste não espelha a classe testada**

Convenção do PHPUnit: o nome da classe de teste espelha a classe sendo testada. `AccountServiceTest.php` testa `AccountService`, não outro nome.

---

**`catch` inconsistente — FQCN vs `use`**

```php
// correto — importa no topo, usa nome curto no catch
use App\Domain\Exception\InsufficientFundsException;
catch (InsufficientFundsException $e) { ... }
```

---

**`json_decode` sem `JSON_THROW_ON_ERROR`**

```php
// correto — lança exceção, tratada explicitamente
try {
    $data = json_decode($body, true, 512, JSON_THROW_ON_ERROR);
} catch (\JsonException) {
    return $this->plain(400, '0');
}
```

---

**Validação de campos inconsistente**

```php
// correto — uniforme com isset
if (!isset($data['origin'], $data['destination'], $data['amount'])) { ... }
```

---

## Frase de Encerramento

> "O projeto aplica as ideias certas na medida certa para o problema: domínio puro e testável, repositório isolado para facilitar extensão, ReactPHP para o estado em memória sem banco. As simplificações foram justificadas pelo tamanho do spec — em produção eu aumentaria a separação conforme a complexidade crescesse."

---

#### My commentaries
-
