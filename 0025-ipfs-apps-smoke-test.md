# 0025 — ipfs-apps/chat: smoke test med CAPTCHA-bypass

**Status:** Accepted  
**Dato:** 2026-05-06  
**Projekt:** ipfs-apps/chat  
**Udskilt fra:** ADR-0015  
**Forudsætter:** ADR-0015 (GDPR-sletnings-mønster)

## Kontekst

Post-deploy smoke testen skal verificere GDPR-sletnings-flowet i produktion
efter hvert deploy (ADR-0015). Applikationen bruger Cloudflare Turnstile
CAPTCHA på `/auth/register`, som ikke kan valideres fra et automatiseret
CI-miljø uden en rigtig browser.

Problemet: smoke testen skal oprette en testbruger, men kan ikke gå via
det normale registrerings-endpoint.

## Beslutning

### Direkte DB-indsættelse

Smoke testen (`scripts/smoke-test.sh`) omgår CAPTCHA ved at indsætte
testbrugeren direkte i PostgreSQL med et forudberegnet argon2id-hash:

```
Password: SmokeTest99!
Hash: $argon2id$v=19$m=19456,t=2,p=1$JD46E95F0HZ6fa5RAezmjQ$ucT1cnSgRmpdOgc7YEGC9MhNhv7kJvDd/4jtSm3/fgQ
```

Hashet er genereret med `Argon2::default()` fra `argon2`-craten med
parametrene `m=19456, t=2, p=1` — identiske med hvad `auth.rs` bruger.

Hvis argon2-parametrene ændres i `src/auth.rs` skal hashet regenereres:

```rust
// chat/examples/gen_hash.rs
fn main() {
    println!("{}", chat::auth::hash_password("SmokeTest99!"));
}
// cargo run --example gen_hash
```

Testbrugeren indsættes med `email_verified = TRUE` (login kræver ikke
e-mail-verifikering i testen). `trap cleanup EXIT` sikrer oprydning
via `DELETE /auth/me` selv ved fejl.

### Hvad smoke testen dækker

Efter hvert deploy verificeres i rækkefølge:

1. `POST /auth/token` → 200, JWT udstedt
2. `GET /auth/me` → 200, alle GDPR-felter til stede (`id`, `username`, `email`, `email_verified`, `created_at`)
3. `GET /rooms` → 200
4. `DELETE /auth/me` → 204
5. `GET /auth/me` efter sletning → 401

Testen køres som sidste trin i `ansible/playbook.yml` og afbryder deployet
ved fejl.

## Begrundelse

Alternativerne er afvist:
- **Test-CAPTCHA-nøgle i prod:** Svækker CAPTCHA-beskyttelsen — Turnstile
  ville acceptere kendte test-tokens i produktionsmiljøet.
- **`/internal/test-register`-endpoint:** En bagdør der præcis modsiger
  formålet med CAPTCHA.
- **Manuel testning:** Ikke reproducerbart og udelades ved travlhed.

Direkte DB-adgang er kun mulig for Ansible-processen på selve serveren —
ikke for eksterne angribere. Det er en acceptabel løsning givet begrænsningen.

## Konsekvenser

- Ændringer til argon2-parametrene i `src/auth.rs` kræver opdatering af
  hashet i `scripts/smoke-test.sh`.
- Smoke testen er afhængig af PostgreSQL-adgang fra Ansible-processen.
- Hvis CAPTCHA erstattes med en løsning der understøtter test-tokens,
  bør denne ADR revideres og den direkte DB-indsættelse fjernes.
