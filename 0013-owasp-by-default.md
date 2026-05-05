# 0013 — OWASP-sikkerhed som del af definition of done

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps og fremtidige projekter

## Kontekst

En OWASP-gennemgang af ipfs-apps/chat afslørede flere sikkerhedsproblemer der
var blevet introduceret gradvist uden at blive opdaget: stored XSS i to
frontend-filer, manglende security headers, ingen input-validering på
backend-endpoints, og email enumeration. Problemerne var ikke resultatet af
dårlig intention, men af at sikkerhed ikke var en eksplicit del af
arbejdsprocessen ved tilføjelse af nye features.

## Beslutning

Alle nye features og endpoints skal gennemgå en OWASP-tjekliste som del af
implementation — ikke som en separat audit-fase bagefter.

Tjeklisten er ikke udtømmende, men dækker de kategorier der er mest relevante
for denne type projekt (REST API + vanilla JS frontend):

### Backend (Rust/Axum)

- [ ] **A03 Injection:** Alle DB-forespørgsler bruger parameteriserede queries (`.bind()`) — aldrig string-interpolation i SQL
- [ ] **A04 Input-validering:** Nye felter har eksplicitte min/max-grænser valideret i handler, ikke kun via DB-constraints
- [ ] **A07 Auth:** Nye endpoints der kræver login kalder `authenticate()` som første handling
- [ ] **A07 Enumeration:** Fejlbeskeder ved opslag på email/brugernavn afslører ikke om ressourcen eksisterer
- [ ] **A09 Logging:** Sikkerhedsrelevante hændelser (fejlet login, ugyldig token, afvist input) logges med `tracing::warn!`

### Frontend (vanilla JS)

- [ ] **A03 XSS:** Alle server-leverede værdier der indsættes i DOM bruger `textContent` eller `escapeHtml()` — aldrig rå `innerHTML` med ukontrollerede data
- [ ] **A02 Token-håndtering:** JWT gemmes og sendes korrekt — ingen eksponeringer i URL'er udover det dokumenterede WebSocket-undtagelse (ADR-0007)

### Infrastruktur (Caddy/Ansible)

- [ ] **A05 Security headers:** Nye sites i Caddyfile.j2 får samme header-blok som eksisterende sites (X-Content-Type-Options, X-Frame-Options, CSP, Referrer-Policy, Permissions-Policy, -Server)

## Begrundelse

Sikkerhed er billigst at indføre ved implementeringstidspunktet. En XSS-fejl
der opdages under code review koster minutter at rette; den samme fejl
opdaget i produktion koster tillid og tid til incident response.

OWASP Top 10 er en veletableret og vedligeholdt standard der præcist dækker
de angrebstyper der er relevante for webapplikationer med REST API og
JavaScript-frontend.

## Konsekvenser

- Tjeklisten er et redskab, ikke bureaukrati — et punkt kan bevidst afviges
  hvis der er en god grund, men afvigelsen skal være eksplicit.
- `OWASP-IMPROVEMENTS.md` i projektets rod dokumenterer kendte tilbageværende
  fund og er den løbende backlog for sikkerhedsforbedringer.
- Den fulde OWASP-gennemgang der lå til grund for denne beslutning er
  dokumenteret i `OWASP-IMPROVEMENTS.md` (commit c09b52d).
