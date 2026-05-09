# 0021 — Adjacency list + edges-tabel til træ- og DAG-semantik

**Status:** Accepted  
**Dato:** 2026-05-09  
**Projekt:** capture

## Kontekst

`capture`'s datamodel skal understøtte to strukturelle relationer:

1. **Hierarki** — noder organiseres i et træ (én forælder, mange børn)
   til brug i organiser-visningen med drag-and-drop.
2. **Krydsreferencer** — en node kan relatere til en vilkårlig anden node
   med en semantisk type: *inspireret af*, *modsiger*, *relaterer til*.
   Én node kan have flere sådanne relationer til forskellige noder (DAG).

Disse to typer relation har forskellig kardinalitet og semantik og bør
modelleres separat.

## Beslutning

**Træ:** Adjacency list via `parent_id` (nullable) på `nodes`-tabellen.

```sql
CREATE TABLE nodes (
    id         TEXT PRIMARY KEY,
    content    TEXT NOT NULL,
    parent_id  TEXT REFERENCES nodes(id) ON DELETE CASCADE,
    position   INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

**Krydsreferencer:** Separat `edges`-tabel med `kind`-felt.

```sql
CREATE TABLE edges (
    id         TEXT PRIMARY KEY,
    from_id    TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    to_id      TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    kind       TEXT NOT NULL,  -- 'inspireret_af' | 'modsiger' | 'relaterer_til'
    created_at TEXT NOT NULL,
    UNIQUE(from_id, to_id, kind)
);
```

Gyldige `kind`-værdier håndhæves i applikationslaget (ikke en SQL-constraint)
så de kan udvides uden migration.

## Begrundelse

**Adjacency list frem for nested sets eller closure table:**
- Simpelt at læse og skrive; `INSERT`, `UPDATE parent_id` og `DELETE` er
  trivielle operationer.
- Tilstrækkelig for en enkeltbruger-app hvor trædybden er begrænset og
  der ikke forespørges på hele undertræer i ét SQL-kald.
- `position`-feltet giver eksplicit rækkefølge inden for et niveau uden
  at kræve re-nummerering af søskende.

**Separat edges-tabel frem for at genbruge parent_id til alle relationer:**
- `parent_id` er én-til-mange og retningsbestemt opad i hierarkiet.
  Krydsreferencer er mange-til-mange og semantisk anderledes.
- Adskillelsen gør det muligt at slette et hierarki-forhold (drag-and-drop)
  uden at påvirke krydsreferencer, og omvendt.
- `UNIQUE(from_id, to_id, kind)` forhindrer duplikerede relationer.
- `ON DELETE CASCADE` på begge fremmednøgler sikrer at relationer automatisk
  fjernes når en node slettes.

**Applikationsvalidering af kind frem for CHECK-constraint:**
- `CHECK (kind IN ('inspireret_af', 'modsiger', 'relaterer_til'))` ville
  kræve en ny migration for hvert nyt relationstype.
- Validering i Rust (`const VALID_KINDS: &[&str]`) returnerer HTTP 422 ved
  ugyldigt input og er let at udvide.

## Konsekvenser

- Klienten modtager noder og edges som to separate lister og bygger træet
  i JavaScript. Ingen rekursiv SQL er nødvendig på nuværende skala.
- DAG-traversal (f.eks. "alle noder der er inspireret af denne node, transitiv")
  kan implementeres med SQLite's `WITH RECURSIVE` hvis behovet opstår.
- Tilføjelse af en ny `kind`-værdi kræver kun ændring i `VALID_KINDS`-konstanten
  og en frontend-opdatering — ingen migration.
