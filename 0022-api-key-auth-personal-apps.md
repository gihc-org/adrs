# 0022 — API-nøgle frem for JWT til apps med lille betroet brugergruppe

**Status:** Accepted  
**Dato:** 2026-05-09

## Kontekst

Personlige apps og interne værktøjer for en lille betroet gruppe (ejer +
få personer med fuld tillid) behøver ikke fuld brugeradministration med
registrering, e-mail-bekræftelse og per-bruger dataadskillelse.

Alternativerne:
- **JWT med password-login:** Argon2id, refresh-tokens, e-mail-bekræftelse —
  dimensioneret til mange brugere, unødvendig kompleksitet for én-bruger-apps.
- **HTTP Basic Auth via reverse proxy:** Beskytter hele sitet men giver ingen
  programmatisk kontrol over fejlrespons og understøttes dårligt af `fetch`.
- **Ingen autentificering + IP-whitelist:** Skrøbeligt og ufleksibelt.

## Beslutning

Autentificering sker via en statisk **API-nøgle** i HTTP-headeren `x-api-key`.

Nøglen er en kryptografisk tilfældig streng (minimum 32 bytes / 64 hex-tegn)
sat i miljøvariablen `API_KEY`. Alle endpoints er beskyttet af middleware
der returnerer 401 hvis headeren mangler eller er forkert.

```bash
# Generér en nøgle
openssl rand -hex 32
```

Klienten gemmer nøglen i `localStorage` på egne enheder.

## Begrundelse

- **Ingen brugeradministration:** Ét token erstatter alt login-flow.
  Gruppen er lille og stabil — ingen selregistrering nødvendig.
- **Delte data er intentionelt:** Alle brugere i gruppen ser og redigerer
  de samme data. Ingen krav om per-bruger dataadskillelse.
- **Funktionelt ækvivalent:** For en lille betroet gruppe er en stærk
  tilfældig streng i `x-api-key` ækvivalent med JWT — begge kræver at
  hemmeligheden beskyttes.
- **HTTPS tvungen via reverse proxy:** Nøglen transmitteres aldrig i klartekst.

## Sikkerhedsgrænser (OWASP ASVS)

**Denne model er ikke egnet til:**
- Apps med ikke-betroede brugere
- Apps med krav om per-bruger adgangskontrol
- Offentligt tilgængelige apps

`localStorage` er sårbart over for XSS-angreb (OWASP ASVS L2 §3.4.3).
For applikationer med højere sikkerhedskrav bruges `HttpOnly`-cookies eller
et fuldt JWT-flow med refresh tokens.

**Kompromittering:** API-nøgle kompromitteret → opdatér `API_KEY`, kør deploy.
Ingen bruger-tabel der skal renses.

## Konsekvenser

- `API_KEY` genereres én gang og opbevares i Ansible vault (eller CI secret store).
  Rotation kræver deploy.
- Browserklienten gemmer nøglen i `localStorage` — acceptabelt for personlige
  apps på egne enheder. Nøglen bruges ikke i delte browsere.
- Alle brugere deler samme nøgle og ser alle data — modellen forudsætter
  gensidig tillid i gruppen.
- Modellen er **ikke** egnet hvis brugerne skal have separate datarum —
  da indføres JWT og fuld brugeradministration (se ADR-0011, ADR-0023).
