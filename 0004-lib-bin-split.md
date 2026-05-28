# 0004 — Adskil applikationslogik fra entry-point (lib+bin-split)

**Status:** Accepted  
**Dato:** 2026-05-05

## Kontekst

Applikationslogik placeret direkte i entry-pointet (`main.rs`, `__main__.py`,
`main.go`) kan ikke importeres af integrationstests — tests kan kun starte
processen som en sort boks. Det gør det vanskeligt at skrive tests der
verificerer intern logik uden at starte en fuld server.

Mønsteret er sprog-uafhængigt:

| Sprog | Entry-point | Library/testbar logik |
|-------|-------------|----------------------|
| Rust | `src/main.rs` | `src/lib.rs` (library crate) |
| Python | `__main__.py` | `src/app.py` (`create_app()`) |
| Go | `cmd/server/main.go` | `internal/app/app.go` |
| Node.js | `index.js` | `src/app.js` (`createApp()`) |

## Beslutning

**Entry-pointet er tyndt.** Det læser konfiguration, opretter eksterne
afhængigheder (database pool, config) og kalder ind i library-modulet.
Al applikationslogik (router, middleware, handlers, state) lever i
library-modulet og kan importeres af tests.

### Rust-eksempel

```
src/
  lib.rs      ← library root: alle moduler, build_app(), AppState
  main.rs     ← binary: læs config, opret pool, kald build_app()
tests/
  api.rs      ← integration tests: use mycrate::build_app;
```

```toml
# Cargo.toml
[lib]
path = "src/lib.rs"

[[bin]]
name = "server"
path = "src/main.rs"
```

### Python-eksempel (Flask/FastAPI)

```python
# src/app.py
def create_app(config=None):
    app = Flask(__name__)
    ...
    return app

# __main__.py
from src.app import create_app
app = create_app()
app.run()
```

## Begrundelse

- **Integrationstests kan importere applikationen:** Tests spinner en
  test-server op med rigtig database og rigtig router — ingen mocks af
  interne komponenter.
- **Ingen duplikering:** Router-konfiguration, middleware og state lever ét
  sted og bruges af både produktion og tests.
- **Klar separation:** Entry-pointet håndterer kun I/O (læs env, bind port).
  Al logik er testbar uden at starte en rigtig server.

## Konsekvenser

- Nye moduler og handlers tilføjes i library-modulet — ikke i entry-pointet.
- Entry-pointet må ikke indeholde forretningslogik.
- Alt der bruges af integrationstests skal eksporteres (Rust: `pub`-visibility,
  Python: importerbar funktion, Go: eksporteret pakke).
