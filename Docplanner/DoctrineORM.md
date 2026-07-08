---
tags: [docplanner-prep, symfony, doctrine]
status: draft
---

# Doctrine ORM

## Por que isso importa
Docplanner usa DDD + hexagonal. Doctrine é Data Mapper — a entity não sabe do banco. Isso permite que o domínio seja puro, que é exatamente o que hexagonal exige. Se fosse Eloquent (Active Record), o model estaria acoplado ao banco.

## Data Mapper vs Active Record

```
Active Record (Eloquent):
  User model ──► sabe se salvar ──► conhece a tabela
  $user->save()

Data Mapper (Doctrine):
  User entity ──► POPO puro ──► não sabe do banco
  EntityManager ──► persiste a entity ──► conhece a tabela
```

A entity é só um objeto PHP. Quem mapeia pro banco é o Doctrine via metadata (annotations/attributes/XML).

## Entity

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: UserRepository::class)]
#[ORM\Table(name: 'users')]
class User {
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;

    #[ORM\Column(length: 255)]
    private string $name;

    #[ORM\Column(length: 255, unique: true)]
    private string $email;

    #[ORM\OneToMany(targetEntity: Order::class, mappedBy: 'user', cascade: ['persist'])]
    private Collection $orders;

    public function __construct(string $name, string $email) {
        $this->name = $name;
        $this->email = $email;
        $this->orders = new ArrayCollection();
    }

    // Getters e behavior methods — sem save(), sem query
}
```

Note: **sem `save()`**, sem `find()`, sem nada de banco. É um objeto de domínio puro.

## EntityManager e Unit of Work

```php
// Buscar
$user = $em->find(User::class, 1);
// ou via repository
$user = $userRepo->findOneBy(['email' => 'dylan@test.com']);

// Criar
$user = new User('Dylan', 'dylan@test.com');
$em->persist($user);  // diz pro Doctrine rastrear esse objeto

// Atualizar
$user->setName('Dylan Oazen');  // só muda o objeto
// Doctrine detecta a mudança automaticamente (dirty checking)

// Salvar tudo de uma vez
$em->flush();  // gera e executa os SQLs necessários
```

`flush()` é onde a mágica acontece — Doctrine compara o estado atual dos objetos rastreados com o que tinha antes e gera os INSERTs/UPDATEs necessários. É o **Unit of Work** pattern.

## Repository

```php
// Doctrine Repository — Query Builder
class UserRepository extends ServiceEntityRepository {
    public function __construct(ManagerRegistry $registry) {
        parent::__construct($registry, User::class);
    }

    public function findActiveByCity(string $city): array {
        return $this->createQueryBuilder('u')
            ->where('u.active = :active')
            ->andWhere('u.city = :city')
            ->setParameter('active', true)
            ->setParameter('city', $city)
            ->orderBy('u.name', 'ASC')
            ->getQuery()
            ->getResult();
    }
}
```

### DQL (Doctrine Query Language)

```php
$query = $em->createQuery(
    'SELECT u FROM App\Entity\User u WHERE u.active = true ORDER BY u.name'
);
$users = $query->getResult();
```

DQL parece SQL mas opera sobre **objetos**, não tabelas. `FROM User` não é a tabela, é a entity class.

## Relationships

```php
// OneToMany / ManyToOne
#[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'orders')]
private User $user;

#[ORM\OneToMany(targetEntity: Order::class, mappedBy: 'user')]
private Collection $orders;

// ManyToMany
#[ORM\ManyToMany(targetEntity: Tag::class)]
#[ORM\JoinTable(name: 'post_tags')]
private Collection $tags;
```

**Owning side vs inverse side**: o lado com `JoinColumn` ou `JoinTable` é o owning. Doctrine só persiste mudanças do owning side.

## Migrations

```bash
# Gera migration baseada na diff entre entities e banco
php bin/console doctrine:migrations:diff

# Executa
php bin/console doctrine:migrations:migrate

# Status
php bin/console doctrine:migrations:status
```

## Comparação com minha stack

| Eloquent | Doctrine |
|----------|----------|
| `$user->save()` | `$em->flush()` |
| `User::where(...)` | `$repo->createQueryBuilder()` |
| Migrations manuais | Migrations auto-geradas (diff) |
| `$user->orders` (lazy) | `$user->getOrders()` (lazy por default) |
| Casts, accessors | Embeddables, custom types |

## Perguntas que podem cair
1. "Qual a diferença entre Active Record e Data Mapper?" → AR o model se persiste. DM o entity é puro e o EntityManager persiste.
2. "O que é Unit of Work?" → Doctrine rastreia objetos e gera os SQLs no flush() comparando estado atual vs original.
3. "Como funciona lazy loading no Doctrine?" → Proxy objects. Doctrine gera uma classe proxy que carrega os dados do banco só quando acessados.
4. "O que é owning side?" → O lado da relação que tem a foreign key. Doctrine só persiste mudanças do owning side.

## Links
- [[Docplanner/SymfonyVsLaravel]]
- [[Docplanner/SymfonyServiceContainer]]
- [[PHP/Persistence]]
- [[Architecture/HexagonalArchitecture]]
