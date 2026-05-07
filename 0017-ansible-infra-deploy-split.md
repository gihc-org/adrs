# 0017 — Opdeling af Ansible-playbook i infrastruktur og applikationsdeploy

**Status:** Accepted  
**Dato:** 2026-05-07  
**Projekt:** ipfs-apps/chat

## Kontekst

Projektet bruger Ansible til både serveropsætning og applikationsdeploy. Oprindeligt lå alt i én `playbook.yml`. To behov pressede på for en opdeling:

1. **CI/CD (Woodpecker):** En pipeline der automatisk deployer ved push må kun køre applikationsdeployments — ikke geninstallere Docker eller ændre OS-opsætning. Infrastructure-trin kræver bevidst manuel beslutning.

2. **Hastighed og risiko:** Docker-installation og apt-opdateringer er langsomme og sjældent nødvendige. At køre dem ved hvert release er spild og introducerer unødig risiko for at infra-ændringer rammer prod uventet.

## Beslutning

Playbook'en opdeles i tre filer:

- **`ansible/infra.yml`** — Infrastrukturopsætning: apt-pakker, Docker-installation, projektmappe. Køres manuelt ved serveropsætning eller når OS/Docker-konfigurationen skal ændres.
- **`ansible/deploy.yml`** — Applikationsdeploy: rsync, Caddyfile, .env, `docker compose up --build`, IPFS-upload, DNS-opdatering, smoke test, ZAP-scan. Køres ved hvert release — manuelt og via CI/CD.
- **`ansible/playbook.yml`** — Convenience-wrapper der importerer begge: bruges kun ved første gangs provisioning af en ny server.

```
ansible-playbook ansible/infra.yml -i ansible/inventory.yml --ask-vault-pass   # ny server / infra-ændring
ansible-playbook ansible/deploy.yml -i ansible/inventory.yml --ask-vault-pass  # hvert release
ansible-playbook ansible/playbook.yml -i ansible/inventory.yml --ask-vault-pass # første gang
```

CI/CD (Woodpecker) kalder udelukkende `deploy.yml`.

## Begrundelse

**Separation of concerns:** Infrastruktur ændres sjældent og kræver root-adgang til OS-niveau operationer. Applikationsdeploy sker hyppigt og bør være idempotent og hurtigt. At blande dem i ét script gør begge dele sværere at vedligeholde.

**Sikkerhed i CI/CD:** En pipeline der kun kan køre `deploy.yml` kan ikke ved fejl eller kompromittering rekonfigurere Docker-daemonen, ændre apt-sources eller lignende. Infra-ændringer kræver bevidst menneskelig handling.

**Hastighed:** `deploy.yml` uden infra-trin er markant hurtigere — ingen apt-cache-opdatering, ingen Docker-GPG-nøgle-download, ingen pakkeinstallation.

## Konsekvenser

- `deploy.yml` forudsætter at `infra.yml` er kørt mindst én gang på serveren. Denne afhængighed er ikke teknisk håndhævet — det er dokumenteret i kommentarerne i begge filer og i `runbooks/deploy.md`.
- Nye infrastruktur-behov (fx ny systempakke eller Docker-plugin) skal tilføjes i `infra.yml` og køres manuelt — de må ikke stilles i `deploy.yml`.
- `playbook.yml` er udelukkende en convenience-wrapper og må ikke indeholde egne tasks.
