# 0014 — GDPR og CIS Docker Benchmark som tilbagevendende standarder

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps og fremtidige projekter

## Kontekst

OWASP (ADR-0013) dækker applikationssikkerhed, men to yderligere standarder
er direkte relevante for dette projekt og bør følges systematisk:

1. **GDPR** — projektet gemmer persondata (email, brugernavn) på brugere der
   befinder sig i EU. Som dansk projekt er GDPR ikke valgfrit.
2. **CIS Docker Benchmark** — konkrete, målbare hærdningsanbefalinger for
   Docker-containere og -compose-opsætninger.

**12-Factor App** er udeladt fra denne ADR: projektet følger allerede de
relevante faktorer (config via env vars, stateless processer, explicit
dependencies via Cargo.toml). Faktor 11 (logs som streams) er det eneste
udestående punkt og er ikke kritisk for projektets nuværende størrelse.

## Beslutning

### GDPR

Nye features der berører persondata skal vurderes ud fra følgende principper:

**Dataminimering:** Gem kun det der er nødvendigt for featuren. Et
brugernavn og en email er nødvendige for auth og verifikation. Fødselsdato,
telefonnummer og lignende er ikke nødvendige medmindre en konkret feature
kræver det.

**Ret til sletning (`DELETE /auth/me`):** Skal implementeres. Sletning skal
cascade til tilknyttede data (beskeder) — dette er allerede sikret via
`ON DELETE CASCADE` i databaseskemaet for messages.

**Ret til indsigt (`GET /auth/me`):** Skal returnere alle gemte felter for
brugeren, ikke kun id og brugernavn.

**Oplysningspligt:** Brugere skal oplyses om hvad der gemmes og hvorfor —
minimum en privacy policy-side i frontend.

**Verifikationstoken er persondata:** Behandles som persondata og slettes
straks ved verifikation og ved sletning af konto.

### CIS Docker Benchmark

Nye services i `docker-compose.yml` skal som udgangspunkt have:

```yaml
security_opt:
  - no-new-privileges:true
```

Backend-containere der ikke skriver til filsystem skal have:

```yaml
read_only: true
```

Nye Dockerfiles til applikations-services (ikke databaser og proxies) skal
køre som non-root:

```dockerfile
RUN useradd -m appuser
USER appuser
```

Resource limits sættes ved kendte workloads:

```yaml
deploy:
  resources:
    limits:
      memory: 256m
      cpus: "0.5"
```

Image-scanning (`docker scout cves` eller `trivy`) køres mod bygget image
ved mistanke om sårbare afhængigheder eller ved større dependency-opdateringer.

## Begrundelse

**GDPR:** Bøder for overtrædelse kan udgøre op til 4% af global omsætning
eller 20 mio. EUR. Selv for et lille projekt er de grundlæggende krav
(sletning, indsigt, oplysning) enkle at implementere og bør ikke udskydes.

**CIS Docker Benchmark:** En container der kører som root og kompromitteres
giver angriberen root-adgang til host-systemet via container escape. Non-root,
`no-new-privileges` og `read_only` er lav-effort mitigations der eliminerer
en hel klasse af angreb. CIS Benchmark er gratis, velvedligeholdt og
specifik nok til at give konkrete tjekpunkter frem for abstrakte principper.

## Konsekvenser

- `DELETE /auth/me` er et nyt endpoint der skal implementeres og testes.
- `GET /auth/me` skal udvides til at returnere email og created_at.
- `chat/Dockerfile` skal opdateres med non-root bruger.
- `docker-compose.yml` skal have `security_opt` og resource limits på
  chat-service.
- Udestående punkter trackedes i `TODO.md`.
