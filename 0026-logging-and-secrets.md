# 0026 — Logging-strategi og secrets-håndtering

**Status:** Accepted  
**Dato:** 2026-05-28

## Kontekst

Logging og secrets-håndtering er tæt koblede: den hyppigste kilde til
lækage af hemmeligheder er at de ender i logs — som query-parametre (CWE-598,
se ADR-0007), i fejlbeskeder eller i debug-output. Begge emner behandles her
fordi reglen "log ikke hemmeligheder" kun kan håndhæves hvis det er klart
hvad der er en hemmelighed, og hvad der bør logges.

## Beslutning

### Hvad der logges

Log på det laveste niveau der giver nyttig information uden at eksponere
brugerdata:

| Niveau | Hvornår | Eksempler |
|--------|---------|-----------|
| `ERROR` | Uventet fejl der kræver handling | Databaseforbindelse mistet, panic |
| `WARN`  | Gendannelig fejl eller anomali | Ukendt config-nøgle, retry-forsøg |
| `INFO`  | Vigtige begivenheder i happy path | Server startet, deploy fuldført |
| `DEBUG` | Udviklingsdiagnostik | Ikke i produktion |
| `TRACE` | Detaljeret flow-tracking | Aldrig i produktion |

Produktion kører på `INFO`. `DEBUG` og `TRACE` aktiveres kun lokalt og
aldrig i staging eller prod.

**Struktureret logging foretrækkes:** Brug et struktureret format (JSON eller
key=value) frem for fritekst. Det gør logs søgbare og kompatible med log-
aggregatorer (Loki, Elasticsearch, CloudWatch).

```rust
// Foretrukket
tracing::info!(user_id = %id, room = %room_id, "user joined room");

// Undgå
tracing::info!("user {} joined room {}", id, room_id);
```

### Hvad der aldrig logges

Følgende må aldrig fremgå af logs — hverken i prod, staging eller CI:

- Passwords og password-hashes
- JWT-tokens og API-nøgler (se ADR-0007 for URL-sanitering)
- Session-cookies og CSRF-tokens
- Persondata: e-mail, brugernavn, IP-adresse (medmindre eksplicit krævet af audit-log)
- Kryptografiske nøgler og seeds
- Vault-passwords og infrastruktur-credentials

**Fejlbeskeder fra autentificering** skal være generiske: "ugyldigt login" —
ikke "e-mail ikke fundet" eller "forkert adgangskode". Specifik information
hjælper angribere.

### Log-sanitering

URL'er med query-parametre saniteres inden logning — fjern parametre der
typisk indeholder secrets (`token`, `key`, `secret`, `password`, `api_key`):

```rust
fn sanitize_url(url: &str) -> String {
    // Erstatter værdier af kendte secret-parametre med [REDACTED]
    regex.replace_all(url, "${param}=[REDACTED]").to_string()
}
```

Test at sanitering virker (se ADR-0007 for eksempel).

---

### Secrets-håndtering

**Regel: ingen hemmeligheder i kildekode eller git-historik.**

#### Lokalt

- Secrets gemmes i `.env`-filen (gitignored).
- `.env.example` committes med pladsholdere og beskrivelse af hvert felt.
- Nye secrets tilføjes til `.env.example` i samme commit som de bruges.

```bash
# .env.example
DATABASE_URL=postgres://user:password@localhost:5432/dbname
JWT_SECRET=minimum-32-tegn-tilfaeldig-streng
ACME_EMAIL=din@email.dk
```

#### CI/CD

- Secrets injiceres fra CI-systemets eget secret store som miljøvariabler.
- Aldrig hardkodet i pipeline-konfigurationsfiler.
- CI-logs inspiceres for accidental exposure efter ændringer til secrets-brug.

#### Produktion

- Ansible Vault til filer der indeholder secrets (`group_vars/all/vault.yml`).
- Vault-filen krypteres med `ansible-vault encrypt` og committes krypteret.
- Vault-password opbevares separat fra repoet — aldrig i git.
- Rotation: ved mistanke om kompromittering roteres secrets straks og
  gamle tokens invalideres.

#### Validering ved opstart

Applikationen validerer ved opstart at alle påkrævede miljøvariabler er sat
og ikke-tomme. En manglende hemmelighed giver en klar fejlbesked og stopper
opstart — aldrig en stille fejl ved første brug:

```rust
fn require_env(key: &str) -> String {
    std::env::var(key)
        .unwrap_or_else(|_| panic!("Påkrævet miljøvariabel mangler: {key}"))
}
```

Sikkerhedskritiske variabler (JWT_SECRET, DATABASE_URL) valideres altid.
Se ADR-0012 for feature flags via tomme env-variabler — brug dette mønster
kun til ikke-sikkerhedskritiske flags.

## Begrundelse

**Struktureret logging:** Fritekst-logs er svære at parse ved hændelser.
Strukturerede logs med faste felter gør det muligt at filtrere på `user_id`
eller `room` på tværs af tusindvis af linjer på sekunder.

**Ingen persondata i logs:** GDPR kræver dataminimering (ADR-0014). Logs er
typisk bredere tilgængelige end databaser og gemmes længere. Persondata i logs
er et GDPR-problem der er svært at rydde op i efterhånden.

**Opstartsvalidering:** En manglende DATABASE_URL opdages ved deploy, ikke
ved første databasekald fra en bruger midt om natten.

## Konsekvenser

- Log-sanitering af URL query-parametre er et krav (ikke anbefaling) for
  alle endpoints der modtager tokens i URL — se ADR-0007.
- `.env.example` skal altid være synkroniseret med de faktisk brugte variabler.
- Ansible Vault-password distribueres out-of-band til alle der skal deploye.
- Debug-logging i produktion kræver eksplicit beslutning og fjernes igen
  efter diagnostik.
