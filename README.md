# Architecture Decision Records

Samling af ADRs (Architecture Decision Records) fra `ipfs-apps`-projektet.
Bruges som reference ved nye projekter, der følger samme principper.

Format: [Michael Nygard's ADR-template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

## Index

| Nr. | Titel | Status |
|-----|-------|--------|
| [0001](0001-axum-over-actix.md) | Axum som web-framework frem for Actix-web | Accepted |
| [0002](0002-sqlx-runtime-api.md) | SQLx runtime API frem for compile-time query!-makro | Accepted |
| [0003](0003-rustls-over-native-tls.md) | rustls frem for native-tls | Accepted |
| [0004](0004-lib-bin-split.md) | Lib + bin-split for testbarhed | Accepted |
| [0005](0005-authenticate-plain-async-fn.md) | authenticate som plain async fn frem for FromRequestParts | Accepted |
| [0006](0006-broadcast-channel-per-room.md) | In-memory broadcast channel pr. rum til WebSocket fan-out | Accepted |
| [0007](0007-jwt-query-param-websocket.md) | JWT via query-parameter til WebSocket-auth | Accepted |
| [0008](0008-ipfs-dnslink-frontend.md) | IPFS + DNSLink til statisk frontend-hosting | Accepted |
| [0009](0009-caddy-reverse-proxy.md) | Caddy som reverse proxy med automatisk TLS | Accepted |
| [0010](0010-ansible-templates-caddyfile.md) | Ansible-templates til Caddyfile frem for env vars | Accepted |
| [0011](0011-argon2id-passwords.md) | Argon2id til password-hashing | Accepted |
| [0012](0012-feature-flags-via-empty-env.md) | Tom env-variabel som feature flag til lokal udvikling | Accepted |
| [0013](0013-owasp-by-default.md) | OWASP-sikkerhed som del af definition of done | Accepted |
| [0014](0014-gdpr-and-cis-docker.md) | GDPR og CIS Docker Benchmark som tilbagevendende standarder | Accepted |
