# 0008 — IPFS + DNSLink til statisk frontend-hosting

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Den statiske frontend (vanilla JS) skal hostes et sted med en stabil, forudsigelig
URL, der kan bruges som `ALLOWED_ORIGIN` i backend-CORS-konfigurationen.
Alternativer:

- **Traditionel web-server / CDN:** Nginx, S3+CloudFront, Netlify, Vercel.
- **IPFS med DNSLink:** Frontend uploades til IPFS, CID bindes til et domæne
  via `_dnslink` TXT-record.

## Beslutning

Frontend deployes til **IPFS** og eksponeres via **DNSLink**
(`chat.apps.gihc.online → /ipfs/<CID>`). Domænet bruges som `ALLOWED_ORIGIN`.

DNS-recorden opdateres automatisk af Ansible-playbook via Simply.com API
efter hver deploy.

## Begrundelse

- **Stabil CORS-origin:** DNSLink giver et fast domænenavn (`chat.apps.gihc.online`)
  der ikke ændrer sig ved ny deploy — kun CID'en bag ændrer sig. Det betyder
  at `ALLOWED_ORIGIN` i backend aldrig skal opdateres.
- **Indholdsadressering:** IPFS CID er kryptografisk bundet til indholdet —
  ingen CDN-cache-invalidation nødvendig.
- **Ingen CDN-udgifter:** IPFS-noden kører på samme VPS som backend.
- **Automatisering:** Ansible uploader til IPFS og opdaterer DNS i samme
  playbook — zero manual steps.

## Konsekvenser

- **Simply.com API-afhængighed:** DNS-opdatering kræver Simply.com-credentials
  i vault. Skift af DNS-udbyder kræver ny `uri`-task i playbook.
- **IPFS-node som single point of failure:** Kun én IPFS-node — ingen
  pin-service (Pinata, Web3.Storage) bruges. Node-nedetid gør frontend
  utilgængeligt.
- **Ingen wildcard-CORS:** `ALLOWED_ORIGIN=*` bruges kun lokalt. I produktion
  er origin altid den specifikke DNSLink-URL.
- Frontend må ikke bruge server-side rendering eller dynamisk routing — ren
  statisk HTML/JS/CSS.
