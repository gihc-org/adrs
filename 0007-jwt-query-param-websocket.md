# 0007 — JWT via query-parameter til WebSocket-auth

**Status:** Accepted  
**Dato:** 2026-05-05  
**Projekt:** ipfs-apps/chat

## Kontekst

WebSocket-forbindelsen (`GET /ws/:room_id`) kræver autentifikation. REST-
endpoints bruger `Authorization: Bearer <token>`-header, men browser-WebSocket
API'et (`new WebSocket(url)`) understøtter ikke custom headers på
upgrade-request.

Alternativer:
1. **Query-parameter:** `wss://api.example.com/ws/room-id?token=<jwt>`
2. **First-message auth:** Første besked efter connect er et auth-payload.
3. **Cookie:** Session-cookie sendes automatisk med upgrade-request.

## Beslutning

JWT sendes som **`?token=<jwt>` query-parameter** på WebSocket URL.

```
WS /ws/:room_id?token=<jwt>
```

Handler validerer token ved connect, afviser med `close(4001)` hvis ugyldigt.

## Begrundelse

- **Browser-begrænsning:** `new WebSocket()` tager kun en URL — ingen headers.
- **Simpelhed:** Query-param er tilgængeligt i URL fra første byte af
  upgrade-request. Ingen ekstra round-trip (first-message kræver at connection
  er established før auth).
- **Cookie-alternativet** kræver at cookie-domænet matcher API-domænet — det
  komplicerer CORS-opsætningen med IPFS-frontend på anden origin.

## Konsekvenser

- **JWT eksponeres i server-logs** (URL logges typisk). Mitigation: token har
  kort levetid (konfigurérbar via `JWT_EXPIRE_HOURS`), og logs bør saniteres
  eller have begrænset adgang.
- **JWT eksponeres i browser-history** hvis URL gemmes. Acceptabelt da token
  er kortlivet.
- Frontend sender token som query-param:
  ```js
  new WebSocket(`wss://api.example.com/ws/${roomId}?token=${token}`)
  ```
- REST-endpoints fortsætter med `Authorization: Bearer` — ingen ændring.
