# 0013 — OWASP-sikkerhed som del af definition of done

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

Sikkerhedsproblemer introduceres typisk gradvist uden at blive opdaget:
stored XSS, manglende security headers, ingen input-validering og
information exposure opstår ikke ved bevidst negligence, men fordi
sikkerhed ikke er en eksplicit del af arbejdsprocessen.

En OWASP Top 10-gennemgang er billigst som del af implementeringen —
ikke som en separat audit-fase bagefter.

## Beslutning

Alle nye features og endpoints gennemgår en OWASP-tjekliste som del af
implementering. Tjeklisten nedenfor dækker de kategorier der er mest
relevante for REST API + web-frontend projekter.

### Backend

- [ ] **A03 Injection:** Alle DB-forespørgsler bruger parameteriserede queries
  — aldrig string-interpolation i SQL eller shell-kommandoer
- [ ] **A04 Input-validering:** Nye felter har eksplicitte min/max-grænser
  valideret i handler, ikke kun via DB-constraints
- [ ] **A07 Auth:** Nye beskyttede endpoints validerer token/session som
  første handling — ingen logik køres inden auth-tjek
- [ ] **A07 Enumeration:** Fejlbeskeder ved opslag på e-mail/brugernavn
  afslører ikke om ressourcen eksisterer ("ugyldigt login", ikke
  "e-mail ikke fundet")
- [ ] **A09 Logging:** Sikkerhedsrelevante hændelser (fejlet login, ugyldig
  token, afvist input) logges — tokens og passwords logges aldrig
  (se ADR-0026)

### Frontend

- [ ] **A03 XSS:** Alle server-leverede værdier der indsættes i DOM bruger
  `textContent` eller en escape-funktion — aldrig rå `innerHTML` med
  ukontrollerede data
- [ ] **A02 Token-håndtering:** Auth-tokens gemmes og sendes korrekt —
  ingen eksponeringer i URL'er ud over dokumenterede undtagelser (se ADR-0007)

### Infrastruktur

- [ ] **A05 Security headers:** Nye sites i reverse proxy-konfigurationen
  får security headers:
  `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`,
  `Permissions-Policy`, `-Server` (fjern server-identificering)

### OWASP ZAP i CI

OWASP ZAP baseline scan køres mod staging efter hvert deploy (se `ci-cd.md`).
Fund fra ZAP der ikke er bevidste afvigelser behandles som CI-fejl.

## Begrundelse

Sikkerhed er billigst at indføre ved implementeringstidspunktet. En XSS-fejl
der opdages under code review koster minutter; den samme fejl opdaget i
produktion koster tillid og incident response-tid.

OWASP Top 10 er en veletableret og vedligeholdt standard der præcist dækker
de angrebstyper der er relevante for webapplikationer.

## Konsekvenser

- Tjeklisten er et redskab, ikke bureaukrati — et punkt kan bevidst afviges
  med en eksplicit begrundelse.
- Stack-specifikke tjeklister (f.eks. Rust/Axum + vanilla JS) lægges som
  separate filer i `checklists/` i projektet og linkes fra `AGENTS.md`.
- `OWASP-IMPROVEMENTS.md` i projektets rod bruges som backlog for kendte
  fund der ikke er rettet endnu.
