# 0018 — WebRTC lyd-opkald: ringesignal-flow og delt RTCPeerConnection

**Status:** Accepted  
**Dato:** 2026-05-08

## Kontekst

En applikation har allerede WebRTC-infrastruktur til en funktion (f.eks.
skærmdeling, ADR-0016): signal-forwarding i backend, TURN-server og en
`RTCPeerConnection` per session. Lyd-opkald bygger oven på denne infrastruktur.

To arkitektoniske spørgsmål adskiller opkald fra skærmdeling:

**1. Ringesignal-flow:** Skærmdeling starter ensidigt. Et opkald kræver
samtykke fra modtageren — invite/accept/reject-flow inden WebRTC-forbindelsen
oprettes.

**2. Forholdet til eksisterende RTCPeerConnection:** Opkald og eksisterende
WebRTC-features skal kunne køre parallelt.

## Beslutning

### Delt RTCPeerConnection via renegotiering

Opkald og eksisterende WebRTC-features deler én `RTCPeerConnection`.
Lyd-track'et tilføjes via `addTrack()` og en ny offer/answer-runde
(renegotiering) gennemføres. Når opkaldet slutter fjernes track'et igen.

To separate PC'er kræver dobbelt TURN-allokering og separat ICE-kandidat-
udveksling uden fordele for 1:1-kommunikation.

### Ringesignal-flow

```
Caller                   Server                   Callee
  |                        |                        |
  |-- call-invite -------> |-- broadcast ---------->|
  |                        |              [ringer]   |
  |                        |<-- call-accept ---------|
  |<-- broadcast ----------|                         |
  |-- offer (WebRTC) ----> |-- broadcast ----------->|
  |<----------- answer + ICE-kandidater -------------|
  | <============= lyd-forbindelse =================>|
  |-- call-end ----------->|-- broadcast ----------->|
```

Signaltyper (alle via eksisterende WebSocket-signalmekanisme):

| Type | Afsender | Betydning |
|------|----------|-----------|
| `call-invite` | Caller | Ønsker at ringe op |
| `call-accept` | Callee | Accepterer opkald |
| `call-reject` | Callee | Afviser opkald |
| `call-end` | Begge | Afslutter aktivt opkald |
| `offer` | Caller | WebRTC SDP offer (sendes efter accept) |
| `answer` | Callee | WebRTC SDP answer |
| `ice-candidate` | Begge | ICE-kandidater |

**Offer sendes efter accept** — ikke ved invite. Undgår ressourcespild
(ICE-gathering, TURN-allokering) for afviste opkald.

Timeout: caller sender `call-end` automatisk efter 30 sekunder uden svar.

## Testkrav

- **Unit tests:** Tilstandsmaskinen for opkalds-flow (invite → accept/reject/timeout → end) testes med mockede signal-callbacks. Verificér at alle state-transitions er korrekte og at timeout-logik affyrer som forventet.
- **Integration tests:** Backend's signal-forwarding verificeres — `call-invite` sendt af A skal modtages af B i samme rum. Ukorrekte signal-typer skal ignoreres eller returnere fejl.
- **E2e tests:** To Playwright-browsere etablerer et lyd-opkald end-to-end. Verificér at `RTCPeerConnection.connectionState` når `"connected"`.

## Sikkerhed

- **TURN-credentials roteres periodisk** — brug short-lived TURN credentials (HMAC-baserede) frem for statiske credentials. Et kompromitteret credential giver kun adgang til TURN-relaying, ikke til applikationsdata.
- **ICE-lækage:** `getUserMedia` og ICE-kandidat-indsamling kan afsløre brugerens lokale IP-adresse. Brug `iceTransportPolicy: "relay"` hvis anonymitet er et krav.
- **Opkaldshistorik gemmes ikke i DB** — GDPR-implikationerne ved at gemme opkaldsmetadata (hvem ringede til hvem, hvornår) overstiger fordelen for direkte kommunikation.

## Konsekvenser

- `RTCPeerConnection`-logikken refaktoreres til at understøtte dynamisk tilføjelse/fjernelse af tracks og renegotiering.
- `ICE_SERVERS`-konfigurationen genbruges af begge features.
- Kamera-video behandles i en separat ADR — `getUserMedia({ video: true })` og `<video>`-element til modtagersiden.
