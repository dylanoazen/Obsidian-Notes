ce# Eloquent ORM

O ORM do Laravel. Implementa o padrão **Active Record** — cada Model é ao mesmo tempo a representação da tabela e o objeto de domínio. O model sabe se salvar, se buscar, se deletar.

Related: [[PHP/Laravel]], [[PHP/DoctrineORM]], [[PHP/SymfonyVsLaravel]]

---

## Conceito: Active Record

```
Model = tabela + dados + operações de persistência
```

O objeto carrega os dados E sabe se persistir. Diferente do Doctrine (Data Mapper), onde o objeto é separado do mecanismo de persistência.

```php
// Active Record (Eloquent)
$user = User::find(1);   // objeto que representa uma linha da tabela
$user->name = 'Dylan';
$user->save();           // o próprio objeto se salva

// Data Mapper (Doctrine)
$user = $em->find(User::class, 1);  // objeto puro
$user->setName('Dylan');
$em->flush();            // EntityManager salva — o objeto não sabe nada sobre banco
```

---

## Setup Básico

```php
// Toda Model estende Eloquent\Model
class User extends Model
{
    // por convenção: tabela = users (plural do nome da classe)
    // se quiser sobrescrever:
    protected $table = 'usuarios';

    // campos que podem ser preenchidos em massa
    protected $fillable = ['name', 'email', 'password'];

    // ou o inverso — campos que NÃO podem ser preenchidos em massa
    protected $guarded = ['id', 'is_admin'];

    // campos ocultados no toArray() / toJson()
    protected $hidden = ['password', 'remember_token'];

    // casting automático de tipos
    protected $casts = [
        'email_verified_at' => 'datetime',
        'is_active'         => 'boolean',
        'metadata'          => 'array',   // JSON → array automaticamente
    ];
}
```

---

## CRUD

### Create

```php
// create() — preenche e salva em uma linha (usa $fillable)
$user = User::create([
    'name'  => 'Dylan',
    'email' => 'dylan@example.com',
]);

// new + save() — mais controle
$user = new User();
$user->name  = 'Dylan';
$user->email = 'dylan@example.com';
$user->save();

// firstOrCreate — busca, cria se não existir
$user = User::firstOrCreate(
    ['email' => 'dylan@example.com'],       // critério de busca
    ['name' => 'Dylan']                     // valores extras se criar
);

// updateOrCreate — atualiza se existir, cria se não existir
$user = User::updateOrCreate(
    ['email' => 'dylan@example.com'],
    ['name' => 'Dylan', 'active' => true]
);
```

### Read

```php
// por ID
$user = User::find(1);           // null se não encontrar
$user = User::findOrFail(1);     // lança ModelNotFoundException se não encontrar

// primeiro resultado
$user = User::first();
$user = User::firstOrFail();

// todos
$users = User::all();            // cuidado em tabelas grandes — sem paginação

// com condições
$users = User::where('active', true)->get();
$users = User::where('age', '>=', 18)->where('active', true)->get();

// OR
$users = User::where('role', 'admin')
             ->orWhere('role', 'moderator')
             ->get();

// IN
$users = User::whereIn('role', ['admin', 'moderator'])->get();

// LIKE
$users = User::where('name', 'like', 'Dy%')->get();

// count
$total = User::where('active', true)->count();

// exists
$exists = User::where('email', 'dylan@example.com')->exists();
```

### Update

```php
// atualiza o objeto em memória e salva
$user = User::find(1);
$user->name = 'Dylan Ozen';
$user->save();

// update() — atualiza direto no banco sem carregar o objeto
User::where('active', false)->update(['deleted_at' => now()]);
```

### Delete

```php
// deleta o objeto
$user = User::find(1);
$user->delete();

// deleta por condição direto no banco
User::where('created_at', '<', now()->subYears(2))->delete();

// Soft Delete — marca como deletado sem remover do banco
// adiciona deleted_at na tabela e use SoftDeletes no model
use Illuminate\Database\Eloquent\SoftDeletes;

class User extends Model {
    use SoftDeletes;
}

$user->delete();             // seta deleted_at — não apaga do banco
User::withTrashed()->get();  // inclui deletados
User::onlyTrashed()->get();  // só os deletados
$user->restore();            // restaura (limpa deleted_at)
$user->forceDelete();        // apaga de verdade
```

---

## Query Builder

Eloquent tem acesso completo ao Query Builder do Laravel:

```php
User::select('name', 'email')
    ->where('active', true)
    ->orderBy('name')
    ->limit(10)
    ->offset(20)
    ->get();

// raw SQL quando necessário
User::whereRaw('YEAR(created_at) = ?', [2024])->get();
User::selectRaw('COUNT(*) as total, role')->groupBy('role')->get();
```

---

## Relacionamentos

### hasOne / belongsTo (1:1)

```php
class User extends Model {
    public function profile(): HasOne {
        return $this->hasOne(Profile::class);
        // SELECT * FROM profiles WHERE user_id = {user.id}
    }
}

class Profile extends Model {
    public function user(): BelongsTo {
        return $this->belongsTo(User::class);
        // SELECT * FROM users WHERE id = {profile.user_id}
    }
}

// uso
$user->profile;      // acessa como propriedade — executa o SQL
$profile->user;
```

### hasMany / belongsTo (1:N)

```php
class Post extends Model {
    public function comments(): HasMany {
        return $this->hasMany(Comment::class);
    }
}

class Comment extends Model {
    public function post(): BelongsTo {
        return $this->belongsTo(Post::class);
    }
}

// uso
$post->comments;        // Collection de Comments
$post->comments()->where('approved', true)->get();  // com filtro adicional
```

### belongsToMany (N:N)

```php
class Post extends Model {
    public function tags(): BelongsToMany {
        return $this->belongsToMany(Tag::class);
        // tabela pivot: post_tag (por convenção, nomes em ordem alfabética)
    }
}

// pivot com colunas extras
$this->belongsToMany(Tag::class)->withPivot('created_by')->withTimestamps();

// uso
$post->tags;
$post->tags()->attach($tagId);           // adiciona na pivot
$post->tags()->detach($tagId);           // remove da pivot
$post->tags()->sync([$id1, $id2]);       // substitui tudo pelos IDs fornecidos
```

### hasOneThrough / hasManyThrough

```php
// User tem muitos Posts, Post tem muitos Comments
// User → Comments (através de Posts)
class User extends Model {
    public function comments(): HasManyThrough {
        return $this->hasManyThrough(Comment::class, Post::class);
    }
}
```

---

## Eager Loading — N+1

**O problema N+1:** sem eager loading, cada acesso a um relacionamento dispara um novo SQL.

```php
// ❌ N+1: 1 query para posts + 1 query por post para carregar author
$posts = Post::all();              // SELECT * FROM posts
foreach ($posts as $post) {
    echo $post->author->name;      // SELECT * FROM users WHERE id = ? (repetido N vezes)
}

// ✅ Eager loading: 2 queries no total
$posts = Post::with('author')->get();
// SELECT * FROM posts
// SELECT * FROM users WHERE id IN (1, 2, 3, ...)

// múltiplos relacionamentos
$posts = Post::with(['author', 'tags', 'comments'])->get();

// nested
$posts = Post::with('comments.author')->get();
```

### Lazy Eager Loading

```php
// carrega o relacionamento depois de já ter a coleção
$posts = Post::all();
$posts->load('author');          // executa o SQL agora
$posts->loadMissing('author');   // só carrega se ainda não carregou
```

---

## Scopes

Encapsulam condições reutilizáveis no próprio model:

```php
class User extends Model {
    // Local Scope
    public function scopeActive(Builder $query): Builder {
        return $query->where('active', true);
    }

    public function scopeOlderThan(Builder $query, int $age): Builder {
        return $query->where('age', '>=', $age);
    }

    // Global Scope — aplicado automaticamente em TODAS as queries
    protected static function booted(): void {
        static::addGlobalScope('active', function (Builder $builder) {
            $builder->where('active', true);
        });
    }
}

// uso de local scope (remove o "scope" do nome)
User::active()->olderThan(18)->get();

// ignorar global scope quando necessário
User::withoutGlobalScope('active')->get();
```

---

## Mutators e Accessors

Transformam valores ao ler/gravar no model:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model {
    // Accessor — transforma ao ler
    protected function name(): Attribute {
        return Attribute::make(
            get: fn ($value) => ucfirst($value),
        );
    }

    // Mutator — transforma ao gravar
    protected function password(): Attribute {
        return Attribute::make(
            set: fn ($value) => bcrypt($value),
        );
    }
}

$user->name;       // retorna com ucfirst aplicado
$user->password = 'secret';  // armazena já com bcrypt
```

---

## Events do Model

Eloquent dispara eventos em operações de ciclo de vida:

```php
class User extends Model {
    protected static function booted(): void {
        // antes de salvar (create e update)
        static::saving(function (User $user) {
            $user->slug = Str::slug($user->name);
        });

        // depois de criar
        static::created(function (User $user) {
            Mail::to($user)->send(new WelcomeMail($user));
        });

        // antes de deletar
        static::deleting(function (User $user) {
            $user->posts()->delete();  // cascade manual
        });
    }
}
```

Eventos disponíveis: `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored`.

---

## Paginação

```php
// paginate() — retorna LengthAwarePaginator com metadados
$users = User::paginate(15);          // 15 por página
$users = User::paginate(15, page: 2); // página específica

// simplePaginate() — sem contar total (mais performático)
$users = User::simplePaginate(15);

// uso em controller + view
return view('users.index', compact('users'));

// no Blade
{{ $users->links() }}  // renderiza botões de navegação

// em API — retorna JSON com metadados de paginação
return response()->json($users);
```

---

## Transactions

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    $from = Account::lockForUpdate()->find(1);  // SELECT FOR UPDATE
    $to   = Account::lockForUpdate()->find(2);

    $from->balance -= 100;
    $to->balance   += 100;

    $from->save();
    $to->save();
});
// commit automático se não lançar exceção
// rollback automático se lançar exceção
```

---

## Tradeoffs

| Ótimo para | Limitação |
|---|---|
| CRUD rápido | Model mistura domínio com persistência |
| Protótipos e painéis | Difícil de usar com DDD / Hexagonal |
| Times pequenos | Magia implícita complica rastreamento |
| Queries simples a médias | Queries muito complexas ficam verbosas |

Para DDD sério: [[PHP/DoctrineORM]] (Data Mapper — objeto puro, separado da persistência).

---

## Related

- [[PHP/Laravel]]
- [[PHP/DoctrineORM]]
- [[PHP/SymfonyVsLaravel]]
- [[Architecture/DDD]]

#### My commentaries
-
