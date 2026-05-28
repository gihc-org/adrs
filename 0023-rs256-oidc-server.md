# 0023 — RS256 frem for HS256 til OIDC Authorization Server

**Status:** Accepted  
**Dato:** 2026-05-11  
**Projekt:** sso-gihc

## Kontekst

security-guidelinen (`guidelines/security.md`) anbefaler HS256 med et stærkt
random secret til JWT. Dette valg er taget i konteksten af interne applikationer
hvor én service både udsteder og validerer tokens.

`sso-gihc` er en OIDC Authorization Server der udsteder tokens til
**resource servers** (chat, notes, fremtidige apps) som er separate services.
Her opstår et problem med HS256:

- HS256 er **symmetrisk**: den samme hemmelighed bruges til at signere *og*
  verificere. Alle resource servers skal kende hemmeligheden.
- At dele et signing-secret med alle apps bryder separation of concerns: en
  kompromitteret app kan udstede gyldige tokens.
- OIDC-protokollen er bygget på asymmetrisk kryptografi og publicerer public
  keys via `/.well-known/jwks.json` — resource servers henter public key og
  verificerer selvstændigt.

Alternativer overvejet:

| Algoritme | Nøgletype | Resource server verificerer | Problem |
|---|---|---|---|
| **HS256** | Shared secret | Ja — kender hemmelighed | Alle apps skal kende signing-secret |
| **RS256** | RSA 2048-bit | Ja — via JWKS (public key) | Privat nøgle skal beskyttes på SSO-service |
| **ES256** | ECDSA P-256 | Ja — via JWKS (public key) | `jsonwebtoken` 9 har begrænset ES256-support |

## Beslutning

Tokens signeres med **RS256** (RSA 2048-bit, SHA-256).

- Privat nøgle opbevares kun på SSO-servicen (env var `JWT_PRIVATE_KEY`).
- Public key publiceres på `/.well-known/jwks.json`.
- Resource servers verificerer tokens med public key — ingen delt hemmelighed.

```
SSO-service          Resource server (notes, chat, ...)
──────────           ─────────────────────────────────
Privat nøgle   →     Signerer access token (RS256)
Public key (JWKS) →  Verificerer token selvstændigt
```

Nøgleparet genereres én gang med:
```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

## Begrundelse

- **Separation of concerns:** Kun SSO kender den private nøgle. En
  kompromitteret resource server kan ikke udstede tokens.
- **OIDC-standard:** OIDC Core-specifikationen (OpenID Foundation, 2014)
  forudsætter asymmetrisk signering og JWKS-discovery. HS256 er teknisk
  muligt i OIDC, men kræver out-of-band key distribution og bruges ikke i
  praksis.
- **Cross-domain-krav:** Fremtidige apps på andre domæner kan hente JWKS og
  verificere tokens uden at kalde SSO-servicen — reducerer latency og
  afhængighed.
- **Key rotation:** Public key har `kid`-felt. Ved rotation publiceres ny
  nøgle i JWKS; resource servers henter automatisk ny nøgle ved første
  token med ukendt `kid`.

## Konsekvenser

- **security.md skal opdateres** med undtagelse: RS256 bruges til OIDC
  Authorization Servers; HS256 forbliver anbefalingen til interne
  service-til-service JWT.
- Privat nøgle er kritisk hemmelighed — opbevares som Docker secret eller
  env var på VPS. **Må aldrig committes til git.**
- `jsonwebtoken` 9-craten håndterer RS256 med `EncodingKey::from_rsa_pem` /
  `DecodingKey::from_rsa_pem`.
- Resource servers der vil verificere tokens stateless tilføjer:
  ```rust
  let key = DecodingKey::from_rsa_pem(public_key_pem)?;
  let mut v = Validation::new(Algorithm::RS256);
  v.set_issuer(&["https://sso.gihc.online"]);
  ```
