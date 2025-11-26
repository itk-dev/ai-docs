---
title: Viden
parent: Builder
---

# {{ page.title }}
Viden kan anvendes til at gøre sprogmodellen klogere på et område ved at berige den med en række dokumenter. Når man uploader viden til OpenWebUI bliver det processeret af systemet. Den processering går ud på at dele dokumenterne op i "chunks" af 500 tegn, herefter bliver det lavet til tokens, der bliver til vektorer. Disse vektorer anvendes når du stiller et spørgsmål,  Dvs. når du spørger en specialist om noget der står i de uploadede dokumenter gør specialisten som den "plejer" den bryder dit spørgsmål op i tokens og finder de vektorer i viden der passer bedst og sender det hele med til sprogmodellen. Dette betyder at den ikke nødvendigvis finder det man forventede, da det ikke er en søgning der sker.

## Eksempel
Du uploader et dokument med viden om kontaktpersoner til forskellige afdelinger. Der kunne stå:
### Digitalisering
Person A 

Telefon nummer: 12345678

Det tilknytter du en specialist, og spørger så "Hvad er telefonnummeret til kontaktpersonen i digitalisering", og får svaret at det kan specialisten ikke finde. Det sker fordi dokumentet har nogle tokens "Person A" "Telefonnummer: 1234567" "Digitalisering", de er blevet til vektorer, men "Digitalisering" og "Telefonnummer: 123456" ligger i to forskellige tokens og har ikke en relation til hinanden. Når du så spørger "Hvad er telefonnummeret til en kontaktperson i digitalisering" bliver det gjort til en række tokens der statistisk matcher "Telefonnummer: 12345678" og det sendes med videre til sprogmodellen, der forsøger via. statistisk mønstergenkendelse at give en rigtigt svar.
