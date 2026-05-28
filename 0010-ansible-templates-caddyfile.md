# 0010 — Template-rendering af Caddyfile frem for env vars

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

Caddyfile skal indeholde domænenavnet i site-adressen. Den naturlige løsning
ville være Caddy's `{$DOMAIN}`-syntax til at læse en environment-variabel.

```
# Ønsket, men virker ikke for site-adressen:
{$DOMAIN} {
    reverse_proxy myapp:8080
}
```

Caddy understøtter `{$VAR}` i direktiver *inde i* en blok, men **ikke** i
selve site-adressen (block header). Site-adressen evalueres ved parse-tid,
ikke runtime.

## Beslutning

Caddyfile genereres via **template-rendering** inden deploy. Med Ansible
bruges en Jinja2-template (`templates/Caddyfile.j2`):

```jinja
{{ domain }} {
    reverse_proxy {{ service_name }}:{{ service_port }}

    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
        -Server
    }
}
```

Variablerne hentes fra `group_vars/` og indsættes ved deploy-tid.
Alternativt kan `envsubst` bruges som simpel template-løsning uden Ansible.

## Begrundelse

- **Eneste fungerende løsning:** Caddy's parse-model tillader ikke
  environment-variable i site-adresser — dette er ikke en begrænsning der
  forventes at ændre sig.
- **Vault-integration:** Template-rendering integrerer naturligt med secrets
  (Ansible Vault, envsubst fra CI secret store).
- **Konfigurations-validering:** Den renderede Caddyfile valideres med
  `caddy validate --config` inden deploy — syntaksfejl stoppes i CI.

**Fravalgt alternativ — Caddy JSON-config:**
Caddy's native JSON API (`/config/`-endpoint) understøtter dynamisk
konfiguration uden filgenerering. Fravalgt fordi JSON-konfigurationen er
significanttterbose for simple setups og mistes ved Caddy-genstart uden
rekonstruktionslogik (se ADR-0019 for conf.d-mønsteret der løser dette).

## Konsekvenser

- Caddyfile redigeres i template-filen — ikke direkte på serveren.
- Domæne og service-navn er deploy-tids-variable, ikke runtime-variable.
- Domæneændring kræver ny deploy.
- Lokal dev bruger en simpel hårdkodet Caddyfile i
  `docker-compose.override.yml` (HTTP-only, ingen template-rendering).
- `caddy validate --config` tilføjes som obligatorisk CI-check inden deploy.
