# 0010 — Ansible-templates til Caddyfile frem for env vars

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Caddyfile skal indeholde domænenavnet i site-adressen (f.eks. `api.gihc.online
{ ... }`). Den naturlige løsning ville være at læse domænet fra en
environment-variabel via Caddy's `{env.DOMAIN}`-syntax.

```
# Ønsket, men virker ikke for site-adressen:
{$DOMAIN} {
    reverse_proxy chat:8001
}
```

Caddy understøtter `{env.VAR}` i direktiver inde i en blok, men **ikke** i
selve site-adressen (block header). Site-adressen evalueres ved parse-tid
(ikke runtime), og environment-variable er ikke tilgængelige på det tidspunkt.

## Beslutning

Caddyfile genereres af **Ansible via Jinja2-template** (`templates/Caddyfile.j2`):

```jinja
{{ domain }} {
    reverse_proxy chat:8001
    ...
}
```

Domænet hentes fra `group_vars/all/vars.yml` og indsættes ved deploy.

## Begrundelse

- **Eneste fungerende løsning:** Caddy's parse-model tillader ikke
  environment-variable i site-adresser.
- **Konsistens med resten af provisioning:** Ansible renderer allerede `.env`
  via `templates/env.j2` — samme mønster for Caddyfile er naturligt.
- **Vault-integration:** Ansible-vault håndterer secrets. Caddyfile behøver
  ingen secrets (kun domænenavn), men mønsteret er konsistent.

## Konsekvenser

- **Caddyfile er ikke direkte redigerbar på serveren** — ændringer skal ske i
  `templates/Caddyfile.j2` og deployes via playbook.
- **Domænet er en deploy-tidsvariabel**, ikke en runtime-variabel — ændring
  af domæne kræver ny deploy.
- Lokal dev bruger `docker-compose.override.yml` med en hårdkodet simpel
  Caddyfile (HTTP-only, ingen template-rendering nødvendig).
