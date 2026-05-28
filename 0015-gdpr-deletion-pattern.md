# 0015 — GDPR-sletning: cascade og eksplicit oprydning

**Status:** Accepted  
**Dato:** 2026-05-06  
**Erstatter:** Original 0015 (som også indeholdt ipfs-apps-specifik smoke test — nu ADR-0025)  
**Forudsætter:** ADR-0014 (GDPR-principper)

## Kontekst

ADR-0014 kræver at alle projekter med bruger-auth implementerer et
sletnings-endpoint. Implementeringen er ikke triviel: databaseskemaer med
relationer kræver at sletningsrækkefølgen overvejes eksplicit. Cascade-FK
håndterer ikke alle tilfælde.

## Beslutning

### Cascade er ikke tilstrækkeligt

`ON DELETE CASCADE` sletter kun rækker med en direkte FK til den slettede
bruger. Enheder der *refererer til brugeren indirekte* — f.eks. via en
many-to-many-tabel eller en entitet der kan have flere ejere — slettes ikke
automatisk.

**Eksempel:** Et "rum" med flere medlemmer har ingen direkte FK til én bruger.
Cascade på `room_members`-tabellen sletter brugerens medlemskab, men ikke
selve rummet — selv om rummet nu er tomt eller kun indeholder slettede brugere.

### Sletningsrækkefølge

Implementér sletnings-handleren i denne rækkefølge:

1. **Eksplicit sletning af entiteter uden direkte ejer-FK** der efterlades
   forældreløse ved brugerens bortgang. Identificér disse ved at gennemgå
   skemaet for entiteter der *kun* giver mening i relation til én bruger,
   men ikke har en direkte FK til `users`.

2. **Slet brugeren.** `ON DELETE CASCADE` håndterer herefter alle entiteter
   med direkte FK.

Hele sekvensen køres i én transaktion. Fejler ét trin rulles alt tilbage.

### Test-krav

Sletnings-endpointet skal have integrationstests der verificerer:

- [ ] Brugeren eksisterer ikke efter sletning (`GET /auth/me` → 401)
- [ ] Direkte tilknyttede data er væk (verificér via DB-query eller API)
- [ ] Forældreløse entiteter er ryddet op (verificér eksplicit)
- [ ] Sletning af ikke-eksisterende bruger returnerer korrekt statuskode
- [ ] Efterfølgende login-forsøg fejler

### Verifikations- og reset-tokens

Kortlivede tokens (e-mail-verifikation, password-reset) behandles som
persondata og slettes:
- Straks ved brug (uanset succes eller fejl)
- Samlet ved sletning af konto

## Begrundelse

Forældreløse entiteter i databasen er et stille dataintegritetsproblem:
de er usynlige i applikationen men tager plads, kan lække data ved fejl
i fremtidige queries, og er svære at rydde op i efterhånden.

Eksplicit sletning som trin 1 frem for at stole på cascade-kæder gør
rækkefølgen synlig i koden og tvinger reviewers til at tage stilling til
om alle entiteter er håndteret.

## Konsekvenser

- Sletnings-handleren skal gennemgå skemaet og identificere alle indirekte
  relationer inden implementering.
- Skema-ændringer der tilføjer nye tabeller med relation til brugere kræver
  en vurdering af om sletnings-handleren skal opdateres.
- Se ADR-0025 for et eksempel på projekt-specifik smoke test der verificerer
  sletnings-flowet i produktion.
