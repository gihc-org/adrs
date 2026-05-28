# 0021 — Adjacency list + edges-tabel til træ- og DAG-semantik

**Status:** Accepted  
**Dato:** 2026-05-09

## Kontekst

En datamodel skal understøtte to strukturelle relationer:

1. **Hierarki** — noder i et træ (én forælder, mange børn) til brug i
   f.eks. en organiser-visning med drag-and-drop.
2. **Krydsreferencer** — en node relaterer til en vilkårlig anden node
   med en semantisk type: *inspired_by*, *contradicts*, *relates_to*.
   Én node kan have flere sådanne relationer (DAG).

Disse to typer relation har forskellig kardinalitet og semantik.

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
    kind       TEXT NOT NULL,  -- 'inspired_by' | 'contradicts' | 'relates_to'
    created_at TEXT NOT NULL,
    UNIQUE(from_id, to_id, kind)
);
```

Gyldige `kind`-værdier håndhæves i applikationslaget (ikke SQL `CHECK`-constraint)
så de kan udvides uden migration. Brug engelske konstantnavne for
fremtidssikring og kompatibilitet med internationale bidragydere.

## Begrundelse

**Adjacency list frem for nested sets eller closure table:**
- Simpelt at læse og skrive — `INSERT`, `UPDATE parent_id` og `DELETE`
  er trivielle operationer.
- `position`-feltet giver eksplicit rækkefølge inden for et niveau.
- Tilstrækkeligt for enkeltbruger-apps og begrænsede trædybder.

**Separat edges-tabel:**
- `parent_id` er én-til-mange og retningsbestemt opad. Krydsreferencer er
  mange-til-mange og semantisk anderledes — de bør ikke blandes.
- `UNIQUE(from_id, to_id, kind)` forhindrer duplikerede relationer.
- `ON DELETE CASCADE` sikrer at relationer automatisk fjernes når en node slettes.

**Applikationsvalidering af kind:**
- `CHECK (kind IN (...))` kræver en ny migration for hvert nyt relationstype.
- Validering i applikationslaget returnerer HTTP 422 ved ugyldigt input og
  er let at udvide.

## Testkrav

- [ ] Test at `ON DELETE CASCADE` på `parent_id` faktisk sletter undertræet
- [ ] Test at `ON DELETE CASCADE` på `edges.from_id` og `edges.to_id` fjerner
  relationer når en node slettes
- [ ] Test at `UNIQUE(from_id, to_id, kind)` afviser duplikerede relationer
- [ ] Test at ugyldig `kind`-værdi returnerer 422 fra API'et

```rust
#[sqlx::test]
async fn delete_parent_cascades_to_children(pool: SqlitePool) {
    let parent = insert_node(&pool, "Parent", None).await.unwrap();
    let child = insert_node(&pool, "Child", Some(parent)).await.unwrap();
    delete_node(&pool, parent).await.unwrap();
    assert!(get_node(&pool, child).await.unwrap().is_none());
}
```

## Konsekvenser

- Klienten modtager noder og edges som to separate lister og bygger træet
  lokalt. Ingen rekursiv SQL er nødvendig på nuværende skala.
- DAG-traversal kan implementeres med `WITH RECURSIVE` hvis behovet opstår.
- Tilføjelse af ny `kind`-værdi kræver kun ændring i valideringskonstanten
  og en frontend-opdatering — ingen migration.
