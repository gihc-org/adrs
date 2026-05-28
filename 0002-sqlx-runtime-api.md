# 0002 — SQLx runtime API frem for compile-time query!-makro

**Status:** Accepted  
**Dato:** 2026-05-05

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

## Testkrav

Fordi compile-time-check er fravalgt, er integrationstests det eneste lag der
fanger SQL-fejl. De er derfor ikke valgfrie.

**Krav:**
- Alle funktioner der kalder `sqlx::query` eller `sqlx::query_as` skal have
  mindst én `#[sqlx::test]`-test der dækker happy path.
- Fejlstier (ugyldigt input, manglende rækker) dækkes med mindst én test pr.
  distinkt fejltype.
- Migrations testes implicit af `#[sqlx::test]` — SQLx kører dem automatisk
  mod en frisk testdatabase. Verificér at alle migrations kører rent ved at
  køre testsuiten mod en tom database.

```rust
#[sqlx::test]
async fn get_user_returns_none_for_unknown_id(pool: PgPool) {
    let result = get_user_by_id(&pool, Uuid::new_v4()).await.unwrap();
    assert!(result.is_none());
}
```

## Konsekvenser

- SQL-fejl (forkert kolonnenavn, manglende tabel) opdages ved kørsel, ikke ved
  compile. `#[sqlx::test]`-integrationstests er det primære safety net og skal
  dække alle query-kald.
- Ingen `sqlx prepare`-step nødvendigt i CI-pipeline.
- Nye kolonner kræver en migration-fil — de køres automatisk af
  `sqlx::migrate!` ved opstart og af `#[sqlx::test]` i tests.
