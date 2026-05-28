# 0017 — Opdel deployment i infrastruktur og applikation

**Status:** Accepted  
**Dato:** 2026-05-07

## Kontekst

Deployment-automatisering håndterer typisk to meget forskellige ting:

1. **Infrastruktur:** OS-pakker, Docker-installation, firewall-regler,
   brugere og rettigheder. Ændres sjældent, kræver root-adgang, er risikabelt
   at køre automatisk.
2. **Applikation:** Kode, konfigurationsfiler, containers, DNS. Ændres
   hyppigt, skal køre automatisk ved hvert release.

At blande dem i ét script giver langsom CI, unødig risiko og uklart ansvar.

## Beslutning

Del deployment i to separate scripts:

- **`infra`-script** — Serveropsætning: pakker, Docker, projektmappe,
  systembrugere. Køres **manuelt** ved ny server eller eksplicit
  infrastrukturændring. Aldrig automatisk fra CI.
- **`deploy`-script** — Applikationsdeploy: kode, config, containers,
  konfigurations-validering, smoke test. Køres automatisk af CI ved
  hvert release.
- **`full`-script (valgfrit)** — Convenience-wrapper der kalder begge.
  Bruges kun ved første gangs opsætning af en ny server.

### Ansible-eksempel

```
ansible/
  infra.yml      ← apt-pakker, Docker, projektmappe
  deploy.yml     ← rsync, .env, docker compose up, validate, smoke test
  playbook.yml   ← importerer begge — kun til første gangs provisioning
```

```bash
# Ny server / infra-ændring (manuelt)
ansible-playbook ansible/infra.yml -i inventory.yml --ask-vault-pass

# Hvert release (automatisk fra CI)
ansible-playbook ansible/deploy.yml -i inventory.yml --ask-vault-pass

# Første gangs provisioning (manuelt)
ansible-playbook ansible/playbook.yml -i inventory.yml --ask-vault-pass
```

CI-pipeline har kun adgang til at køre `deploy`-scriptet.

### Verificering i deploy-scriptet

`deploy`-scriptet bør tidligt verificere at infrastrukturen er på plads,
så fejlbeskeden er klar frem for en kryptisk downstream-fejl:

```yaml
# ansible/deploy.yml — tidligt check
- name: Verificér at Docker er installeret
  command: docker --version
  changed_when: false
  failed_when: false
  register: docker_check

- name: Fejl hvis Docker mangler
  fail:
    msg: "Docker er ikke installeret. Kør infra.yml først."
  when: docker_check.rc != 0
```

## Begrundelse

**Sikkerhed i CI:** En pipeline der kun kan køre `deploy`-scriptet kan ikke
ved fejl eller kompromittering rekonfigurere Docker-daemonen, ændre
apt-sources eller installere systempakker. Infra-ændringer kræver bevidst
menneskelig handling.

**Hastighed:** `deploy`-scriptet uden infra-trin er markant hurtigere —
ingen apt-cache-opdatering, ingen pakkeinstallation ved hvert release.

**Klarhed:** Opdelingen gør det åbenlyst hvilke ændringer der kræver manuel
koordination (infra) og hvilke der er automatiske (deploy).

## Konsekvenser

- `deploy`-scriptet forudsætter at `infra`-scriptet er kørt mindst én gang.
  Dokumentér denne afhængighed i `README.md` eller et runbook.
- Nye infrastruktur-behov (ny systempakke, Docker-plugin) tilføjes i
  `infra`-scriptet og køres manuelt — aldrig i `deploy`.
- Se `ci-cd.md` i guidelines for den fulde CI-pipeline der kalder `deploy`.
