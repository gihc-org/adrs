# 0022 — API-nøgle frem for JWT til apps med lille betroet brugergruppe

**Status:** Accepted  
**Dato:** 2026-05-09  
**Projekt:** capture

## Kontekst

`capture` er et personligt værktøj for en lille betroet gruppe (ejer +
et par venner). Appen skal kunne tilgås fra flere enheder og må ikke være
offentligt
tilgængeligt.

ipfs-apps bruger JWT med Argon2id-hashet password (ADR-0011) og fuld
brugeradministration (registrering, bekræftelses-e-mail, sletning iht. GDPR).
Den samme model kunne genbruges, men den er dimensioneret til mange brugere.

## Beslutning

Autentificering sker via en statisk **API-nøgle** i HTTP-headeren `x-api-key`.

Nøglen er en tilfældig streng der sættes i miljøvariablen `API_KEY` og
opbevares i Ansible vault. Klienten gemmer nøglen i `localStorage` efter
første login.

Alle API-endpoints er beskyttet af et Axum-middleware der returnerer 401
hvis headeren mangler eller er forkert.

## Begrundelse

- **Ingen brugeradministration:** Ét hemmeligt token erstatter alt
  login-flow, password-hashing, token-rotation og sessionshåndtering.
  Gruppen er lille og stabil — ingen selvregi­strering er nødvendig.
- **Delte data er intentionelt:** Alle brugere i gruppen ser og redigerer
  de samme noder. Der er ingen krav om per-bruger dataadskillelse.
- **Samme sikkerhedsniveau i praksis:** For en lille betroet gruppe er en
  stærk tilfældig streng i `x-api-key` funktionelt ækvivalent med JWT —
  begge kræver at hemmeligheden beskyttes. JWT's fordel (short-lived tokens,
  revocation per bruger) er irrelevant når alle deler ét adgangsniveau.
- **HTTPS tvungen via Caddy:** Nøglen transmitteres aldrig i klartekst.
  TLS-terminering håndteres af platform-Caddy (ADR-0017).
- **Simpelt fejlscenarie:** Kompromitteret nøgle → opdater `API_KEY` i vault,
  kør deploy. Ingen bruger-tabel der skal renses.

**Fravalgte alternativer:**
- *JWT med password-login:* Unødvendig kompleksitet (Argon2id, refresh-tokens,
  e-mail-bekræftelse) for én bruger.
- *HTTP Basic Auth via Caddy:* Ville beskytte hele sitet inkl. statiske filer,
  men giver ingen programmatisk kontrol over fejlrespons fra API'et og
  understøttes dårligt af `fetch` i moderne browsere.
- *Ingen autentificering + IP-whitelist:* Skrøbeligt og ufleksibelt ved
  mobil-adgang.

## Konsekvenser

- `API_KEY` genereres én gang og opbevares i Ansible vault som
  `vault_notes_api_key`. Rotation kræver deploy.
- Browserklienten gemmer nøglen i `localStorage` — acceptabelt for en
  personlig app på egne enheder. Nøglen må ikke bruges i delte browsere.
- Smoke tests autentificerer med vault-nøglen direkte fra Ansible.
- Alle brugere deler samme nøgle og ser alle data — modellen forudsætter
  gensidig tillid i gruppen.
- Modellen er **ikke** egnet hvis brugerne skal have separate datarum
  — da skal JWT og brugeradministration indføres.
