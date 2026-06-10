# PHP 8.1+ — Libs e Ferramentas Essenciais

O que um dev Pleno/Sênior precisa dominar no ecossistema PHP moderno (8.1+).

---

## Composer — Gerenciador de Dependências

Não é lib, mas é a base de tudo. Equivalente ao npm/go mod.

```bash
composer init                    # novo projeto
composer require vendor/package  # instalar
composer update                  # atualizar deps
composer dump-autoload -o        # otimizar autoload
```

- `composer.lock` → trava versões (sempre commitar)
- PSR-4 autoloading → padrão de autoload moderno
- Packagist.org → registry central

## Frameworks

### Laravel

O framework dominante no mercado PHP. Saber Laravel é quase obrigatório.

```php
// Routing
Route::get('/users/{id}', [UserController::class, 'show']);

// Eloquent ORM
$users = User::where('active', true)
    ->orderBy('name')
    ->paginate(15);

// Queue jobs
dispatch(new ProcessPayment($order));

// Events
event(new OrderShipped($order));
```

Ecossistema Laravel essencial:
- **Laravel Sail** — ambiente Docker local
- **Laravel Sanctum** — autenticação API (tokens/SPA)
- **Laravel Horizon** — dashboard para filas (Redis)
- **Laravel Telescope** — debug/profiling em dev
- **Laravel Pint** — code style fixer (baseado no PHP-CS-Fixer)
- **Laravel Cashier** — billing com Stripe/Paddle
- **Laravel Nova / Filament** — admin panels
- **Laravel Livewire** — componentes reativos sem JS
- **Inertia.js** — SPA com Vue/React usando rotas do Laravel

### Symfony

Mais modular, usado em projetos enterprise. Muitos componentes do Laravel são Symfony por baixo.

```php
// Symfony Console (usado por artisan, phpunit, etc.)
#[AsCommand(name: 'app:import')]
class ImportCommand extends Command { }

// Dependency Injection
#[Autowire]
public function __construct(private UserRepository $users) { }

// HTTP Foundation (usado por quase todo framework PHP)
$request = Request::createFromGlobals();
```

Componentes Symfony que todo dev PHP usa (mesmo sem Symfony):
- **HttpFoundation** — Request/Response objects
- **Console** — CLI apps
- **Mailer** — envio de emails
- **Messenger** — message bus / queues
- **Security** — autenticação e autorização

## ORM / Database

### Doctrine (Symfony)

```php
#[Entity]
class User {
    #[Id, GeneratedValue]
    #[Column(type: 'integer')]
    private int $id;

    #[Column(length: 255)]
    private string $name;

    #[OneToMany(targetEntity: Order::class, mappedBy: 'user')]
    private Collection $orders;
}
```

### Eloquent (Laravel)

```php
class User extends Model {
    protected $fillable = ['name', 'email'];
    
    public function orders(): HasMany {
        return $this->hasMany(Order::class);
    }
}

// Scopes
User::active()->verified()->get();
```

## Testes

### PHPUnit

Padrão da indústria:

```php
class UserServiceTest extends TestCase {
    public function test_creates_user_with_valid_data(): void {
        $service = new UserService();
        $user = $service->create(['name' => 'Dylan', 'email' => 'a@b.com']);
        
        $this->assertInstanceOf(User::class, $user);
        $this->assertEquals('Dylan', $user->name);
    }
}
```

### PEST

Framework de testes moderno (wrapper do PHPUnit, mais limpo):

```php
it('creates a user with valid data', function () {
    $user = UserService::create(['name' => 'Dylan']);
    
    expect($user)
        ->toBeInstanceOf(User::class)
        ->name->toBe('Dylan');
});
```

### Mockery

```php
$mock = Mockery::mock(PaymentGateway::class);
$mock->shouldReceive('charge')
     ->once()
     ->with(1000)
     ->andReturn(true);
```

## Análise Estática

### PHPStan / Larastan

Encontra bugs sem rodar o código. Essencial em projetos sérios.

```bash
# Instalar
composer require --dev phpstan/phpstan
# Laravel
composer require --dev larastan/larastan

# Rodar
vendor/bin/phpstan analyse src --level=8
```

Níveis 0-9: quanto maior, mais rigoroso. Projetos maduros devem mirar level 6+.

### Psalm

Alternativa ao PHPStan (Vimeo). Mais focado em tipos.

## Code Quality

### PHP-CS-Fixer / Laravel Pint

```bash
# Pint (Laravel)
vendor/bin/pint

# PHP-CS-Fixer
vendor/bin/php-cs-fixer fix src/
```

### Rector

Refactoring automático e upgrades de versão do PHP:

```bash
# Upgrade automatico PHP 7.4 → 8.1
vendor/bin/rector process src
```

Atualiza syntax, types, deprecated functions automaticamente.

## HTTP / API

### Guzzle

Client HTTP mais usado:

```php
$client = new GuzzleHttp\Client();
$response = $client->get('https://api.example.com/users', [
    'headers' => ['Authorization' => 'Bearer ' . $token],
    'query' => ['page' => 1],
]);
$data = json_decode($response->getBody(), true);
```

### Symfony HTTP Client

Alternativa moderna:

```php
$response = $client->request('GET', 'https://api.example.com/users');
$data = $response->toArray();
```

### Laravel HTTP Client (wrapper do Guzzle)

```php
$response = Http::withToken($token)
    ->timeout(10)
    ->retry(3, 100)
    ->get('https://api.example.com/users');

$users = $response->json();
```

## Filas e Mensageria

```php
// Laravel Queues
class ProcessPayment implements ShouldQueue {
    use Dispatchable, InteractsWithQueue, Queueable;
    
    public function handle(): void {
        // process payment
    }
    
    public function failed(Throwable $e): void {
        // handle failure
    }
}

// Dispatch
ProcessPayment::dispatch($order)->onQueue('payments');
```

Drivers: Redis (mais comum), Amazon SQS, RabbitMQ, database.

## Cache

```php
// Laravel Cache
Cache::remember('users:active', 3600, fn () => 
    User::where('active', true)->get()
);

// Tags (Redis/Memcached)
Cache::tags(['users'])->flush();
```

## Autenticação e Autorização

- **Laravel Sanctum** — tokens para API, session auth para SPA
- **Laravel Passport** — OAuth2 completo (se precisar)
- **JWT (tymon/jwt-auth)** — JWT tokens
- **Spatie Permission** — roles e permissions

```php
// Spatie Permission
$user->assignRole('editor');
$user->givePermissionTo('publish articles');

if ($user->can('publish articles')) { }
```

## Pacotes Spatie (obrigatórios)

Spatie é o publisher mais importante do ecossistema Laravel:

| Pacote | O que faz |
|--------|-----------|
| spatie/laravel-permission | Roles e permissions |
| spatie/laravel-medialibrary | Upload e manipulação de arquivos |
| spatie/laravel-activitylog | Audit log de models |
| spatie/laravel-backup | Backup automatizado |
| spatie/laravel-data | DTOs tipados |
| spatie/laravel-query-builder | Filter/sort/include via query string |
| spatie/laravel-settings | Settings key-value persistidas |

## Features do PHP 8.1+ que Deve Dominar

```php
// Enums (8.1)
enum Status: string {
    case Active = 'active';
    case Inactive = 'inactive';
}

// Readonly properties (8.1)
class User {
    public function __construct(
        public readonly string $name,
        public readonly string $email,
    ) {}
}

// Fibers (8.1) — base para async
$fiber = new Fiber(function (): void {
    Fiber::suspend('paused');
});
$fiber->start(); // returns 'paused'
$fiber->resume();

// Intersection types (8.1)
function process(Countable&Iterator $items): void { }

// Readonly classes (8.2)
readonly class Money {
    public function __construct(
        public int $amount,
        public string $currency,
    ) {}
}

// DNF types (8.2)
function handle((Countable&Iterator)|null $items): void { }

// Typed class constants (8.3)
class Config {
    const string VERSION = '1.0.0';
}

// #[Override] attribute (8.3)
class Child extends Parent {
    #[Override]
    public function handle(): void { }
}
```

## Monitoramento e Debug

- **Laravel Telescope** — debug dashboard (dev)
- **Laravel Debugbar** — debug bar no browser
- **Sentry (sentry/sentry-laravel)** — error tracking (prod)
- **Xdebug** — step debugging, profiling
- **Clockwork** — profiling de requests

## DevOps / Deploy

- **Laravel Forge** — provisioning de servers
- **Laravel Vapor** — serverless deploy na AWS Lambda
- **Deployer (deployer/deployer)** — deploy automation
- **Laravel Octane** — servidor de alta performance (Swoole/RoadRunner)

```php
// Octane — mantém app em memória entre requests
// Cuidado com estado compartilhado!
php artisan octane:start --server=frankenphp
```

## Related

- [[PHP/EventLoop]]
- [[PHP/ReactPHP]]
- [[PHP/Concurrency]]
- [[DevOps/Docker/Docker]]
- [[DevOps/CICD/CICD]]
- [[Security/IAM]]

## Resources

- https://laravel.com/docs
- https://symfony.com/doc/current/index.html
- https://phpstan.org
- https://pestphp.com
- https://spatie.be/open-source

#### My commentaries
- 
