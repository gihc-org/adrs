# 0005 — authenticate som plain async fn frem for FromRequestParts

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Axum tilbyder `FromRequestParts`-trait til at udtrykke per-request afhængigheder
direkte i handler-signaturer (f.eks. `async fn me(user: User) -> ...`). Det er
idiomatisk Axum, men kræver en `async fn`-implementering af en trait, som Rust
1.88 har strammet lifetime-reglerne for.

## Beslutning

Auth implementeres som en plain `async fn authenticate(state, headers) -> Result<User, ApiError>`,
der kaldes eksplicit øverst i hver beskyttet handler:

```rust
pub async fn me(State(state): State<AppState>, headers: HeaderMap) -> ... {
    let user = authenticate(&state, &headers).await?;
    ...
}
```

## Begrundelse

Rust 1.88 indførte strengere lifetime-checks for `async fn` i traits. Vores
`FromRequestParts`-implementering returnerede en `Future` der lånte fra
`&self` og `&Parts` med overlappende lifetimes — et mønster compileren nu
afviser uden eksplicit HRTB-annotation (`for<'a>`), som er kompleks og
fejlprone at skrive korrekt.

Den plain `async fn`-tilgang:
- Compilerer med nuværende og fremtidige Rust-versioner uden lifetime-tricks
- Er eksplicit — det er tydeligt i handler-koden at auth foregår
- Er nem at mocke i tests (pass en stub `AppState`)

## Konsekvenser

- Alle beskyttede handlers skal kalde `authenticate(&state, &headers).await?`
  manuelt — det kan glemmes. Code review skal verificere dette.
- `FromRequestParts` kan genindføres hvis Axum eller Rust løser
  lifetime-problemet i en fremtidig version.
- WebSocket-handleren bruger en query-parameter i stedet for header til auth
  (se ADR-0007) — `authenticate` bruges ikke direkte der.
