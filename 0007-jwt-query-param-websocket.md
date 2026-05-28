# 0007 — JWT via query-parameter til WebSocket-auth

**Status:** Accepted  
**Dato:** 2026-05-05

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

## Testkrav

Auth-logikken på WebSocket-connectet er sikkerhedskritisk. Følgende tests er
obligatoriske:

- [ ] Gyldigt token → forbindelse etableres (happy path)
- [ ] Manglende token → `close(4001)` (ingen stille accept)
- [ ] Ugyldigt token (tampered signature) → `close(4001)`
- [ ] Udløbet token → `close(4001)`
- [ ] Token til forkert bruger/rum → `close(4001)` hvis adgangskontrol er implementeret

```rust
#[sqlx::test]
async fn websocket_rejects_invalid_token(pool: PgPool) {
    let app = test_app(pool).await;
    let resp = app.ws("/ws/room-1?token=not.a.jwt").await;
    assert_eq!(resp.close_code(), Some(4001));
}
```

**Log-sanitering (krav):** Serverens access log må ikke indeholde JWT-tokens i
klartekst. Implementér sanitering ved at fjerne `token`-parameteren fra loglinjen
inden skrivning — brug en regex af typen `token=[^&\s]+` → `token=[REDACTED]`.
Manglende sanitering klassificeres som CWE-598 (Information Exposure Through
Query Strings in GET Request).

Test at sanitering virker:

```rust
#[test]
fn token_query_param_is_redacted_in_log() {
    let url = "/ws/room-1?token=eyJhb...secret";
    let sanitized = sanitize_log_url(url);
    assert!(!sanitized.contains("eyJhb"));
    assert!(sanitized.contains("token=[REDACTED]"));
}
```

## Konsekvenser

- **JWT eksponeres i server-logs** uden sanitering (CWE-598). Log-sanitering
  af `token`-parameteren er et krav, ikke en anbefaling.
- **JWT eksponeres i browser-history** hvis URL gemmes. Acceptabelt da token
  er kortlivet (konfigurérbar via `JWT_EXPIRE_HOURS`).
- Frontend sender token som query-param:
  ```js
  new WebSocket(`wss://api.example.com/ws/${roomId}?token=${token}`)
  ```
- REST-endpoints fortsætter med `Authorization: Bearer` — ingen ændring.
