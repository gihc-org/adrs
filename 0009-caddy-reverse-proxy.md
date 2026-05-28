# 0009 — Caddy som reverse proxy med automatisk TLS

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

En backend-service skal eksponeres på port 443 med TLS. Alternativer:

- **Nginx:** Velkendt, bredt understøttet — men certifikater kræver manuel
  Let's Encrypt-integration (certbot, cron-job).
- **Caddy:** Automatisk TLS via Let's Encrypt og ZeroSSL ud af boksen.
- **Traefik:** Docker-native discovery — men overkill for single-service setup.

## Beslutning

Brug **Caddy 2** som reverse proxy via `caddy:2-alpine` Docker-image.

## Begrundelse

- **Automatisk TLS:** Caddy håndterer certifikat-udstedelse og -fornyelse
  uden manuel konfiguration eller cron-jobs.
- **Simpel konfiguration:** En Caddyfile til et standard setup er < 20 linjer.
- **HTTP/2 og HTTP/3 out of the box:** Eksponér `443/udp` i compose for QUIC.
- **WebSocket-proxy:** Caddy proxy-er WebSocket-upgrade-requests transparent.

## Konsekvenser

- **Caddy understøtter ikke `{$VAR}` i site-adresser.** Domænenavnet kan
  ikke injiceres via environment-variabel i Caddyfile-headeren — det
  evalueres ved parse-tid. Løsning: brug en template-mekanisme
  (Ansible/Jinja2, envsubst) til at indsætte domænet ved deploy —
  se ADR-0010.
- **Caddy gemmer certifikater** i et Docker-volume (`caddy_data`) —
  backup anbefales, men certifikater kan genudstedes gratis.
- **HTTP-only lokalt:** `docker-compose.override.yml` konfigurerer Caddy til
  HTTP i lokal dev — ingen selvsignerede certs nødvendigt.
- **Konfigurations-validering i CI:** Kør `caddy validate --config Caddyfile`
  inden deploy for at fange syntaksfejl (se `ci-cd.md`).
