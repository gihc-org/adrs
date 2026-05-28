# 0003 — rustls frem for native-tls i Rust+Docker-projekter

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

Rust-projekter med udgående HTTP-kald (via `reqwest`) og databaseforbindelser
over TLS (via `SQLx`) kræver en TLS-backend. To muligheder:

- **`native-tls`:** Kalder systemets TLS-bibliotek (OpenSSL på Linux).
  Kræver `libssl-dev` og `pkg-config` i build-miljøet.
- **`rustls-tls`:** Pure-Rust TLS-implementering. Ingen systemafhængigheder
  ud over Rust-toolchain.

## Beslutning

Brug **`rustls-tls`** som feature i `reqwest` og `SQLx`.

```toml
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
sqlx    = { version = "0.8",  features = ["runtime-tokio-rustls", ...] }
```

## Begrundelse

- **Simplere Dockerfile:** Ingen `apt-get install libssl-dev pkg-config` i
  builder-image. `rust:1-slim` er tilstrækkelig.
- **Ingen OpenSSL-versionsproblemer:** OpenSSL-versionen i builder-image behøver
  ikke matche runtime-image. rustls er statisk linket ind i binæren.
- **Reproducerbarhed:** Build-output afhænger kun af Rust-toolchain, ikke
  systembiblioteker.
- **Hurtigere CI-builds:** Færre system-pakker at installere.

## Konsekvenser

- Binæren er lidt større (rustls er statisk linket) — typisk < 1 MB forskel.
- Systemcertifikater bruges via `rustls-native-certs` til at validere
  serverens certifikat. Tilføj eksplicit i `Cargo.toml` hvis det ikke
  trækkes ind som transitiv afhængighed:
  ```toml
  rustls-native-certs = "0.7"
  ```
- Edge cases med eksotiske TLS-features (PKCS#11, hardware tokens) understøttes
  ikke af rustls — `native-tls` er fallback i de tilfælde.
