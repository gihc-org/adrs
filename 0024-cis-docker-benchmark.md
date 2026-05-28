# 0024 — CIS Docker Benchmark som baseline for containerhærdning

**Status:** Accepted  
**Dato:** 2026-05-05  
**Udskilt fra:** ADR-0014

## Kontekst

En container der kører som root og kompromitteres giver angriberen root-adgang
til host-systemet via container escape. CIS Docker Benchmark er gratis,
velvedligeholdt og giver konkrete, målbare hærdningsanbefalinger frem for
abstrakte principper.

De tre vigtigste mitigations er lav-effort og bør anvendes som standard på
alle nye services.

## Beslutning

### Alle nye services i docker-compose

```yaml
security_opt:
  - no-new-privileges:true
```

Forhindrer at processer inde i containeren kan eskalere privilegier via
`setuid`/`setgid`-binaries.

### Backend-containere der ikke skriver til filsystem

```yaml
read_only: true
```

Kombinér med eksplicitte `tmpfs`-mounts til de stier der faktisk kræver skrivning:

```yaml
tmpfs:
  - /tmp
```

### Nye Dockerfiles til applikations-services

Kør som non-root. Databaser og reverse proxies håndteres af officielle images
og er undtaget.

```dockerfile
RUN useradd -m --no-log-init appuser
USER appuser
```

### Resource limits ved kendte workloads

Sæt eksplicitte grænser når workload-profilen er kendt. Forhindrer at én
runaway-container tager hele hosten ned:

```yaml
deploy:
  resources:
    limits:
      memory: 256m
      cpus: "0.5"
```

Juster tallene til det konkrete projekt — ovenstående er et udgangspunkt,
ikke en universel grænse.

### Image-scanning

`docker scout cves <image>` eller `trivy image <image>` køres:
- Inden første deploy af et nyt image
- Ved større dependency-opdateringer
- Ved mistanke om sårbare afhængigheder

## Begrundelse

Non-root + `no-new-privileges` + `read_only` er tre uafhængige lag der
eliminerer forskellige angrebsklasser. Kombineret kræver en angriber der
kompromitterer en container at finde en kernel-sårbarhed for at eskalere
— frem for blot at udnytte default root-adgang.

CIS Benchmark dækker mange flere punkter. Denne ADR beskriver de tre der
giver størst effekt pr. implementeringsindsats. Den fulde benchmark anvendes
som periodisk review-checkliste, ikke som implementeringskrav per commit.

## Konsekvenser

- Alle nye `docker-compose.yml`-services skal have `no-new-privileges`.
- Alle nye Dockerfiles til applikations-services skal have en non-root bruger.
- `read_only: true` tilføjes til services der kan køre uden filsystem-skrivning.
- Image-scanning er ikke automatiseret i CI endnu — det er en åben opgave.
