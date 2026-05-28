# 0006 — In-memory broadcast channel pr. rum til WebSocket fan-out

**Status:** Accepted  
**Dato:** 2026-05-05

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
WebSocket-forbindelse. Kanalen har kapacitet `BROADCAST_CAPACITY` (default 256)
— definer denne som en navngivet konstant, ikke en magic number.

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

## Testkrav

**`get_or_create_sender` — concurrent oprettelse:**

Double-checked locking er en klassisk race condition-risiko. Test at concurrent
kald til samme rum-id ikke opretter to kanaler:

```rust
#[tokio::test]
async fn concurrent_get_or_create_returns_same_sender() {
    let map: RoomMap = Arc::new(RwLock::new(HashMap::new()));
    let handles: Vec<_> = (0..16).map(|_| {
        let m = Arc::clone(&map);
        tokio::spawn(async move { get_or_create_sender(&m, "room-1").await })
    }).collect();
    let results: Vec<_> = futures::future::join_all(handles).await;
    // Alle 16 kald skal returnere en Sender der peger på samme kanal
    let first_id = results[0].as_ref().unwrap().same_channel(results[1].as_ref().unwrap());
    assert!(first_id);
}
```

**`RecvError::Lagged` — slow client:**

Test at en slow receiver der er bagud ikke blokerer hurtige receivers og at
`RecvError::Lagged` udløses korrekt ved overflow:

```rust
#[tokio::test]
async fn lagged_receiver_gets_error_not_panic() {
    let (tx, mut rx) = broadcast::channel(4);
    for i in 0..8 { tx.send(i.to_string()).unwrap(); }
    assert!(matches!(rx.recv().await, Err(RecvError::Lagged(_))));
}
```

**OWASP WebSocket:** Verificér at uautoriserede klienter ikke kan subscribe
på en kanal — WS-connectet skal validere token inden `get_or_create_sender`
kaldes (se ADR-0007).
