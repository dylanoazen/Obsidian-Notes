# EBANX — Prep Entrevista

Cards de estudo: decisões técnicas, perguntas de sênior e o que poderia ter sido melhor.

Related: [[PHP/EBANX]], [[PHP/ReactPHP]], [[PHP/EventLoop]]

---

## Decisões Técnicas — "Por que você fez assim?"

---

**Por que ReactPHP e não PHP tradicional (FPM/Apache)?**

> PHP-FPM descarta todo o estado ao fim de cada request — variáveis, arrays, objetos, tudo some. Para manter estado em memória entre requests sem banco de dados, preciso de um processo que fique vivo. ReactPHP é exatamente isso: um processo de longa duração com event loop. Assim o `AccountRepository` persiste na memória enquanto o servidor está rodando.

---

**Por que `int` para valores monetários e não `float`?**

> Float tem problema de representação binária — `0.1 + 0.2` em ponto flutuante não é exatamente `0.3`. Em dinheiro isso é inaceitável. Usar `int` com centavos (10 reais = 1000) elimina o problema. A alternativa correta para produção seria `bcmath` para aritmética de precisão arbitrária, ou uma classe `Money` com `amount` + `currency`.

---

**Como a transferência é atômica sem usar `MULTI/EXEC` ou transação de banco?**

> O event loop do ReactPHP é single-threaded — só um callback executa por vez. Não tem duas transferências acontecendo ao mesmo tempo no mesmo processo. Por isso debitar antes de creditar é seguro: se `withdraw` lançar `InsufficientFundsException`, o `deposit` nunca acontece e o estado fica consistente. Sem race condition possível nesse modelo.

---

**Por que exceções de domínio tipadas (`InsufficientFundsException`, `AccountNotFoundException`) e não retornar `null` ou `false`?**

> O domínio não sabe nada de HTTP — ele não conhece status code, não conhece `Response`. Ele só sabe que a operação falhou e por quê. O controller captura a exceção e traduz para HTTP: `InsufficientFundsException` → 404, `InvalidArgumentException` → 400. Essa separação mantém o domínio puro e testável sem subir o servidor.

---

**Por que `AccountRepository` separado e não salvar direto no service?**

> Separar o repositório isola a camada de persistência. Hoje o estado é um array em memória. Para tornar o projeto durável, bastaria criar uma `PdoAccountRepository` implementando a mesma interface e trocar no `server.php` — o `EventService` não muda nada. É o princípio de inversão de dependência na prática.

---

## Perguntas de Sênior — "E se eu te perguntar..."

---

**Como escalar para múltiplos processos/servidores?**

> O modelo atual não escala horizontalmente — cada processo tem o seu próprio array em memória. Se subir dois containers, cada um tem estado diferente. Para escalar precisaria de estado compartilhado: Redis (mais natural para cache em memória) ou banco relacional. O `AccountRepository` seria trocado por uma implementação com `Predis` ou `PDO` sem tocar no resto do código.

---

**Por que ReactPHP e não Swoole ou RoadRunner?**

> Simplicidade e menos dependências. Swoole é mais performático e tem corrotinas nativas, mas exige extensão C compilada — over-engineering para um take-home. RoadRunner é uma boa opção para produção mas adiciona complexidade de configuração. ReactPHP resolve o problema com uma dependência `composer require` simples.

---

**Por que `EventService` com um método `handle` e não casos de uso separados (`DepositUseCase`, `WithdrawUseCase`)?**

> O spec é pequeno — três operações, uma entidade. Casos de uso separados adicionariam três arquivos e três classes para o mesmo resultado. Se o projeto crescesse (novas operações, validações mais complexas, eventos de domínio), separaria. Para o que foi pedido, é over-engineering que prejudica legibilidade.

---

**Como tornaria o estado durável (persistente entre restarts)?**

> Trocar a implementação do `AccountRepository`. Criaria uma `PdoAccountRepository` com os mesmos métodos `find`, `save`, `reset` mas usando banco. O `EventService` e o `EventController` não saberiam da diferença — injetaria a nova implementação no `server.php`. Se quiser intermediário, snapshot em JSON no disco a cada write também funciona.

---

**Por que não testou o `EventController`?**

> A lógica de negócio está no domínio — `Account` e `EventService` são testados unitariamente. Testar o controller exigiria mockar a interface do ReactPHP (`ServerRequestInterface`, `Response`) sem agregar cobertura real de lógica. O valor está em testar onde as regras vivem. Teste de integração do controller faria mais sentido com um cliente HTTP real contra o servidor subido.

---

## O Que Poderia Ter Feito Melhor

---

**`setUp()` no PHPUnit para evitar repetição**

Hoje cada teste repete `new AccountRepository()` e `new AccountService()`. O certo:

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

**Nome do arquivo errado — `AccountRepositoryTest` testa `AccountService`**

O arquivo que testa o `EventService` deveria se chamar `EventServiceTest.php` (ou `AccountServiceTest.php` se o service se chamar assim). Convenção PHPUnit: o nome do test class espelha a classe sendo testada.

---

**Faltou teste de reset**

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

Misturar `use App\Domain\Exception\InsufficientFundsException` no topo com `catch (\App\Domain\Exception\InsufficientFundsException)` no corpo é inconsistente. Escolhe um:

```php
// opção 1 — importa no topo, usa nome curto no catch
use App\Domain\Exception\InsufficientFundsException;
catch (InsufficientFundsException $e) { ... }

// opção 2 — sem import, FQCN completo no catch
catch (\App\Domain\Exception\InsufficientFundsException $e) { ... }
```

---

## Frase de Fechamento (se pedirem resumo)

> "O projeto aplica as ideias certas na dose certa para o problema: domínio puro e testável, repositório isolado para facilitar extensão, ReactPHP para o estado em memória sem banco. As simplificações que fiz (EventService único, sem casos de uso separados) são justificadas pelo tamanho do spec — em produção eu escalaria a separação conforme a complexidade crescesse."

---

#### My commentaries
-
