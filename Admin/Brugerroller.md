---
title: Brugerroller
parent: Admin
---

# {{ page.title }}
OpenWebUI har to roller: Administrator og bruger. Aarhus AI har udviddet dette med en rolle vi kalder "builder". Dette har vi gjort fordi vi har et behov for at nogle brugere kan anvende specialister og assistener, og andre kan bygge assistenter, men ikke har administrator rettigheder.
En Builder i Aarhus AI er en brugergruppe der giver brugeren flere rettigheder. I OpenWebUI giver man rettigheder til brugergrupper og ikke til roller. Derfor har vi oprettet en gruppe der hedder "builder" der har udviddet rettighederne for de brugere der er i gruppen. 

Der en patch der i brugeroverblikket viser rollen "Builder". Derudover har vi i vores OIDC patch lavet en mapning fra rollen "Builder" (som vi har i vores AD) til OpenWebUI så en bruger der kommer med rollen "Builder" placeres i gruppen "builder" i OpenWebUI.
