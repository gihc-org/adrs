# 0003 — rustls frem for native-tls

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Projektet bruger `reqwest` til udgående HTTP-kald (Cloudflare Turnstile,
Resend). reqwest understøtter to TLS-backends:

- **`native-tls`:** Kalder på systemets TLS-bibliotek (OpenSSL på Linux).
  Kræver `libssl-dev` og `pkg-config` i build-miljøet.
- **`rustls-tls`:** Pure-Rust TLS-implementering. Ingen systemafhængigheder
  udover Rust-toolchain.

## Beslutning

Vi bruger **`rustls-tls`** som feature i reqwest og SQLx.

```toml
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
sqlx    = { version = "0.8",  features = ["runtime-tokio-rustls", ...] }
```

## Begrundelse

- **Simplere Dockerfile:** Ingen `apt-get install libssl-dev pkg-config` i
  builder-image. `rust:1-slim` er tilstrækkelig.
- **Hurtigere builds:** Færre system-pakker at installere og færre linkede
  native-biblioteker.
- **Ingen OpenSSL-versionsproblemer:** OpenSSL-versionen i builder-image behøver
  ikke matche runtime-image. rustls er statisk linket ind i binæren.
- **Reproducerbarhed:** Build-output afhænger kun af Rust-toolchain, ikke
  systembiblioteker.

## Konsekvenser

- Binæren er større (rustls er statisk linket), men forskellen er typisk < 1 MB.
- Systemcertifikater bruges stadig (via `rustls-native-certs`) til at validere
  serverens certifikat — det kræver ingen ekstra konfiguration.
- Eventuelle edge cases med eksotiske TLS-features (PKCS#11, hardware tokens)
  understøttes ikke af rustls.
