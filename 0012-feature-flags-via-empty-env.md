# 0012 — Tom env-variabel som feature flag til lokal udvikling

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

Applikationer har ofte valgfrie tredjeparts-integrationer (CAPTCHA, e-mail,
betalingsgateway, SMS) der kræver credentials som ikke er tilgængelige i
lokal dev. I produktion er de obligatoriske.

Der er behov for en konvention der slår integrationen fra lokalt uden at
ændre kode eller indføre separate `ENABLE_X=false`-variabler.

## Beslutning

En **tom string** i den respektive env-variabel bruges som feature flag:

- Tom string → integrationen springes over (lokal/test-adfærd)
- Non-empty string → integrationen aktiveres (produktionsadfærd)

```rust
// Eksempel: valgfri CAPTCHA-integration
pub async fn verify_captcha(secret: &str, token: &str) -> bool {
    if secret.is_empty() { return true; }  // spring over hvis ikke konfigureret
    // ... kald til ekstern service
}
```

`.env.example` indeholder tomme værdier for valgfrie integrationer som standard.

```bash
# .env.example
CAPTCHA_SECRET=        # Tom = CAPTCHA deaktiveret (kun lokalt)
EMAIL_API_KEY=         # Tom = emails logges i stedet for at sendes
```

## Begrundelse

- **Nul kode-ændringer** mellem lokal dev og produktion — samme binær,
  forskellig konfiguration.
- **Eksplicit toggle:** Tom string er semantisk tydelig som "ikke konfigureret".
- **Konsistent mønster:** Alle valgfrie integrationer følger samme konvention.

## Testkrav

Test at applikationen opfører sig korrekt i begge tilstande:

```rust
#[test]
fn captcha_skipped_when_secret_is_empty() {
    assert!(verify_captcha("", "any-token"));
}

#[test]
fn captcha_checked_when_secret_is_set() {
    // Mock ekstern service eller brug en test-nøgle
}
```

## Konsekvenser

- **Sikkerhedskritiske variable må aldrig bruge dette mønster i produktion.**
  En tom `DATABASE_URL` eller `JWT_SECRET` skal give en hård fejl ved
  opstart — ikke stille fallback-adfærd. Brug opstartsvalidering
  (se ADR-0026) til at håndhæve dette:
  ```rust
  // Aldrig dette for sikkerhedskritiske vars:
  if jwt_secret.is_empty() { return default_behavior(); }
  // I stedet:
  assert!(!jwt_secret.is_empty(), "JWT_SECRET må ikke være tom i produktion");
  ```
- Mønsteret egner sig til: CAPTCHA, e-mail, SMS, analytics, ekstern logging.
- Mønsteret egner sig **ikke** til: database-URL, hemmeligheder, krypteringsnøgler.
- Nye valgfrie integrationer bør følge samme konvention og dokumenteres i
  `.env.example`.
