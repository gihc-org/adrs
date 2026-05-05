# 0001 — Axum som web-framework frem for Actix-web

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Projektet er en Rust-backend med WebSocket-support og REST-endpoints. De to
dominerende frameworks i Rust-webøkosystemet er Axum (Tokio-projektet) og
Actix-web. Begge understøtter async/await og har god WebSocket-support.

## Beslutning

Vi bruger **Axum 0.7** med `ws`-featuren.

## Begrundelse

- **Extractor-model:** Axum's `FromRequest`/`FromRequestParts`-traits gør det
  muligt at udtrykke afhængigheder (database, config, auth) direkte i
  handler-signaturen. Det reducerer boilerplate og gør handlers testbare i
  isolation.
- **Tokio-native:** Axum er bygget oven på Tower og Tokio og følger samme
  middleware-model. Ingen "actor"-abstraktion der skjuler Tokio-runtimen.
- **Vedligeholdelsesstatus:** Axum vedligeholdes aktivt af Tokio-projektet og
  er i dag det mest brugte Rust web-framework i nye projekter.
- **Tower-middleware:** `tower-http`-crates (CORS, tracing, komprimering) virker
  direkte med Axum uden adapter-lag.

## Konsekvenser

- WebSocket-handler er en simpel `async fn` med `WebSocketUpgrade`-extractor.
- CORS konfigureres via `tower_http::cors::CorsLayer` — se `lib.rs`.
- Rust 1.88 strammede lifetime-regler for async fns i traits, som brød vores
  forsøg på at bruge `FromRequestParts` til auth (se ADR-0005).
