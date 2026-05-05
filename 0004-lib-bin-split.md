# 0004 — Lib + bin-split for testbarhed

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

En Rust-binary kan struktureres som ét binært crate (alt i `main.rs`) eller
som et library-crate med en tynd binary-wrapper. Integrationstests i
`tests/`-mappen kan kun importere library-crates — ikke binary-crates.

## Beslutning

`chat/src/lib.rs` er library-roden og ejer alle modul-deklarationer og
eksporterer `AppState`, `Config`, `RoomMap`, `build_app` og `build_cors`.
`src/main.rs` er et tyndt binært entry-point der kalder ind i lib.

```
chat/
  src/
    lib.rs      ← library root: alle moduler, build_app(), AppState
    main.rs     ← binary: læser config, opretter pool, kalder build_app()
  tests/
    api.rs      ← integration tests: `use chat::build_app;`
```

## Begrundelse

- **Integrationstests kan importere crate:** `tests/api.rs` kan skrive
  `use chat::{build_app, AppState}` og spinne en test-server op med en rigtig
  database via `#[sqlx::test]`.
- **Ingen duplikering af setup-kode:** Router-konfiguration, CORS og state
  lever ét sted (`lib.rs`) og bruges af både produktion og tests.
- **Klar separation:** `main.rs` håndterer kun I/O (læs env, bind port) —
  al applikationslogik er testbar uden at starte en rigtig server.

## Konsekvenser

- `Cargo.toml` skal deklarere både `[lib]` og `[[bin]]` med eksplicitte `path`.
- Nye moduler tilføjes i `lib.rs` (`pub mod nyt_modul;`) — ikke i `main.rs`.
- `pub`-visibility i lib er nødvendig for alt der bruges af integrationstests.
