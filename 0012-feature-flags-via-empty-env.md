# 0012 — Tom env-variabel som feature flag til lokal udvikling

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

Projektet har to valgfri tredjeparts-integrationer:

- **Cloudflare Turnstile** (CAPTCHA ved registration)
- **Resend** (email-verifikation efter registration)

Begge kræver credentials der ikke er tilgængelige i lokal dev-miljø.
I produktion er de obligatoriske. Vi har brug for en måde at springe dem over
lokalt uden at ændre kode.

## Beslutning

En **tom string** i den respektive env-variabel bruges som feature flag:

- `TURNSTILE_SECRET=""` → CAPTCHA-validering springes over, alle tokens accepteres
- `RESEND_API_KEY=""` → Email sendes ikke, bruger auto-verificeres, verify-URL logges

```rust
// captcha.rs
pub async fn verify(client: &reqwest::Client, secret: &str, token: &str) -> bool {
    if secret.is_empty() { return true; }
    ...
}

// routes/auth.rs
let auto_verify = state.config.resend_api_key.is_empty();
```

`.env.example` indeholder tomme værdier for disse variable som default.

## Begrundelse

- **Nul kode-ændringer** mellem lokal dev og produktion — samme binær, ander
  konfiguration.
- **Eksplicit toggle:** Tom string er semantisk tydelig som "ikke konfigureret"
  — ingen separat `ENABLE_CAPTCHA=false`-variabel.
- **Sikker standard:** Udeladelse af `TURNSTILE_SECRET` fra `.env` er
  tilstrækkeligt til at slå CAPTCHA fra lokalt.
- **Konsistent mønster:** Begge integrationer følger samme konvention — let at
  forstå og let at tilføje nye valgfri integrationer.

## Konsekvenser

- En tom `TURNSTILE_SECRET` i **produktion** ville slå CAPTCHA fra. Ansible-vault
  og `.env.example`-dokumentation skal gøre det klart at variable SKAL sættes
  i produktion.
- Lokale auto-verificerede konti vil fejle i produktion hvis email ikke er sat
  korrekt op — det opdages ved første deploy-test.
- Nye valgfri integrationer bør følge samme mønster: tom string = skip,
  non-empty = aktiv.
