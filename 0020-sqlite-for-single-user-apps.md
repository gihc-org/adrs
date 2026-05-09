# 0020 — SQLite frem for PostgreSQL til enkeltbruger-apps

**Status:** Accepted  
**Dato:** 2026-05-09  
**Projekt:** capture

## Kontekst

`capture` er et personligt produktivitetsværktøj med én bruger. Datamodellen
er en træstruktur af noder med tilhørende krydsreferencer. Der er intet krav
om samtidige skrivninger fra flere klienter — én bruger tilgår appen fra
forskellige enheder, men aldrig simultant.

ipfs-apps bruger PostgreSQL og kører allerede på samme VPS. Den oplagte
genanvendelse ville være at tilslutte `capture` til den eksisterende
Postgres-instans.

En grafdatabase (Neo4j, Dgraph) blev ligeledes overvejet, da datamodellen
indeholder relationer med semantisk type.

## Beslutning

Vi bruger **SQLite** via SQLx med `create_if_missing(true)`.

Databasefilen opbevares i et Docker-volume (`/app/data/capture.db`).

## Begrundelse

**SQLite frem for PostgreSQL:**
- Ingen separat database-service; binæren og databasefilen udgør hele
  applikationen. Simplere Docker-setup, ingen healthcheck-afhængighed.
- SQLite's write-ahead log (WAL) håndterer læsning fra én enhed mens en
  anden skriver, hvilket dækker enkeltbruger-scenariet fuldt ud.
- Backup reduceres til kopiering af én fil.
- SQLx understøtter SQLite med samme API som PostgreSQL — ingen ekstra
  læringsomkostning.

**SQLite frem for grafdatabase:**
- Datamodellen er primært et træ (noder med én `parent_id`) med et begrænset
  antal typede krydsreferencer. Disse modelleres tilfredsstillende med en
  `edges`-tabel i SQLite (se ADR-0019).
- En grafdatabase (Neo4j) ville kræve en separat service, JVM-overhead og
  et nyt query-sprog (Cypher) — uforholdsmæssigt for et personligt værktøj.
- SQLite's rekursive CTEs (`WITH RECURSIVE`) dækker de traversal-behov der
  måtte opstå i en træ- eller DAG-struktur.

## Konsekvenser

- `DATABASE_URL` har formatet `sqlite:/app/data/capture.db` (ikke `sqlite://`
  med dobbelt skråstreg — SQLx fortolker `//` som absolut sti og fejler hvis
  mappen ikke eksisterer).
- Migrationer køres via `sqlx::migrate!("./migrations")` ved opstart.
- Skalering til flere samtidige brugere ville kræve migration til PostgreSQL,
  men det er ikke et krav for dette projekt.
