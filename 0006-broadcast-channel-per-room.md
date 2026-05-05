# 0006 — In-memory broadcast channel pr. rum til WebSocket fan-out

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

WebSocket-chat kræver fan-out: en besked sendt af én klient skal leveres til
alle andre klienter i samme rum. Der er flere måder at implementere dette på:

- **In-memory broadcast channel:** Tokio's `broadcast::channel` — simpel,
  ingen I/O, server-intern tilstand.
- **Database polling:** Klienter poller PostgreSQL for nye beskeder.
- **Redis pub/sub:** Ekstern message broker — understøtter horisontalt scale.

## Beslutning

Vi bruger **én `tokio::sync::broadcast::Sender<String>` pr. rum**, gemt i en
`Arc<RwLock<HashMap<String, Sender<String>>>>` (`RoomMap`) i `AppState`.

```rust
pub type RoomMap = Arc<RwLock<HashMap<String, broadcast::Sender<String>>>>;
```

Et nyt rum oprettes on-demand med `get_or_create_sender()` ved første
WebSocket-forbindelse. Kanalen har kapacitet 256 beskeder.

## Begrundelse

- **Simplicitet:** Ingen ekstern afhængighed (Redis). Kun Tokio-primitiver.
- **Lav latens:** In-memory fan-out er hurtigere end database-polling eller
  netværkskald til en broker.
- **Tilstrækkelig kapacitet:** Single-server deployment — horisontalt scale er
  ikke et krav for dette projekt.
- **Slow-client isolering:** `RecvError::Lagged` disconnecter slow clients
  automatisk uden at blokere hurtige klienter.

## Konsekvenser

- **Server-restart dropper alle aktive forbindelser.** Klienter skal reconnecte.
  Det er acceptabelt for dette project, men bør dokumenteres i frontend (auto-reconnect).
- **Ingen persistens på tværs af forbindelser i hukommelsen.** Beskeder gemmes
  i PostgreSQL af WebSocket-handleren — historik er tilgængelig via
  `GET /rooms/:id/messages`.
- **Horisontalt scale kræver Redis.** Hvis projektet skal køre på flere
  servere, skal `RoomMap` erstattes af Redis pub/sub.
- Double-checked locking i `get_or_create_sender` er nødvendig for at undgå
  race condition ved concurrent oprettelse af samme rum.
