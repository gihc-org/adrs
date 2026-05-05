# 0002 — SQLx runtime API frem for compile-time query!-makro

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

SQLx tilbyder to måder at skrive databaseforespørgsler på:

1. **`query!`-makro:** Tjekker SQL mod en rigtig database ved compile-tid og
   genererer stærkt typede result-structs. Kræver `DATABASE_URL` i miljøet
   under kompilering.
2. **Runtime API (`sqlx::query`, `sqlx::query_as`):** Ingen compile-time-check.
   Forespørgsler valideres ikke statisk, men fungerer uden en database under
   build.

Projektet bygges via Docker (multi-stage build). Builder-containeren har ikke
adgang til PostgreSQL.

## Beslutning

Vi bruger **SQLx's runtime API** (`query` / `query_as`) overalt.

## Begrundelse

- Docker's builder-stage har ikke adgang til en kørende PostgreSQL-instans.
  `query!`-makroen kræver `DATABASE_URL` ved compile-tid og ville enten fejle
  eller kræve en separat database-container under build — det gør CI og lokale
  builds mere komplekse.
- Runtime API er tilstrækkeligt type-sikkert via `query_as::<_, User>()` —
  mapping-fejl opdages under test, ikke ved produktionsdeployment.
- Migrationer håndteres af `sqlx::migrate!("./migrations")`, som er embedded
  ved compile-tid og kører automatisk ved opstart — schema er dermed altid
  synkroniseret.

## Konsekvenser

- SQL-fejl (forkert kolonnenavn, manglende tabel) opdages ved kørsel, ikke ved
  compile. Integrationstests (`tests/api.rs`) med `#[sqlx::test]` giver det
  primære safety net.
- Ingen `sqlx prepare`-step nødvendigt i CI-pipeline.
- Nye kolonner kræver en migration-fil i `chat/migrations/` — de køres
  automatisk af `sqlx::migrate!` ved næste opstart.
