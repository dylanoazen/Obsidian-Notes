# Representação de Dinheiro

Por que nunca usar float pra dinheiro e como fazer certo — especialmente em fintech multi-país.

Related: [[PHP/Concurrency]]

---

## O Problema com Float

```php
$a = 0.1 + 0.2;
echo $a;        // 0.30000000000000004
echo $a == 0.3; // false!
```

```php
$balance = 10.00;
$balance -= 0.10;
$balance -= 0.10;
$balance -= 0.10;
// $balance = 9.700000000000001   ← centavo fantasma
```

Float usa representação binária (IEEE 754) — **não consegue representar 0.1 exatamente**. Em dinheiro, isso é inaceitável: centavos somem ou aparecem do nada.

## Solução: Inteiros em Centavos

Armazene na **menor unidade da moeda**:

```php
// ❌ Float
$balance = 149.99;

// ✅ Inteiros (centavos)
$balance = 14999; // R$ 149,99

// Operações são exatas
$balance -= 1000;  // debita R$ 10,00
// $balance = 13999 → R$ 139,99 ✅

// Pra exibir:
echo number_format($balance / 100, 2, ',', '.');
// "139,99"
```

## Money Pattern (Valor + Moeda)

```php
readonly class Money {
    public function __construct(
        public int $amount,      // em centavos
        public string $currency  // ISO 4217: BRL, USD, EUR
    ) {}

    public function add(Money $other): Money {
        if ($this->currency !== $other->currency) {
            throw new CurrencyMismatchException();
        }
        return new Money($this->amount + $other->amount, $this->currency);
    }

    public function subtract(Money $other): Money {
        if ($this->currency !== $other->currency) {
            throw new CurrencyMismatchException();
        }
        if ($this->amount < $other->amount) {
            throw new InsufficientFundsException();
        }
        return new Money($this->amount - $other->amount, $this->currency);
    }

    public function format(): string {
        return number_format($this->amount / 100, 2, ',', '.') 
               . ' ' . $this->currency;
    }
}

// Uso
$price = new Money(14999, 'BRL');  // R$ 149,99
$tax = new Money(1500, 'BRL');     // R$ 15,00
$total = $price->add($tax);        // R$ 164,99
```

## Cuidado com Divisão

```php
// Dividir R$ 10,00 por 3 pessoas?
10000 / 3 = 3333.333...

// Solução: arredonda + distribui o resto
$share = intdiv(10000, 3);  // 3333
$remainder = 10000 % 3;      // 1

// Pessoa 1: 3334 (fica com o centavo extra)
// Pessoa 2: 3333
// Pessoa 3: 3333
// Total: 10000 ✅
```

## No Banco de Dados

```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY,
    balance BIGINT NOT NULL DEFAULT 0,  -- centavos, NUNCA DECIMAL
    currency CHAR(3) NOT NULL DEFAULT 'BRL',
    CHECK (balance >= 0)
);
```

- `BIGINT` não `DECIMAL` — mais rápido, sem surpresas de arredondamento
- `CHECK (balance >= 0)` — constraint no banco como última defesa
- Moedas com subdivisões diferentes: JPY não tem centavos (1 JPY = menor unidade)

## Na Entrevista

> "Uso inteiros representando centavos porque float tem erro de ponto flutuante — em dinheiro isso é inaceitável. O padrão Money (valor + moeda) encapsula isso e impede operações entre moedas diferentes."

## Related

- [[PHP/Concurrency]]
- [[PHP/Persistence]]

## Resources

- https://martinfowler.com/eaaCatalog/money.html
- moneyphp/money (lib PHP para Money pattern)

#### My commentaries
- 
