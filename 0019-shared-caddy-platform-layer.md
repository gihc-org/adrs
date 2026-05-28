# 0019 — Delt Caddy via platform-lag og conf.d-import

**Status:** Accepted  
**Dato:** 2026-05-09

## Kontekst

En server kører flere selvstændige projekter der alle skal eksponeres på
port 80/443 over TLS. Kun én proces kan binde til disse porte ad gangen.

Den første løsning var at ét projekt ejede Caddy og den fulde Caddyfile.
Da et andet projekt skulle tilføjes opstod et isolationsproblem: projekterne
ville potentielt overskrive hinandens Caddyfile ved deploy.

Tre alternativer:

1. **Delt Caddy via platform-lag** (valgt)
2. **Caddy admin-API** — dynamisk konfiguration via HTTP POST; mistes ved
   Caddy-genstart og kræver rekonstruktionslogik.
3. **Separat server per projekt** — fuldstændig isolation, men
   uforholdsmæssig driftskompleksitet for personlige projekter.

## Beslutning

Et selvstændigt `infra`-projekt kører én Caddy-instans med:

```caddy
{
    email {$ACME_EMAIL}
}

import /etc/caddy/conf.d/*.caddy
```

Hvert projekt ejer præcis én fil i `conf.d/`:

```
/opt/platform/caddy/conf.d/
  myapp.caddy      ← skrives af myapp's deploy-script
  otherapp.caddy   ← skrives af otherapp's deploy-script
```

Projekterne kommunikerer med Caddy via et delt Docker-netværk (`platform_net`).
Alle app-containers joiner `platform_net` som eksternt netværk.

Caddy genindlæses via `caddy reload` (ingen nedetid) efter hvert deploy.
Caddy validerer den nye konfiguration automatisk inden aktivering.

## Begrundelse

- **Fuld isolation:** Hvert projekt kan kun redigere sin egen `.caddy`-fil.
  En fejl i ét projekts deploy kan maksimalt gøre dets egne domæner
  utilgængelige.
- **Statisk konfiguration:** Filer i `conf.d/` overlever Caddy-genstarter
  — i modsætning til admin-API-tilgangen.
- **Minimal kobling:** Projekterne er enige om ét interface: service-navn
  i `platform_net` (f.eks. `myapp:8080`). Ingen anden koordination nødvendig.

## Konsekvenser

- `infra`-projektet skal opsættes én gang på serveren inden øvrige projekter
  deployer (dokumentér i runbook).
- TLS-certifikater (Let's Encrypt) er platform-ansvar, ikke projekt-ansvar.
- Nye projekter på samme server følger mønsteret: opret en `<app>.caddy.j2`
  template og skriv til `conf.d/` ved deploy.
- `caddy validate` køres i CI inden deploy som del af konfigurations-
  validering (se `ci-cd.md`).
