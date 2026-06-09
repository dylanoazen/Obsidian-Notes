# Testing em PHP

PHPUnit, PEST, test doubles e como testar código de verdade — não testar mocks.

Related: [[PHP/PHP]], [[PHP/SOLID]]

---

## Tipos de Teste

```
                Velocidade    Confiança    Custo
Unit test       ████████░░    ██░░░░░░    Baixo
Integration     █████░░░░░    █████░░░    Médio
E2E / Feature   ██░░░░░░░░    ████████    Alto
```

- **Unit**: testa uma classe/função isolada
- **Integration**: testa componentes juntos (service + repository)
- **E2E**: testa o sistema inteiro (HTTP request → response)

## Arrange-Act-Assert (AAA)

Todo teste segue esse padrão:

```php
public function test_transfer_debits_sender(): void {
    // ARRANGE — monta o cenário
    $repo = new InMemoryAccountRepository();
    $repo->save(new Account('A', 1000));
    $repo->save(new Account('B', 500));
    $service = new AccountService($repo);

    // ACT — executa a ação
    $service->transfer('A', 'B', 300);

    // ASSERT — verifica o resultado
    $this->assertEquals(700, $repo->findById('A')->balance);
    $this->assertEquals(800, $repo->findById('B')->balance);
}
```

## Testar Estado Real vs Mocks

```php
// ✅ Testa estado real — alta confiança
$service->transfer('A', 'B', 300);
$this->assertEquals(700, $repo->findById('A')->balance);
// Verifica que o saldo REALMENTE mudou

// ⚠️ Testa com mock — testa o mock, não o código
$repo = Mockery::mock(AccountRepositoryInterface::class);
$repo->shouldReceive('save')->twice();
$service->transfer('A', 'B', 300);
// Verifica que save() foi chamado... mas o saldo tá certo? 🤷
```

> "Mock demais testa o mock, não o código. Eu prefiro usar um repositório fake in-memory e verificar o estado final — isso me dá confiança de que a lógica realmente funciona."

## Test Doubles

| Tipo | O que faz | Quando usar |
|------|----------|-------------|
| **Fake** | Implementação simplificada funcional | InMemoryRepo, SQLite in-memory |
| **Stub** | Retorna valores pré-definidos | External API responses |
| **Mock** | Verifica se métodos foram chamados | Testar interações (email enviado?) |
| **Spy** | Grava chamadas pra verificar depois | Logging, event dispatch |
| **Dummy** | Preenche parâmetros, não é usado | Satisfazer type hints |

```php
// Fake — implementação real, mas simplificada
class InMemoryAccountRepository implements AccountRepositoryInterface {
    private array $accounts = [];
    
    public function save(Account $account): void {
        $this->accounts[$account->id] = $account;
    }
    
    public function findById(string $id): ?Account {
        return $this->accounts[$id] ?? null;
    }
}

// Stub — retorna valor fixo
$gateway = $this->createStub(PaymentGateway::class);
$gateway->method('charge')->willReturn(true);

// Mock — verifica interação
$mailer = $this->createMock(Mailer::class);
$mailer->expects($this->once())
       ->method('send')
       ->with($this->isInstanceOf(Email::class));
```

## Testes de Erro

Tão importantes quanto os de sucesso:

```php
public function test_transfer_fails_with_insufficient_funds(): void {
    $repo = new InMemoryAccountRepository();
    $repo->save(new Account('A', 100));
    $repo->save(new Account('B', 0));
    $service = new AccountService($repo);

    $this->expectException(InsufficientFundsException::class);
    $service->transfer('A', 'B', 500);

    // Verifica que saldo NÃO mudou
    $this->assertEquals(100, $repo->findById('A')->balance);
}

public function test_transfer_fails_with_nonexistent_account(): void {
    $this->expectException(AccountNotFoundException::class);
    $service->transfer('INEXISTENTE', 'B', 100);
}
```

## PEST (alternativa moderna)

```php
it('transfers money between accounts', function () {
    $repo = new InMemoryAccountRepository();
    $repo->save(new Account('A', 1000));
    $repo->save(new Account('B', 500));
    $service = new AccountService($repo);

    $service->transfer('A', 'B', 300);

    expect($repo->findById('A')->balance)->toBe(700)
        ->and($repo->findById('B')->balance)->toBe(800);
});

it('rejects transfer with insufficient funds')
    ->throws(InsufficientFundsException::class);
```

## Related

- [[PHP/SOLID]]
- [[PHP/PHP]]
- [[PHP/Concurrency]]

## Resources

- https://pestphp.com
- https://phpunit.de/documentation.html

#### My commentaries
- 
