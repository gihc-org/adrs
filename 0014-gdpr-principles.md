# 0014 — GDPR-principper som tilbagevendende standard

**Status:** Accepted  
**Dato:** 2026-05-05  
**Erstatter:** Original 0014 (som også indeholdt CIS Docker — nu ADR-0024)

## Kontekst

Projekter der gemmer persondata om EU-borgere er underlagt GDPR uanset
projektets størrelse. De grundlæggende krav (sletning, indsigt, oplysning,
dataminimering) er enkle at implementere fra starten og bør ikke udskydes.

Bøder for overtrædelse kan udgøre op til 4 % af global omsætning eller
20 mio. EUR. Selv for et lille open source-projekt er det et risiko der
skal adresseres.

## Beslutning

Nye features der berører persondata vurderes mod følgende principper inden
implementering:

### Dataminimering

Gem kun det der er strengt nødvendigt for featuren. Spørg ved hvert felt:
"Kan applikationen fungere uden dette?" Brugernavn og e-mail er typisk
nødvendige til auth og verifikation. Fødselsdato, telefonnummer og
lokaliseringsdata kræver en eksplicit feature-begrundelse.

### Ret til sletning

Et endpoint til sletning af brugerens konto og alle tilknyttede data skal
implementeres (`DELETE /auth/me` eller tilsvarende). Se ADR-0015 for
mønsteret til korrekt cascade- og eksplicit sletning.

Krav til implementeringen:
- Sletning er permanent og ikke-reversibel (ingen soft delete medmindre
  der er en konkret begrundelse)
- Alle tilknyttede data slettes i samme transaktion eller i korrekt rækkefølge
- Endpointet testes med assertions på at data faktisk er væk (se ADR-0015)

### Ret til indsigt

Et endpoint der returnerer alle gemte felter for den autentificerede bruger
(`GET /auth/me` eller tilsvarende). Skal returnere *alle* felter der gemmes
om brugeren — ikke kun visningsnavn og id.

### Oplysningspligt

Brugere skal oplyses om hvad der gemmes og hvorfor — minimum en
privacy policy-side eller et `PRIVACY.md` i repoet for open source-projekter.

### Verifikationstoken og midlertidig data er persondata

Tokens til e-mail-verifikation, password-reset og lignende er persondata.
De slettes straks ved brug og ved sletning af konto.

## Konsekvenser

- Hvert projekt med bruger-auth skal have et sletnings-endpoint.
- GDPR-vurdering tilføjes til definition of done for features der berører persondata.
- Se ADR-0015 for det konkrete sletnings-mønster.
- Se ADR-0024 for CIS Docker-hærdning der supplerer GDPR på infrastrukturniveau.
