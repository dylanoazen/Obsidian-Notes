# Meios de Pagamento

Como o dinheiro se move — os métodos, redes e ferramentas do ecossistema de pagamentos.

Related: [[Finance/CoreBanking]], [[Finance/FinanceEng]]

---

## O Ecossistema

```
Comprador → Adquirente → Bandeira → Emissor → Banco do vendedor
```

- **Comprador** — quem paga
- **Adquirente** — processa o pagamento do comerciante (Cielo, Rede, Stone)
- **Bandeira** — rede de liquidação (Visa, Mastercard, Elo)
- **Emissor** — banco que emitiu o cartão do comprador (Itaú, Bradesco, Nubank)
- **Banco do vendedor** — onde o dinheiro cai

---

## Métodos no Brasil

### PIX
Transferência instantânea criada pelo Banco Central. Disponível 24/7, liquida em segundos.

```
Características:
- Gratuito para pessoas físicas
- Liquidação: imediata (D+0)
- Chaves: CPF, CNPJ, email, telefone, chave aleatória
- Limit: sem limite definido (cada banco define)
```

Do ponto de vista técnico: é uma API do Banco Central que bancos e fintechs integram via certificado digital.

### Boleto Bancário
Documento de cobrança com código de barras. Pago em banco, lotérica, app.

```
Características:
- Aceito por qualquer pessoa (sem conta, sem cartão)
- Liquidação: D+1 a D+3
- Risco: pode não ser pago (boleto não pago = inadimplência)
- Muito usado em e-commerce e cobranças recorrentes
```

### Cartão de Crédito / Débito
Rede Visa, Mastercard, Elo, Amex. Débito é imediato, crédito tem ciclo de faturamento.

```
Crédito: autorização → captura → liquidação (D+30 a D+31)
Débito:  autorização → liquidação (D+1)
```

### TED / DOC
Transferências bancárias tradicionais. TED é o mesmo dia (até 17h), DOC é D+1. Sendo substituídos pelo PIX.

---

## EBANX — O que Fazem

O EBANX é um **payment processor** focado em mercados emergentes da América Latina e outros países. Eles atuam como intermediários entre empresas globais (Netflix, Spotify, Shopify) e os métodos de pagamento locais.

```
Netflix (EUA)
    │
    ▼
EBANX ← integra com PIX, boleto, cartões locais, carteiras digitais
    │
    ▼
Usuário no Brasil paga em reais com método local
    │
    ▼
Netflix recebe em dólares
```

### Produtos EBANX

**EBANX Pay** — checkout que aceita métodos locais de múltiplos países
**EBANX Account** — conta digital para usuários finais
**EBANX for Business** — APIs para empresas integrarem pagamentos
**EBANX Payout** — pagamentos em massa (desembolsos)

### Países de operação
Brasil, México, Argentina, Colômbia, Chile, Peru, Equador, Bolívia, e outros.

### Desafio técnico do EBANX
Cada país tem regulação diferente, moeda diferente, métodos diferentes. O sistema precisa abstrair tudo isso para o cliente global.

```
México: OXXO (pagamento em conveniência), SPEI (transferência)
Argentina: Rapipago, PagoFácil
Brasil: PIX, boleto, TED
```

---

## Carteiras Digitais

Intermediários que guardam saldo e processam pagamentos:

| Carteira | País | Característica |
|---|---|---|
| Mercado Pago | LatAm | Mais usado, integrado ao Mercado Livre |
| PicPay | Brasil | P2P e pagamentos |
| PayPal | Global | Internacional, taxas altas |
| Apple Pay / Google Pay | Global | Tokenização de cartão |

---

## Redes Internacionais

### SWIFT
Sistema de mensageria para transferências internacionais entre bancos. Lento (D+1 a D+5) e caro. Usado para transferências internacionais entre instituições financeiras.

### SEPA
Equivalente europeu ao PIX — transferências instantâneas na zona do Euro.

### Visa / Mastercard
Redes de liquidação de cartão. Definem as regras, tarifas e padrões de segurança (PCI-DSS).

---

## Conceitos Importantes

### MDR (Merchant Discount Rate)
Taxa que o comerciante paga para aceitar cartão. Geralmente 1-3% do valor da transação.

### Chargeback
Cliente contesta cobrança. Dinheiro volta ao cliente. Comerciante perde a mercadoria E o dinheiro.

### Tokenização
Substituir dados reais do cartão por um token. Nunca trafega o número real do cartão — reduz risco de fraude.

```
Cartão real:  4111 1111 1111 1111
Token:        tok_abc123xyz (usado nas transações)
```

### 3DS (3D Secure)
Autenticação adicional em compras online — a tela de "confirme no app do banco". Reduz fraude mas adiciona fricção.

---

## Related

- [[Finance/CoreBanking]]
- [[Finance/FinanceEng]]
- [[DistributedSystems/Idempotency]]
- [[DistributedSystems/SecurityAuth]]

#### My commentaries
-
