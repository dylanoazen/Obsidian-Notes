# EBANX — Visão Geral

O que é o EBANX, como funciona e por que existe.

Related: [[Finance/MeiosDePagamento]], [[Finance/CoreBanking]], [[Finance/EntrevistaFelipe]]

---

## O que é

O EBANX é uma empresa brasileira fundada em 2012 em Curitiba. É um **payment processor** — processa pagamentos entre empresas globais e consumidores em mercados emergentes da América Latina e outros países.

Não é um banco. Não é um BaaS. É o intermediário especializado em fazer dinheiro fluir entre empresas de fora e consumidores locais.

---

## O Problema que Resolve

Uma empresa americana como a Netflix quer cobrar usuários no Brasil. Problema:

- Usuário brasileiro quer pagar com PIX ou boleto
- Netflix está nos EUA, quer receber em dólar
- Netflix não tem licença para operar no Brasil
- Integrar com cada método de pagamento local de cada país é complexo e caro

```
Netflix (EUA) quer R$30/mês do usuário brasileiro
     │
     │  não sabe como aceitar PIX
     │  não tem CNPJ
     │  não quer lidar com regulação brasileira
     ▼
EBANX resolve tudo isso
     │
     ▼
Usuário paga R$30 via PIX
EBANX converte para dólar e repassa para a Netflix
```

O EBANX faz isso em dezenas de países, cada um com seus métodos locais.

---

## Como Funciona na Prática

**Para o consumidor:**
Você assina a Netflix. Na hora de pagar, aparece a opção PIX ou boleto — isso é o EBANX por baixo, não a Netflix.

**Para a empresa global:**
Integra uma única API do EBANX. O EBANX cuida de tudo: PIX no Brasil, OXXO no México, Rapipago na Argentina, etc. A empresa recebe em dólar, sem precisar saber que existe boleto.

```
Netflix integra uma API → EBANX
                              ├── Brasil: PIX, boleto, cartão
                              ├── México: OXXO, SPEI, cartão
                              ├── Argentina: Rapipago, PagoFácil
                              ├── Colômbia: PSE, cartão
                              └── + 15 países...
```

---

## Números e Escala

- Fundado: 2012, Curitiba
- Presença: +20 países na América Latina, África e outros mercados emergentes
- Clientes: Netflix, Spotify, Airbnb, Shopify, Steam, Headspace, e centenas de outros
- Processam bilhões de dólares por ano em pagamentos
- ~1.000 funcionários

---

## Produtos

**EBANX Pay**
Checkout que aceita métodos locais. O cliente global embute no seu site de pagamento. É o produto principal.

**EBANX Account**
Conta digital para o consumidor final — cartão pré-pago, carteira digital. Permite que pessoas sem cartão de crédito acessem serviços globais.

**EBANX Payout**
Pagamentos em massa para pessoas físicas — desembolsos. Usado por marketplaces para pagar vendedores locais.

**EBANX for Business**
APIs para empresas construírem produtos de pagamento em cima da infraestrutura do EBANX.

---

## Métodos de Pagamento por País

| País | Métodos principais |
|---|---|
| Brasil | PIX, boleto, cartão, TED |
| México | OXXO, SPEI, cartão |
| Argentina | Rapipago, PagoFácil, cartão |
| Colômbia | PSE, Efecty, cartão |
| Chile | Webpay, cartão |
| Peru | PagoEfectivo, cartão |

---

## Desafios Técnicos do EBANX

**Múltiplas integrações**
Cada método de pagamento de cada país tem sua própria API, protocolo e formato de dados. Manter dezenas de integrações funcionando simultaneamente é complexo.

**Regulação por país**
Cada país tem regras diferentes — KYC (verificação de identidade), limites de transação, relatórios para o banco central. O sistema precisa aplicar a regra certa para cada país.

**Conversão de moeda**
PIX recebe em reais, Netflix quer em dólar. É preciso converter no momento certo, com a taxa certa, e registrar tudo para auditoria.

**Alta disponibilidade**
Pagamento que falha é receita perdida. O sistema precisa estar disponível 24 horas por dia, 7 dias por semana em todos os países.

**Idempotência**
Reenvio de pagamento não pode cobrar duas vezes. Com dezenas de métodos e países, o volume de reenvios é alto.

**Reconciliação**
O que o EBANX registrou precisa bater com o que cada banco ou método registrou. Qualquer diferença é investigada.

---

## Posicionamento no Mercado

| Empresa | O que faz | Diferença do EBANX |
|---|---|---|
| Stripe | Payment processor global | Foca em mercados desenvolvidos (EUA, Europa) |
| PayPal | Carteira digital + processor | Global mas não especializado em LatAm |
| Adyen | Payment processor enterprise | Foca em grandes varejistas globais |
| **EBANX** | Payment processor LatAm | Especializado em mercados emergentes, métodos locais |

O nicho do EBANX é exatamente onde Stripe e PayPal são fracos — mercados onde cartão de crédito não é o método dominante e onde a regulação local é complexa.

---

## Relevância para a Entrevista

O Felipe vai querer saber se você entende o negócio. Perguntas certas para fazer no final:

- "Quais são os maiores desafios técnicos de escalar para novos países?"
- "Como vocês lidam com a consistência dos dados entre múltiplas integrações simultâneas?"
- "Qual é o papel do time de engenharia no processo de expansão para um novo mercado?"

---

## Frase de Abertura para a Entrevista

> "O EBANX resolve um problema real de infraestrutura — empresas globais querem acessar consumidores em mercados emergentes, mas cada mercado tem seus próprios métodos de pagamento, regulação e moeda. O EBANX abstrai essa complexidade: a empresa integra uma API e o EBANX cuida do resto em cada país. O desafio técnico é operar dezenas de integrações com alta disponibilidade, idempotência e conformidade regulatória simultânea em múltiplos países."

---

#### My commentaries
-
