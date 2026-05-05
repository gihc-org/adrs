# 0009 — Caddy som reverse proxy med automatisk TLS

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Backend-service (Axum på port 8001) skal eksponeres på port 443 med TLS.
Alternativer:

- **Nginx:** Velkendt, bredt understøttet — men certifikater kræver manuel
  Let's Encrypt-integration (certbot, cron-job).
- **Caddy:** Automatisk TLS via Let's Encrypt og ZeroSSL ud af boksen.
- **Traefik:** Docker-native discovery — men overkill for single-service setup.

## Beslutning

Vi bruger **Caddy 2** som reverse proxy via `caddy:2-alpine` Docker-image.

## Begrundelse

- **Automatisk TLS:** Caddy håndterer certifikat-udstedelse og -fornyelse uden
  manuel konfiguration eller cron-jobs.
- **Simpel konfiguration:** Caddyfile til dette setup er < 20 linjer.
- **HTTP/2 og HTTP/3 out of the box:** `443:443/udp` i compose for QUIC.
- **WebSocket-proxy:** Caddy proxy-er WebSocket-upgrade-requests transparent.

## Konsekvenser

- **Caddy understøtter ikke `{env.VAR}` i site-adresser.** Domænenavnet kan
  ikke injiceres via environment-variabel i Caddyfile-headeren (f.eks.
  `{$DOMAIN} { ... }` virker ikke for site-adressen). Løsning: Ansible
  renderer Caddyfile via Jinja2-template (`templates/Caddyfile.j2`) inden
  deploy — se ADR-0010.
- **Caddy gemmer certifikater** i `caddy_data` Docker-volume — backup af
  denne volume er anbefalet, men certifikater kan genudstedes gratis.
- **HTTP-only lokalt:** `docker-compose.override.yml` konfigurerer Caddy til
  HTTP (ingen TLS) i lokal dev — ingen `mkcert` eller selvsignerede certs
  nødvendigt.
