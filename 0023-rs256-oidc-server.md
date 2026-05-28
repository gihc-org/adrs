# 0023 — RS256 frem for HS256 til OIDC Authorization Server

**Status:** Accepted  
**Dato:** 2026-05-11

## Kontekst

HS256 er anbefalingen til JWT i interne applikationer hvor én service både
udsteder og validerer tokens. Denne ADR dokumenterer undtagelsen: en dedikeret
OIDC Authorization Server der udsteder tokens til **separate resource servers**.

Problemet med HS256 i multi-service OIDC:
- HS256 er **symmetrisk** — den samme hemmelighed bruges til at signere *og*
  verificere. Alle resource servers skal kende hemmeligheden.
- Deling af et signing-secret bryder separation of concerns: en kompromitteret
  resource server kan udstede gyldige tokens.
- OIDC-protokollen er bygget på asymmetrisk kryptografi og publicerer public
  keys via `/.well-known/jwks.json`.

Alternativer:

| Algoritme | Nøgletype | Resource server verificerer | Bemærkning |
|---|---|---|---|
| **HS256** | Shared secret | Ja — kender hemmelighed | Alle apps skal kende signing-secret |
| **RS256** | RSA 2048-bit | Ja — via JWKS | Valgt |
| **ES256** | ECDSA P-256 | Ja — via JWKS | Foretrækkes på sigt — mindre nøgle, hurtigere |

## Beslutning

Tokens signeres med **RS256** (RSA 2048-bit, SHA-256).

```
OIDC-service                 Resource server
────────────                 ───────────────
Privat nøgle  →  Signerer access token (RS256)
Public key via JWKS  →  Verificerer token selvstændigt
```

- Privat nøgle opbevares kun på OIDC-servicen (`JWT_PRIVATE_KEY` env var).
- Public key publiceres på `/.well-known/jwks.json`.
- Resource servers verificerer tokens med public key — ingen delt hemmelighed.

Nøgleparet genereres:
```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

**Fremtidig præference:** ES256 (ECDSA P-256) foretrækkes når biblioteks-
understøttelsen er komplet — nøglen er kortere og operationerne hurtigere
end RSA-2048. Migrér ved næste større revision af auth-stakken.

## Begrundelse

- **Separation of concerns:** Kun OIDC-servicen kender den private nøgle.
  En kompromitteret resource server kan ikke udstede tokens.
- **OIDC-standard:** OIDC Core-specifikationen forudsætter asymmetrisk
  signering og JWKS-discovery. HS256 i OIDC kræver out-of-band key
  distribution og bruges ikke i praksis.
- **Key rotation:** Public key har `kid`-felt. Ved rotation publiceres ny
  nøgle i JWKS; resource servers henter automatisk ny nøgle ved første
  token med ukendt `kid`.

## Testkrav

- [ ] Integration test: resource server verificerer et token udstedt af
  OIDC-servicen via JWKS-endpointet
- [ ] Integration test: token med ukendt `kid` afvises korrekt
- [ ] Integration test: udløbet token afvises
- [ ] Integration test: `iss`-claim matcher den konfigurerede issuer

## Konsekvenser

- Privat nøgle er en kritisk hemmelighed — opbevares som Docker secret
  eller env var. **Må aldrig committes til git.**
- `security.md` opdateres med undtagelsen: RS256 til OIDC Authorization
  Servers; HS256 forbliver anbefalingen til interne service-til-service JWT.
- Resource servers der verificerer tokens stateless tilføjer JWKS-hentning
  og caching — hent ikke JWKS ved hvert request.

### Rust-implementation (appendiks)

```rust
// Udstedelse (OIDC-service)
let key = EncodingKey::from_rsa_pem(private_key_pem)?;
let token = encode(&Header::new(Algorithm::RS256), &claims, &key)?;

// Verifikation (resource server)
let key = DecodingKey::from_rsa_pem(public_key_pem)?;
let mut validation = Validation::new(Algorithm::RS256);
validation.set_issuer(&["https://your-oidc-server.example.com"]);
let data = decode::<Claims>(&token, &key, &validation)?;
```
