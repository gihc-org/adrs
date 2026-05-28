# 0001 — Axum som web-framework frem for Actix-web

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

Rust-backend med WebSocket-support og REST-endpoints. De to dominerende
frameworks i Rust-webøkosystemet er Axum (Tokio-projektet) og Actix-web.
Begge understøtter async/await og har god WebSocket-support.

## Beslutning

Vi bruger **Axum 0.7** med `ws`-featuren.

## Begrundelse

- **Extractor-model:** Axum's `FromRequest`/`FromRequestParts`-traits gør det
  muligt at udtrykke afhængigheder (database, config, auth) direkte i
  handler-signaturen. Det reducerer boilerplate og gør handlers testbare i
  isolation — en extractor kan mockes eller erstattes i tests uden at ændre
  handleren.
- **Tokio-native:** Axum er bygget oven på Tower og Tokio og følger samme
  middleware-model. Ingen "actor"-abstraktion der skjuler Tokio-runtimen.
- **Vedligeholdelsesstatus:** Axum vedligeholdes aktivt af Tokio-projektet og
  er i dag det mest brugte Rust web-framework i nye projekter.
- **Tower-middleware:** `tower-http`-crates (CORS, tracing, komprimering,
  rate limiting) virker direkte med Axum uden adapter-lag.

## Testkrav

Axum's extractor-model er en direkte fordel for testbarhed. Udnyt den:

- Handler-funktioner testes i isolation ved at konstruere extractors direkte
  — ingen fuld HTTP-server nødvendig for unit tests.
- Integrationstests spinner en `TestServer` op via `axum::test` eller
  `axum_test::TestServer` og rammer rigtige endpoints.

```rust
// Integration test — ingen mock, rigtig router
#[sqlx::test]
async fn get_rooms_requires_auth(pool: PgPool) {
    let app = build_app(test_state(pool).await);
    let resp = app.oneshot(
        Request::get("/rooms").body(Body::empty()).unwrap()
    ).await.unwrap();
    assert_eq!(resp.status(), StatusCode::UNAUTHORIZED);
}
```

## Konsekvenser

- WebSocket-handler er en simpel `async fn` med `WebSocketUpgrade`-extractor.
- CORS konfigureres via `tower_http::cors::CorsLayer`.
- Auth implementeres som en plain `async fn` frem for `FromRequestParts`
  (se ADR-0005 for begrundelse — Rust-compiler lifetime-begrænsning).
