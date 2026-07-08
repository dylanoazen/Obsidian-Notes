---
tags: [docplanner-prep, elasticsearch]
status: draft
---
# ElasticSearch Basics

## Conceitos
Index ≈ tabela. Document ≈ row (JSON). Mapping ≈ schema. Shard ≈ partição.

## Queries
```json
// Full-text
{ "query": { "match": { "name": "cardiologista" } } }

// Bool (combina)
{ "query": { "bool": {
    "must": [{ "match": { "specialty": "cardiology" } }],
    "filter": [{ "range": { "rating": { "gte": 4.0 } } }]
} } }
```
`must` afeta relevância. `filter` só filtra (sim/não).

## Analyzers
lowercase → stop words → stemming. Transformam texto antes de indexar.

## ES vs SQL
SQL: transactions, joins, ACID. ES: full-text, geo, facets. Use ambos: SQL source of truth, ES pra search.

## Links
- [[Docplanner/DoctrineORM]]
