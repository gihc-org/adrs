# 0019 — Delt Caddy via platform-lag og conf.d-import

**Status:** Accepted  
**Dato:** 2026-05-09  
**Projekt:** infra, ipfs-apps, capture

## Kontekst

VPS'en kører flere selvstændige projekter (`ipfs-apps/chat`, `capture/notes`, ...)
der alle skal eksponeres på port 80/443 over TLS. Kun én proces kan binde til
disse porte ad gangen.

Den første løsning var at ipfs-apps ejede Caddy og den fulde `Caddyfile`. Da
`capture` skulle tilføjes opstod et isolationsproblem: hvert projekt ville
potentielt overskrive hinandens Caddyfile ved deploy, og intet projekt kunne
deploye selvstændigt uden risiko for at bryde det andet.

Tre alternativer blev overvejet:

1. **Delt Caddy via platform-lag** (valgt)
2. **Caddy admin-API** — dynamisk konfiguration via HTTP POST til port 2019;
   konfiguration mistes ved Caddy-genstart og kræver rekonstruktionslogik.
3. **Separat VPS per projekt** — fuldstændig isolation, men uforholdsmæssig
   driftskompleksitet og omkostning for personlige projekter.

## Beslutning

Et selvstændigt `infra`-projekt kører én Caddy-instans med følgende hoved-konfiguration:

```caddy
{
    email kristian.n.jensen@gmail.com
}

import /etc/caddy/conf.d/*.caddy
```

Hvert projekt ejer præcis én fil i `conf.d/`:

```
/opt/platform/caddy/conf.d/
  chat.caddy    ← skrives af ipfs-apps ansible
  notes.caddy   ← skrives af capture ansible
```

Projekterne kommunikerer med Caddy via det delte Docker-netværk `platform_net`.
Alle app-containers joiner `platform_net` som et eksternt netværk.

Caddy genindlæses via `caddy reload` (ingen nedetid) efter hvert deploy.

## Begrundelse

- **Fuld isolation:** Hvert projekt kan kun redigere sin egen `.caddy`-fil.
  En fejl i et projekts deploy kan maksimalt gøre dets egne domæner utilgængelige
  — aldrig et andet projekts.
- **Statisk konfiguration:** Filer i `conf.d/` overlever Caddy-genstarter modsat
  admin-API-tilgangen.
- **Minimal kobling:** Projekterne er enige om ét interface: service-navnet
  i `platform_net` (f.eks. `notes:3000`). Ingen anden koordination er nødvendig.
- **Caddy reload er atomisk:** Caddy validerer den nye konfiguration før den
  aktiveres — en ugyldig `.caddy`-fil fra ét projekt afvises uden at påvirke
  de øvrige.

## Konsekvenser

- `infra` skal opsættes én gang på VPS'en inden øvrige projekter kan deploye
  (se `infra/ansible/infra.yml`).
- ipfs-apps mistede sin egen Caddy-service; `caddy_data`- og
  `caddy_config`-volumes flyttes til `infra`.
- Nye projekter på samme VPS følger samme mønster: lav en `<app>.caddy.j2`
  og skriv til `conf.d/` — se `infra/README.md`.
- TLS-certifikater (Let's Encrypt) er nu platform-ansvar, ikke projekt-ansvar.
