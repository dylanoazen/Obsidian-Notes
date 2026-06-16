# EBANX — Preparação para Entrevista

Cards de estudo: decisões técnicas, perguntas de sênior e o que poderia ter sido melhor.

Related: [[PHP/EBANX]], [[PHP/ReactPHP]], [[PHP/EventLoop]]

---

## Decisões Técnicas — "Por que você fez dessa forma?"

---

**Por que ReactPHP e não PHP tradicional (FPM/Apache)?**

> O PHP-FPM descarta todo o estado ao fim de cada request — variáveis, arrays, objetos, tudo é perdido. Para manter estado em memória entre requests sem banco de dados, preciso de um processo que fique vivo. ReactPHP é exatamente isso: um processo de longa duração com event loop. Assim o `AccountRepository` persiste na memória enquanto o servidor está rodando.

---

**Por que `int` para valores monetários e não `float`?**

> `Float` tem problema de representação binária — `0.1 + 0.2` em ponto flutuante não é exatamente `0.3`. Em dinheiro isso é inaceitável. Usar `int` com centavos (10 reais = 1000) elimina o problema. A alternativa correta para produção seria `bcmath` para aritmética de precisão arbitrária, ou uma classe `Money` com `amount` + `currency`.

---

**Como a transferência é atômica sem usar `MULTI/EXEC` ou transação de banco?**

> O event loop do ReactPHP é single-threaded — só um callback executa por vez. Não existem duas transferências acontecendo ao mesmo tempo no mesmo processo. Por isso debitar antes de creditar é seguro: se `withdraw` lançar `InsufficientFundsException`, o `deposit` nunca acontece e o estado fica consistente. Sem possibilidade de race condition nesse modelo.

---

**Por que exceções de domínio tipadas (`InsufficientFundsException`, `AccountNotFoundException`) e não retornar `null` ou `false`?**

> O domínio não sabe nada de HTTP — não conhece status code, não conhece `Response`. Ele só sabe que a operação falhou e por quê. O controller captura a exceção e traduz para HTTP: `InsufficientFundsException` → 404, `InvalidArgumentException` → 400. Essa separação mantém o domínio puro e testável sem precisar subir o servidor.

---

**Por que `AccountRepository` separado e não salvar diretamente no service?**

> Separar o repositório isola a camada de persistência. Hoje o estado é um array em memória. Para tornar o projeto durável, bastaria criar uma `PdoAccountRepository` implementando a mesma interface e trocá-la no `server.php` — o `EventService` não muda nada. É o princípio de inversão de dependência na prática.

---

## Perguntas de Sênior — "E se eu perguntar..."

---

**Como escalar para múltiplos processos/servidores?**

> O modelo atual não escala horizontalmente — cada processo tem seu próprio array em memória. Se subir dois containers, cada um tem estado diferente. Para escalar seria necessário estado compartilhado: Redis (mais natural para cache em memória) ou banco relacional. O `AccountRepository` seria trocado por uma implementação com `Predis` ou `PDO` sem alterar o restante do código.

---

**Por que ReactPHP e não Swoole ou RoadRunner?**

> Simplicidade e menos dependências. Swoole é mais performático e tem corrotinas nativas, mas exige extensão C compilada — complexidade desnecessária para um take-home. RoadRunner é uma boa opção para produção, mas adiciona complexidade de configuração. ReactPHP resolve o problema com uma única dependência via `composer require`.

---

**Por que `EventService` com um método `handle` e não casos de uso separados (`DepositUseCase`, `WithdrawUseCase`)?**

> O spec é pequeno — três operações, uma entidade. Casos de uso separados adicionariam três arquivos e três classes para o mesmo resultado. Se o projeto crescesse (novas operações, validações mais complexas, eventos de domínio), eu separaria. Para o que foi pedido, seria uma complexidade desnecessária que prejudica a legibilidade.

---

**Como tornaria o estado durável (persistente entre restarts)?**

> Trocaria a implementação do `AccountRepository`. Criaria uma `PdoAccountRepository` com os mesmos métodos `find`, `save`, `reset`, mas usando banco. O `EventService` e o `EventController` não saberiam da diferença — injetaria a nova implementação no `server.php`. Como solução intermediária, um snapshot em JSON no disco a cada escrita também funcionaria.

---

**Por que não testou o `EventController`?**

> A lógica de negócio está no domínio — `Account` e `EventService` são testados unitariamente. Testar o controller exigiria mockar a interface do ReactPHP (`ServerRequestInterface`, `Response`) sem agregar cobertura real de lógica. O valor está em testar onde as regras residem. Um teste de integração do controller faria mais sentido com um cliente HTTP real contra o servidor no ar.

---

## O Que Poderia Ter Sido Melhor

---

**`setUp()` no PHPUnit para evitar repetição**

Hoje cada teste repete `new AccountRepository()` e `new AccountService()`. O correto seria:

```php
class AccountServiceTest extends TestCase
{
    private AccountRepository $repository;
    private EventService $service;

    protected function setUp(): void
    {
        $this->repository = new AccountRepository();
        $this->service    = new EventService($this->repository);
    }

    public function test_deposit_creates_account(): void
    {
        // $this->service já pronto, sem repetir new
    }
}
```

---

**Nome de arquivo errado — `AccountRepositoryTest` testa `AccountService`**

O arquivo que testa o `EventService` deveria se chamar `EventServiceTest.php` (ou `AccountServiceTest.php` se o service se chamar assim). Convenção do PHPUnit: o nome da classe de teste espelha a classe sendo testada.

---

**Faltou teste do reset**

```php
public function test_reset_clears_all_accounts(): void
{
    $this->service->handle(['type' => 'deposit', 'destination' => '100', 'amount' => 10]);
    $this->repository->reset();

    $this->assertNull($this->repository->find('100'));
}
```

---

**`catch` inconsistente — FQCN vs `use`**

Misturar `use App\Domain\Exception\InsufficientFundsException` no topo com `catch (\App\Domain\Exception\InsufficientFundsException)` no corpo é inconsistente. Escolha um dos dois:

```php
// opção 1 — importa no topo, usa nome curto no catch
use App\Domain\Exception\InsufficientFundsException;
catch (InsufficientFundsException $e) { ... }

// opção 2 — sem import, FQCN completo no catch
catch (\App\Domain\Exception\InsufficientFundsException $e) { ... }
```

---

## Code Review — Alterações que o Sênior Pode Solicitar

---

**"Mova o `AccountService` para `Application`"**

`AccountService` está em `App\Domain`, mas é um orquestrador — chama o repositório e coordena operações. O domínio deveria conter apenas entidades e exceções. A correção é mover para `src/Application/` e ajustar o namespace.

```
src/Domain/AccountService.php      ← está aqui (errado)
src/Application/AccountService.php ← deveria estar aqui
```

> "Concordo — coloquei no Domain por pressa. O correto é Application porque o service orquestra o domínio, não faz parte dele. A mudança é só de pasta e namespace, o código em si não muda."

---

**"Falta o `Loop::run()` no `server.php`"**

O `server.php` registra o socket e imprime a mensagem, mas não chama `Loop::run()` explicitamente. O ReactPHP 1.9+ auto-inicia o loop em alguns contextos, mas a prática correta é sempre chamar explicitamente.

```php
// adicionar no fim do server.php
React\EventLoop\Loop::run();
```

> "Verdade, a omissão foi intencional porque o ReactPHP 1.9 auto-inicia, mas explícito é sempre melhor — comunica a intenção e não depende de comportamento implícito da lib."

---

**"Validação de `amount` está inconsistente"**

No `EventController`, `origin` e `destination` usam `isset`, mas `amount` usa `=== null`:

```php
// atual — inconsistente
if (!isset($data['origin']) || $data['amount'] === null)

// correto — uniforme
if (!isset($data['origin']) || !isset($data['amount']))
// ou usando null coalescing:
if (!isset($data['origin'], $data['amount']))
```

Se a chave `amount` não existir, `$data['amount']` gera um warning no PHP 8 antes de retornar null. Funciona por acidente — `isset` é o correto.

> "Bug real. Devia ter usado `isset` nos três para ser uniforme. A forma mais limpa é `!isset($data['origin'], $data['amount'])` — `isset` aceita múltiplos argumentos e retorna false se qualquer um for null ou não existir."

---

**"Use constructor promotion no `EventController`"**

`Account.php` usa o estilo moderno do PHP 8.1, mas `EventController` ainda usa o estilo antigo:

```php
// atual — estilo antigo
private readonly AccountService $service;
public function __construct(AccountService $service) {
    $this->service = $service;
}

// correto — constructor promotion, consistente com o resto
public function __construct(private readonly AccountService $service) {}
```

> "Inconsistência de estilo. Deveria ter usado promotion igual ao `Account` — menos código, mais legível, e é exatamente pra isso que o PHP 8.1 trouxe o recurso."

---

**"`InsufficientFundsException` retorna 422, o spec pede 404"**

O brief define que operação inválida retorna `0` com status `404`. O controller retorna `422 Unprocessable Entity` para saldo insuficiente, o que é semanticamente mais correto, mas desvia do spec.

```php
// atual
} catch (InsufficientFundsException) {
    return $this->plain(422, '0');

// conforme spec
} catch (InsufficientFundsException) {
    return $this->plain(404, '0');
```

> "Escolhi 422 porque semanticamente é mais correto — a entidade existe mas a operação não pode ser processada. 404 implica que o recurso não foi encontrado, o que não é o caso. Dito isso, se o spec define 404, ajusto — em take-home o avaliador roda testes automatizados e spec é lei."

---

**"`json_decode` sem `JSON_THROW_ON_ERROR`"**

O helper `json()` usa `JSON_THROW_ON_ERROR` no encode, mas não no decode:

```php
// atual — retorna null silenciosamente se JSON for inválido
$data = json_decode((string)$request->getBody(), true);

// correto — lança exceção, tratada explicitamente
try {
    $data = json_decode((string)$request->getBody(), true, 512, JSON_THROW_ON_ERROR);
} catch (\JsonException) {
    return $this->plain(400, '0');
}
```

> "Inconsistência que eu deixei passar. Com `JSON_THROW_ON_ERROR` o erro é explícito e tratado — sem depender de checar `=== null` depois, que também pode ser um JSON válido `null`."

---

## Frase de Encerramento (se pedirem resumo)

> "O projeto aplica as ideias certas na medida certa para o problema: domínio puro e testável, repositório isolado para facilitar extensão, ReactPHP para o estado em memória sem banco. As simplificações que fiz (EventService único, sem casos de uso separados) são justificadas pelo tamanho do spec — em produção eu aumentaria a separação conforme a complexidade crescesse."

---

#### My commentaries
-
