# 0011 — Argon2id til password-hashing

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Bruger-passwords skal hashes sikkert i databasen. Klassiske alternativer:

- **bcrypt:** Bred understøttelse, men begrænset til 72-byte input.
- **PBKDF2:** NIST-standard, men ikke memory-hard.
- **Argon2id:** Vinder af Password Hashing Competition (2015). Memory-hard,
  thread-parallel, modstandsdygtig over for GPU- og ASIC-angreb.

## Beslutning

Vi bruger **Argon2id** via `argon2`-crate med standardparametre og tilfældig
salt genereret af `OsRng`.

```rust
let salt = SaltString::generate(&mut OsRng);
Argon2::default().hash_password(password.as_bytes(), &salt)
```

Output er en selvbærende PHC-streng der inkluderer algoritme-parametre og salt.

## Begrundelse

- **Memory-hardness:** Argon2id kræver betydelig RAM pr. hash-forsøg — det
  gør brute-force med GPU-farme uøkonomisk.
- **OWASP-anbefaling:** Argon2id er OWASP's primære anbefaling til
  password-hashing pr. 2024.
- **PHC-format:** Hash er selvbeskrivende — algoritme og parametre er embedded.
  Det gør fremtidige parameter-opgraderinger mulige uden databae-migration
  (re-hash ved næste login).
- **Bevidst langsom:** Login-latens på ~100ms er en feature der begrænser
  online angreb.

## Konsekvenser

- `verify_password` returnerer `false` (ikke panic) ved malformed hash, så
  en korrupt databaserække degraderer gracefully til "forkert adgangskode".
- Standardparametre (`Argon2::default()`) bruges — ved øget sikkerhedskrav
  kan `Params` konfigureres eksplicit (memory, iterations, parallelism).
- Registration og login er langsommere end med hurtige hashing-algoritmer —
  dette er forventet og ønsket.
