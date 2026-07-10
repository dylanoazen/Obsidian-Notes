# Laravel

O framework PHP focado em produtividade e experiência do desenvolvedor.

Related: [[PHP/PHP]], [[PHP/SymfonyVsLaravel]], [[SoftwareEngineering/REST]]

---

## Por que surgiu

Em 2011, Taylor Otwell estava usando **CodeIgniter** — o framework PHP mais popular da época. O problema: CodeIgniter não tinha autenticação embutida, não tinha namespaces, e o código era verboso para coisas simples.

Otwell criou o Laravel para resolver isso: um framework que tornasse as tarefas comuns **agradáveis de escrever**. A frase que define a filosofia:

> "PHP for web artisans" — desenvolvedores que se importam com a qualidade e elegância do código.

Em 2013 lançou o Laravel 4, reescrito em cima dos **componentes do Symfony** (HttpFoundation, Routing, Console). O Laravel nunca reinventou a roda — pegou as peças sólidas do Symfony e colocou uma camada de conveniência em cima.

---

## A Filosofia

**Convention over configuration** — o framework assume defaults razoáveis. Você não precisa configurar nada para começar.

```
Model User      → tabela 'users' (automático)
Model OrderItem → tabela 'order_items' (automático)
Controller      → fica em app/Http/Controllers (automático)
```

Se quiser mudar o padrão, pode. Mas você só configura o que foge ao padrão — não o que segue.

---

## Os Componentes Principais

### Eloquent — ORM

O coração do Laravel. Cada tabela tem um Model correspondente. O Model é a tabela.

```php
// busca, modifica, salva — tudo no mesmo objeto
$user = User::find(1);
$user->name = 'Dylan';
$user->save();

// relacionamentos declarados no próprio model
class Post extends Model {
    public function comments() {
        return $this->hasMany(Comment::class);
    }
}

$post->comments; // SQL automático — SELECT * FROM comments WHERE post_id = 1
```

Simples e rápido. O tradeoff: o model conhece o banco — mistura domínio com persistência. Para CRUDs isso é ótimo. Para DDD isso é um problema.

---

### Service Container

O gerenciador de dependências. Sabe instanciar qualquer classe, resolvendo dependências automaticamente.

```php
// o container cria UserService e injeta UserRepository automaticamente
$service = app(UserService::class);

// ou via construtor em controllers (o Laravel injeta sozinho)
class UserController extends Controller {
    public function __construct(
        private UserService $service
    ) {}
}
```

→ [[PHP/SymfonyServiceContainer]] para entender containers em detalhe.

---

### Facades

Atalhos estáticos para serviços do container. Chamada estática que vira chamada de objeto por baixo.

```php
Cache::get('chave');   // por baixo: app('cache')->get('chave')
Log::info('mensagem'); // por baixo: app('log')->info('mensagem')
Route::get('/home');   // por baixo: app('router')->get('/home')
```

Conveniente para escrita rápida. Problema: esconde dependências — a classe usa sem declarar no construtor.

---

### Artisan — CLI

Interface de linha de comando do Laravel. Tudo que você faz no dia a dia tem um comando:

```bash
php artisan make:model Product -m      # cria Model + Migration
php artisan make:controller Api/ProductController --api
php artisan make:job SyncProductJob
php artisan migrate                    # roda as migrations
php artisan queue:work                 # processa a fila
php artisan tinker                     # REPL interativo
```

Você também cria seus próprios comandos:

```php
class ImportProducts extends Command {
    protected $signature = 'products:import {--limit=100}';

    public function handle(): void {
        $limit = $this->option('limit');
        // lógica aqui
    }
}
```

---

### Migrations — Versionamento do Banco

Cada alteração no schema é um arquivo versionado. O time inteiro roda `php artisan migrate` e todos ficam com o mesmo banco.

```php
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->integer('price');  // centavos
    $table->boolean('available')->default(true);
    $table->timestamps();
});
```

---

### Queue — Fila de Jobs

Processa tarefas em background, fora do ciclo de request/response.

```php
// dispara o job para a fila — retorna imediatamente
ProcessPayment::dispatch($order);

// o job executa no worker em background
class ProcessPayment implements ShouldQueue {
    public function handle(): void {
        // processa pagamento
    }
}
```

Backends suportados: Redis, SQS, banco de dados, RabbitMQ via pacote.

---

### Events — Publicar e Escutar

Um lugar do código dispara um evento. Listeners reagem sem que o dispatcher saiba quem são.

```php
// dispara
event(new OrderPlaced($order));

// listener — registrado em EventServiceProvider
class SendConfirmationEmail {
    public function handle(OrderPlaced $event): void {
        Mail::to($event->order->user)->send(new OrderConfirmation($event->order));
    }
}
```

---

### Middleware

Filtra ou transforma requests antes de chegarem ao controller.

```php
// registra na rota
Route::get('/admin', AdminController::class)->middleware('auth');

// ou cria o seu
class EnsureUserIsActive {
    public function handle(Request $request, Closure $next): Response {
        if (!$request->user()->active) {
            return redirect('/inactive');
        }
        return $next($request); // passa pra frente
    }
}
```

---

### Blade — Templates

Engine de templates com herança de layouts:

```blade
{{-- layout base --}}
<!DOCTYPE html>
<html>
<body>
    @yield('content')
</body>
</html>

{{-- página que herda --}}
@extends('layouts.app')
@section('content')
    <h1>{{ $user->name }}</h1>
    @foreach ($products as $product)
        <p>{{ $product->name }}</p>
    @endforeach
@endsection
```

---

## Onde Laravel brilha

- CRUDs e painéis admin — Eloquent + Filament/Nova = produto rápido
- APIs REST — Resources, Form Requests, Sanctum
- Sistemas de filas — Queue + Horizon (painel de monitoramento)
- Times pequenos e médios — convenção acelera, menos config

## Onde tem limitação

- Domínio complexo com DDD — Eloquent mistura persistência com domínio
- Times grandes — facades e magia dificultam rastreamento de dependências
- Sistemas que precisam de arquitetura hexagonal — Symfony + Doctrine encaixa melhor

---

## Related

- [[PHP/SymfonyVsLaravel]]
- [[PHP/PHP]]
- [[PHP/SymfonyServiceContainer]]
- [[SoftwareEngineering/SOLID]]

#### My commentaries
-
