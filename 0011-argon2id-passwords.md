# 0011 — Argon2id til password-hashing

**Status:** Accepted  
**Dato:** 2026-05-05

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

### Standardparametre (`Argon2::default()`)

| Parameter | Værdi | Betydning |
|-----------|-------|-----------|
| `m_cost` | 19456 KiB (~19 MB) | Hukommelsesforbrug pr. hash |
| `t_cost` | 2 | Antal iterationer |
| `p_cost` | 1 | Paralleliseringsgrad |
| Algoritme | Argon2id | Hybrid: modstandsdygtig mod side-channel og GPU |

Disse parametre er OWASP's minimumskrav pr. 2024. Justér `m_cost` og `t_cost`
opad ved øget sikkerhedskrav eller hurtigere hardware. PHC-formatet sikrer at
gamle hashes forbliver gyldige efter en parameter-opgradering — re-hash sker
ved næste login.

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

## Testkrav

Password-hashing er sikkerhedskritisk. Alle tre stier skal have unit tests:

```rust
#[test]
fn hash_and_verify_roundtrip() {
    let hash = hash_password("korrekt-password");
    assert!(verify_password("korrekt-password", &hash));
}

#[test]
fn verify_returns_false_for_wrong_password() {
    let hash = hash_password("korrekt-password");
    assert!(!verify_password("forkert-password", &hash));
}

#[test]
fn verify_returns_false_for_malformed_hash() {
    // Korrupt databaserække må ikke give panic
    assert!(!verify_password("password", "ikke-et-gyldigt-phc-hash"));
}

#[test]
fn re_hash_works_after_parameter_upgrade() {
    // Verificér at et hash genereret med gamle parametre stadig kan verificeres
    // og at re-hash producerer et hash med nye parametre
    let old_hash = hash_with_params("password", OLD_PARAMS);
    assert!(verify_password("password", &old_hash));
    let new_hash = hash_password("password"); // nye parametre
    assert!(verify_password("password", &new_hash));
    assert_ne!(old_hash, new_hash);
}
```

Test må ikke være langsomme nok til at blokere CI. Brug reducerede parametre
(`m_cost: 256, t_cost: 1`) i testhjælpefunktioner der kun tester logik, ikke
sikkerhedsstyrke.

## Konsekvenser

- `verify_password` returnerer `false` (ikke panic) ved malformed hash —
  dette er et krav der skal dækkes af en unit test.
- Standardparametre (`Argon2::default()`) er dokumenteret ovenfor. Ved øget
  sikkerhedskrav konfigureres `Params` eksplicit.
- Registration og login er langsommere end med hurtige hashing-algoritmer —
  dette er forventet og ønsket.
- Se ADR-0025 for håndtering af smoke-test-scenariet hvor hash skal
  forudberegnes uden live database.
