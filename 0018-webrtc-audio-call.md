# 0018 — WebRTC lyd-opkald i DM-rum

**Status:** Accepted  
**Dato:** 2026-05-08  
**Projekt:** ipfs-apps/chat

## Kontekst

DM-rum har allerede WebRTC-infrastruktur til skærmdeling (ADR-0016): signal-
forwarding i backend, TURN-server og en `RTCPeerConnection` per session. Et
naturligt næste skridt er lyd-opkald — `getUserMedia({ audio: true })` i stedet
for `getDisplayMedia()`.

To arkitektoniske spørgsmål adskiller opkald fra skærmdeling:

**1. Ringesignal-flow:** Skærmdeling starter ensidigt (shareren bestemmer).
Et opkald kræver samtykke fra modtageren — der er behov for et
invite/accept/reject-flow inden WebRTC-forbindelsen oprettes.

**2. Forholdet til skærmdeling:** Opkald og skærmdeling skal kunne køre
parallelt. Det kræver en beslutning om `RTCPeerConnection`-arkitekturen.

## Beslutning

### Fase 1: lyd-only

`getUserMedia({ audio: true })` — ingen kamera. Fase 2 (kamera-video) behandles
i en separat ADR.

### Delt RTCPeerConnection via renegotiering

Opkald og skærmdeling deler én `RTCPeerConnection`. Når et opkald startes
tilføjes lyd-track'et til den eksisterende PC via `addTrack()`, og en ny
offer/answer-runde (renegotiering) gennemføres. Når opkaldet slutter fjernes
track'et igen.

Alternativet — to separate PC'er — kræver dobbelt TURN-allokering og separat
ICE-kandidat-udveksling, og giver ingen fordele for 1:1-rummet.

Konsekvens: `RTCPeerConnection` og dens levetid ejes af skærmdelings-logikken.
Opkalds-logikken *låner* den eksisterende PC og renegotierer, når den har brug
for det. Hvis der ikke er en aktiv PC opretter opkaldet en ny.

### Ringesignal-flow

```
Caller                   Server                   Callee
  |                        |                        |
  |-- call-invite -------> |-- broadcast ---------->|
  |                        |              [ringer]   |
  |                        |<-- call-accept ---------|   (eller call-reject)
  |<-- broadcast ----------|                         |
  |                        |                         |
  |-- offer (WebRTC) ----> |-- broadcast ----------->|
  |<----------- answer + ICE-kandidater -------------|
  |                        |                         |
  | <============= lyd-forbindelse =================>|
  |                        |                         |
  |-- call-end ----------->|-- broadcast ----------->|
```

Signaltyper (alle via eksisterende `signal`-mekanisme i WS):

| Type | Afsender | Betydning |
|------|----------|-----------|
| `call-invite` | Caller | Ønsker at ringe op |
| `call-accept` | Callee | Accepterer opkald |
| `call-reject` | Callee | Afviser opkald |
| `call-end` | Begge | Afslutter aktivt opkald |
| `offer` | Caller | WebRTC SDP offer (efter accept) |
| `answer` | Callee | WebRTC SDP answer |
| `ice-candidate` | Begge | ICE-kandidater |

`call-end` sendes af den part der lægger på — den anden part rydder op lokalt.

### Timeout og mistet opkald

Hvis callee ikke svarer inden 30 sekunder sender caller automatisk `call-end`
og viser "Intet svar". Ingen opkaldshistorik gemmes i databasen — mistet opkald
vises kun som en kortvarig UI-besked.

### Backend

Ingen ændringer. De nye signal-typer videresendes uændret af den eksisterende
`signal`-håndtering i `src/routes/chat.rs`.

### Frontend-UI

- **"Ring op"-knap** i DM-nav (ved siden af "Del skærm") — kun synlig når
  `isDm && peerId`
- **Indgående opkald**: overlay med callers navn, ringetone via Web Audio API,
  "Accepter" og "Afvis"-knapper
- **Aktivt opkald**: statuslinje med varighed, "Dæmp mikrofon"-toggle og
  "Læg på"-knap
- Opkald og skærmdeling kan køre parallelt — begge knapper forbliver tilgængelige

## Begrundelse

**Delt PC frem for to separate:** Én PC er tilstrækkeligt for to medietyper i
et 1:1-rum. Renegotiering (`onnegotiationneeded`) er veldefineret i moderne
browsere. To PC'er ville fordoble TURN-belastningen og kompleksere signalflow.

**Offer sendes efter accept:** Caller sender ikke WebRTC offer med det samme —
først når callee accepterer. Herved undgås ressourcespild (ICE-gathering,
TURN-allokering) for opkald der afvises, og modtageren slipper for at se en
WebRTC-forbindelsesforsøg inden de har taget stilling.

**Ingen opkaldshistorik i DB:** Formålet er direkte kommunikation, ikke
kommunikationslog. GDPR-implikationerne ved at gemme opkaldsmetadata
(hvem ringede til hvem, hvornår) er større end fordelen.

## Konsekvenser

- `RTCPeerConnection`-logikken i `frontend/chat.html` refaktoreres til at
  understøtte dynamisk tilføjelse/fjernelse af tracks og renegotiering.
- `ICE_SERVERS` og `addTurnServer()` bruges uændret af begge features.
- Fase 2 (kamera) kræver ny ADR — `getUserMedia` med video og `<video>`-element
  til modtager-siden.
- Ringetone kræver en lydfil eller Web Audio API-genereret tone i `frontend/`.
