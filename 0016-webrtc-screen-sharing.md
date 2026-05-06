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

### STUN — offentlige servere, ingen TURN

```javascript
const iceServers = [
  { urls: 'stun:stun.l.google.com:19302' },
  { urls: 'stun:stun.cloudflare.com:3478' },
];
```

TURN (relay-server der videresender mediaforbindelser bag symmetrisk NAT)
udelades i fase 1. Det rammer brugere bag restriktive firewalls, men TURN
kræver dedikeret infrastruktur og løbende driftsomkostninger. Omfanget
vurderes ved fase 2.

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

### Scope for fase 1

- Skærmdeling tilbydes kun i DM-rum (præcis 2 deltagere)
- Ingen video- eller lydopkald — kun `getDisplayMedia()` (skærm/vindue/fane)
- Forbindelsen er envejs: én deler, én ser
- Stop-knap afslutter `MediaStream`-tracks og lukker `RTCPeerConnection`

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

**Ingen TURN i fase 1:** TURN er infrastruktur der koster penge og kræver
vedligeholdelse. Vi accepterer at forbindelsen fejler for brugere bag
symmetrisk NAT og adresserer det hvis det viser sig at være et reelt problem.

## Konsekvenser

- WS-protokollen udvides med to nye beskedtyper: `signal` (klient→server→klient)
  og `screen-share-started`/`screen-share-stopped` (til UI-indikation).
- Eksisterende chat-funktionalitet er uberørt — `message`-typen ændres ikke.
- Fase 2 (gruppe-huddles) kræver ny ADR og muligvis per-bruger kanaler i stedet
  for broadcast-filtrering.
- Hvis TURN viser sig nødvendigt tilføjes det som en ny service i
  `docker-compose.yml` (f.eks. `coturn`) med tilhørende ADR.
