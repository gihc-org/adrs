# 0016 — WebRTC skærmdelings-signalering via eksisterende WebSocket

**Status:** Accepted  
**Dato:** 2026-05-06  
**Projekt:** ipfs-apps/chat

## Kontekst

Brugere ønsker at dele deres skærm med hinanden. WebRTC er browser-standarden
for peer-to-peer mediaforbindelser og kræver en *signalseringskanal* for at
udveksle `RTCSessionDescription` (SDP offer/answer) og ICE-kandidater inden
den direkte peer-forbindelse kan oprettes. Selve mediestrømmen går direkte
mellem browserne — ikke igennem serveren.

Der er allerede en WebSocket-kanal per rum der broadcaster JSON-beskeder med
et `type`-felt. Backenden forstår ikke beskedindholdet — den videresender til
alle forbundne klienter i rummet.

### Fase 1 (denne ADR): 1:1 skærmdeling i DM-rum

DM-rum har præcis 2 deltagere, hvilket gør signaleringen enkel: én sender SDP
offer, den anden svarer med SDP answer, begge udveksler ICE-kandidater.

### Fase 2 (fremtidig ADR): Gruppe-huddles i offentlige rum

Mesh-topologi (alle forbundet med alle) fungerer op til ~4 deltagere.
Større grupper kræver en SFU (Selective Forwarding Unit), som er ekstern
infrastruktur og behandles i en separat ADR.

## Beslutning

### Signalering over eksisterende WebSocket-broadcast

Backenden udvides til at viderebringe en ny beskedtype — `signal` — uden at
gemme den i databasen. Klienter sender signalering som:

```json
{
  "type": "signal",
  "target": "<modtager-uuid>",
  "signal": {
    "type": "offer" | "answer" | "ice-candidate",
    "sdp": "...",
    "candidate": { ... }
  }
}
```

Backenden broadcaster beskeden til alle i rummet. Klienten ignorerer beskeder
hvor `from` ikke er den forventede peer. `from`-feltet tilføjes af backenden
(ikke klienten) ud fra den autentificerede brugers UUID — klienten kan ikke
forfalske afsender-identiteten.

Alternativet — dedikerede per-bruger kanaler — er unødvendigt komplekst for
DM-rum med 2 deltagere og udskydes til gruppe-fasen.

### STUN + TURN

```javascript
const ICE_SERVERS = [
  { urls: 'stun:stun.l.google.com:19302' },
  { urls: 'stun:stun.cloudflare.com:3478' },
];
// TURN tilføjes dynamisk fra config.js hvis TURN_SECRET er defineret
if (typeof TURN_URL !== 'undefined') {
  ICE_SERVERS.push({ urls: TURN_URL, username, credential }); // HMAC-SHA1, 24h TTL
}
```

STUN alene fejler for brugere bag symmetrisk NAT (typisk mobile hotspots og
visse firewalls). `coturn` kører som en service i `docker-compose.prod.yml`
på port 3478 med relay-ports 49152-49200.

**TURN-autentificering:** `--use-auth-secret` med et statisk HMAC-SHA1
shared secret (fra Ansible vault). Frontenden genererer tidsbegrænsede
credentials via Web Crypto API:

```javascript
const expires = Math.floor(Date.now() / 1000) + 86400;
const username = String(expires);
// credential = base64(HMAC-SHA1(TURN_SECRET, username))
```

Credentials er synlige i frontend-JS men udløber efter 24 timer.
`TURN_SECRET` injiceres i `frontend/config.js` af Ansible (template
`ansible/templates/config.js.j2`) umiddelbart før `ipfs add` i deploy-flowet.

### Beskedflow for 1:1

```
Alice                    Server                   Bob
  |                        |                        |
  |-- signal(offer) -----> |-- broadcast ---------->|
  |                        |                        |-- signal(answer) -->|
  |<-- broadcast ----------|<-- signal(answer) -----|
  |                        |                        |
  | [ICE-kandidater udveksles på samme måde]        |
  |                        |                        |
  | <======= direkte peer-to-peer mediaforbindelse ========> |
```

### Rejoin-håndtering

Hvis modtageren forlader DM-rummet og vender tilbage, er deres
`RTCPeerConnection` tabt. Shareren registrerer modtagerens `join`-besked
og sender automatisk et nyt offer via `reinitiateOffer()`.

For at undgå at rive en fungerende forbindelse ned (race condition når
modtageren åbner rummet mens shareren netop har startet): `reinitiateOffer`
springer over hvis et offer er sendt indenfor de seneste 3 sekunder.
`connectionState === 'connected'` er ikke en pålidelig guard — WebRTC
holder forbindelsen "connected" i ~30 sekunder efter peeren er gået.

### Scope for fase 1

- Skærmdeling tilbydes kun i DM-rum (præcis 2 deltagere)
- Ingen video- eller lydopkald — kun `getDisplayMedia()` (skærm/vindue/fane)
- Forbindelsen er envejs: én deler, én ser
- Stop-knap afslutter `MediaStream`-tracks og lukker `RTCPeerConnection`
- Modtager genoptager automatisk visning ved rejoin (uden at shareren genstarter)

### Backend-ændring (`src/routes/chat.rs`)

`recv_task` i `handle_socket` håndterer i dag kun beskeder med `content`.
Den udvides til at genkende `type: "signal"` og broadcaste dem videre med
`from` sat til den autentificerede brugers UUID:

```rust
if data["type"] == "signal" {
    if let Some(target) = data["target"].as_str() {
        if Uuid::parse_str(target).is_ok() {
            let mut fwd = data.clone();
            fwd["from"] = serde_json::Value::String(user.id.to_string());
            let _ = tx2.send(fwd.to_string());
        }
    }
    continue; // gem ikke i databasen
}
```

### Frontend-ændring (`frontend/chat.html`)

- "Del skærm"-knap vises kun i DM-rum
- `getDisplayMedia({ video: true })` starter optagelsen
- `RTCPeerConnection` med STUN-servere ovenfor
- Indgående `signal`-beskeder håndteres: offer → svar med answer, ICE-kandidater tilføjes løbende
- Modtagerens side viser stream i et `<video>`-element

## Begrundelse

**Signalering over eksisterende WS:** Ingen ny infrastruktur, ingen nye
endpoints, ingen ændringer i auth-flow. Backenden er allerede en dumb relay
for rummet — denne tilgang udvider det princip til signalering.

**Broadcast + client-side filtrering:** Enklere end per-bruger kanaler.
I et DM-rum med 2 deltagere er overhead-et én ekstra besked som ignoreres af
afsenderen selv. Acceptabelt.

**`from` tilføjes af backenden:** Forhindrer at en klient kan udgive sig for
at være en anden bruger i signaleringen og dermed manipulere peer-forbindelsen.

**TURN med `--use-auth-secret`:** `--lt-cred-mech` med `--user=username:password`
virker ikke med plaintext passwords i nyere coturn — serveren forventer MD5 af
`username:realm:password` som nøgle. `--use-auth-secret` med HMAC-SHA1 er den
korrekte og anbefalede tilgang til WebRTC TURN-autentificering.

**Tidsbegrænsede TURN-credentials:** Credentials udløber efter 24 timer og
genereres i browseren via SubtleCrypto. Selv om TURN_SECRET er synlig i
frontend-JS, begrænser TTL på 24 timer misbrugspotentialet.

## Konsekvenser

- WS-protokollen udvides med to nye beskedtyper: `signal` (klient→server→klient)
  og `screen-share-started`/`screen-share-stopped` (til UI-indikation).
- Eksisterende chat-funktionalitet er uberørt — `message`-typen ændres ikke.
- `coturn` er en ny prod-service der kræver ports 3478 (UDP+TCP) og 49152-49200
  (UDP relay) åbne i firewallen. `TURN_SECRET` opbevares i Ansible vault.
- `frontend/config.js` er ikke længere rent statisk — den renders af Ansible fra
  `templates/config.js.j2` ved hvert deploy og må ikke redigeres direkte på VPS.
- Fase 2 (gruppe-huddles) kræver ny ADR og muligvis per-bruger kanaler i stedet
  for broadcast-filtrering.
