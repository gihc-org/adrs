# 0015 — GDPR-sletningstrategi og post-deploy smoke test

**Status:** Accepted  
**Dato:** 2026-05-06  
**Projekt:** ipfs-apps/chat

## Kontekst

ADR-0014 besluttede at implementere `DELETE /auth/me` (ret til sletning) og
udvide `GET /auth/me` (ret til indsigt). Denne ADR dokumenterer de konkrete
implementeringsbeslutninger der ikke var åbenlyse og som fremtidige ændringer
skal forstå.

### To ikke-trivielle beslutninger

**1. Sletningsrækkefølge og cascade**

`DELETE /auth/me` skal slette brugeren og alle tilknyttede data. Databaseskemaet
har `ON DELETE CASCADE` på `messages` og `room_members` — disse slettes
automatisk når brugeren slettes. DM-rum derimod har ingen FK fra `rooms` til
`users`, fordi et rum kan have flere medlemmer og ikke tilhører én bruger.
Cascade vil derfor ikke rydde op i DM-rum automatisk.

Konsekvensen er at handleren skal slette DM-rum eksplicit *inden* brugeren
slettes, ellers efterlades tomme rum i databasen uden mulighed for oprydning.

**2. Smoke test bypass af CAPTCHA**

Applikationen bruger Cloudflare Turnstile CAPTCHA på `/auth/register`. I
produktion er CAPTCHA-hemmeligheden en rigtig nøgle, og der er ingen API-
endpoint der kan oprette brugere uden om validering. Det er ikke muligt at
autentificere en Turnstile-token fra et automatiseret CI-miljø uden en rigtig
browser.

## Beslutning

### Sletningsrækkefølge

`DELETE /auth/me` sletter i denne rækkefølge:

1. Slet eksplicit alle rum hvor `is_dm = TRUE` og brugeren er medlem.
   DM-rum har ingen værdi uden begge parter og efterlades ikke som tomme rum.
2. Slet brugeren. `ON DELETE CASCADE` håndterer herefter:
   - `messages` (alle beskeder brugeren har skrevet i alle rum)
   - `room_members` (alle rumsmedlemskaber)

Offentlige rum (`is_dm = FALSE`) slettes ikke — de tilhører ikke én bruger.

### Smoke test via direkte DB-indsættelse

Post-deploy smoke testen i `scripts/smoke-test.sh` omgår CAPTCHA ved at
indsætte testbrugeren direkte i PostgreSQL med et forudberegnet argon2id-hash:

```
Password: SmokeTest99!
Hash: $argon2id$v=19$m=19456,t=2,p=1$JD46E95F0HZ6fa5RAezmjQ$ucT1cnSgRmpdOgc7YEGC9MhNhv7kJvDd/4jtSm3/fgQ
```

Hashet er genereret med `Argon2::default()` fra Rust-crate'en `argon2` —
samme parametre som applikationen bruger: `m=19456, t=2, p=1`. Hvis
parametrene ændres i `auth.rs` skal hashet regenereres:

```rust
// Tilføj midlertidigt til chat/examples/gen_hash.rs:
fn main() {
    println!("{}", chat::auth::hash_password("SmokeTest99!"));
}
// cargo run --example gen_hash
```

Testbrugeren indsættes med `email_verified = TRUE` så login er muligt uden
email-verifikering. Brugeren slettes af testen selv via `DELETE /auth/me` —
`trap cleanup EXIT` sikrer oprydning selv ved fejl.

### Hvad smoke testen dækker

Testen verificerer efter hvert deploy:

1. `POST /auth/token` → 200, JWT udstedt
2. `GET /auth/me` → 200, alle GDPR-felter til stede (id, username, email, email_verified, created_at)
3. `GET /rooms` → 200
4. `DELETE /auth/me` → 204
5. `GET /auth/me` efter sletning → 401

Testen kører som sidste trin i `ansible/playbook.yml` og afbryder deployet
hvis et check fejler.

## Begrundelse

**Eksplicit sletning af DM-rum:** Uden dette efterlades forældreløse rum i
databasen med `room_members`-rækker der peger på ikke-eksisterende brugere —
et dataintegritetsproblem der er svært at opdage og rette i efterhånden.

**Direkte DB-indsættelse i smoke test:** Alternativerne er uacceptable:
- En separat test-CAPTCHA-nøgle i prod ville kræve at Turnstile-valideringen
  accepterer kendte test-tokens — det svækker CAPTCHA-beskyttelsen i prod.
- Et `/internal/test-register`-endpoint uden CAPTCHA er en sikkerhedsrisiko
  der præcis modsiger formålet med CAPTCHA.
- Manuel testning efter hvert deploy er ikke reproducerbart og udelades.

Direkte DB-adgang er kun mulig for deploymentprocessen (Ansible på serveren),
ikke for eksterne angribere — det er derfor en acceptabel løsning.

## Konsekvenser

- `DELETE /auth/me`-handleren skal altid slette DM-rum *inden* brugeren
  slettes. Ændringer til sletningslogikken skal huske begge trin.
- Hvis argon2-parametrene ændres i `src/auth.rs` skal smoke-testens hash
  regenereres og opdateres i `scripts/smoke-test.sh`.
- Smoke testen er afhængig af at PostgreSQL-containeren er tilgængelig fra
  Ansible-processen — dette er garanteret da Ansible kører på selve serveren.
