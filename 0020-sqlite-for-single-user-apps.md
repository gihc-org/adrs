# 0020 — SQLite frem for PostgreSQL til enkeltbruger-apps

**Status:** Accepted  
**Dato:** 2026-05-09

## Kontekst

Personlige produktivitetsværktøjer og enkeltbruger-apps har typisk:
- Én bruger (eller en lille betroet gruppe)
- Ingen krav om samtidige skrivninger fra mange klienter
- Simpelt deployment-krav

Den oplagte løsning er at genbruge en eksisterende PostgreSQL-instans, men
det introducerer en ekstern service-afhængighed der ikke er nødvendig.

En grafdatabase (Neo4j, Dgraph) kan overvejes ved komplekse graf-relationer,
men har typisk JVM-overhead og et nyt query-sprog.

## Beslutning

Brug **SQLite** via SQLx med `create_if_missing(true)`.

Databasefilen opbevares i et Docker-volume (f.eks. `/app/data/app.db`).

## Begrundelse

**SQLite frem for PostgreSQL:**
- Ingen separat database-service — binæren og databasefilen udgør hele
  applikationen. Simplere Docker-setup, ingen healthcheck-afhængighed.
- SQLite's WAL-mode håndterer én skriver og mange læsere, hvilket dækker
  enkeltbruger-scenariet.
- Backup reduceres til kopiering af én fil.
- SQLx understøtter SQLite med samme API som PostgreSQL.

**SQLite frem for grafdatabase:**
- Træstrukturer og typede krydsreferencer modelleres tilfredsstillende med
  adjacency list + edges-tabel i SQLite (se ADR-0021).
- SQLite's rekursive CTEs (`WITH RECURSIVE`) dækker traversal-behov.
- Ingen ekstern service, intet nyt query-sprog.

## Testkrav

SQLite's `":memory:"`-database giver hurtige, isolerede integrationstests
uden diskadgang:

```rust
#[sqlx::test]
async fn insert_and_retrieve_node(pool: SqlitePool) {
    // #[sqlx::test] spinner en frisk in-memory database op med migrations
    let id = insert_node(&pool, "Testnode", None).await.unwrap();
    let node = get_node(&pool, id).await.unwrap();
    assert_eq!(node.content, "Testnode");
}
```

Alle query-funktioner skal have mindst én `#[sqlx::test]`-test (se ADR-0002).

## Konsekvenser

- `DATABASE_URL` formatet for SQLite via SQLx er `sqlite:/app/data/app.db`
  med enkelt skråstreg — `sqlite://` med dobbelt skråstreg fortolkes som
  absolut sti og fejler hvis mappen ikke eksisterer.
- Migrationer køres via `sqlx::migrate!("./migrations")` ved opstart.
- Skalering til mange samtidige brugere kræver migration til PostgreSQL,
  men det er ikke et krav for enkeltbruger-apps.
