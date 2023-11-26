# Onderzoek Ethical Hacking: HTTP/2 Rapid Reset Attack (CVE-2023-44487)

## Introductie

In dit onderzoek gaan wij kennis maken met de Denial Of Service aanval genaamd **HTTP/2 rapid Reset**. Een aanval dat is ontdekt in oktober 2023 en volgens verschillende bronnen nog enkele jaren sporadisch zal opduiken. We hebben deze aanval gekozen wegens zijn impact: alle webservers die het protocol HTTP/2 zijn namelijk vatbaar voor deze aanval, indien deze niet zijn voorzien van de correcte patch.

In dit onderzoek zullen de volgende elementen aan bod komen

- HTTP/2 Rapid Reset uitgelegd
- Waarom werkt deze aanval?
- Wat kunnen we tegen deze aanval doen?
- Demonstratie aanval

## HTTP/2 Rapid Reset uitgelegd

_Deze sectie is gebaseerd op het grondig verrichtte onderzoek van Juho Snellman en Daniele Lamartino. Link naar [website](https://cloud.google.com/blog/products/identity-security/how-it-works-the-novel-http2-rapid-reset-ddos-attack)_

### Algemene info

CVE-2023-44487 is een Volume Distributed Denial of Service. Dit betekent dat de aanval een zwakheid uitbuit die het mogelijk maakt om een systeem zijn resources in te palmen. Belangrijk om te begrijpen is dat deze aanval niet gegarandeerd een systeem doet falen.

Afhankelijk van de hoeveelheid resources de target machine heeft, is er een grote hoeveelheid aan computerkracht nodig om de aanval effectief te laten slagen. Denk maar aan 2 tot 10 tot duizenden computers.

Vooraleer we kunnen toelichten hoe de aanval werkt, is het belangrijk om te begrijpen hoe het HTTP/2 protocol werkt en welke eigenschap kwetsbaar is voor de aanval.

### Werking HTTP/2 streams


On normale omstandigheden, opent een client een **HTTP/2 stream** met behulp van een request. De HTTP/2 server beantwoordt deze request met een response. Als resultaat is een HTTP/2 stream opgemaakt en kunnen pakketten uitgewisseld worden.

HTTP/2 heeft een extra eigenschap namelijk "stream multiplexing": één de belangrijkste kenmerken van dit protocol. Het geeft namelijk de mogelijkheid om meerdere streams tegelijkertijd te onderhouden en resulteert in een efficiënter gebruik van de gemaakte TCP-connecties. 

Bij HTTP/1 zou voor elk verzoek een nieuwe TCP connectie opgezet moeten worden. Bij HTTP/2 hoeft dit dus niet. Stream multiplexing maakt het namelijk mogelijk om meerdere "in-flight" requests te versturen, zonder meerdere individuele connecties te moeten beheren.

Als we dieper ingaan op hoe HTTP/2 connecties werken, kunnen we vertellen dat een client de mogelijkheid heeft om meerdere streams tegelijkertijd te starten in een TCP-connectie. Over elke stream wordt dan een HTTP verzoek verzonden. Dit principe wordt zeer belangrijk wanneer we verder gaan bekijken hoe deze denial of service werkt.

### Verschil normale DOS en HTTP/2 Rapid Reset

Laten we kort bekijken wat het verschil is tussen de Rapid Reset DOS en een normale HTTP DOS.

De onderstaande foto geeft het principe van de verschillende aanvallen duidelijk weer:

![Rapid reset diagram example](Images/rapidResetExample.png)

De eerste en de tweede aanval lijken sterk op elkaar: een client start een connectie met de server, de server antwoord hierop. Het antwoorden op zo'n request kost computerkracht. 

Wat er typisch gebeurt tijdens zo'n aanval, is dat de aanvaller een zeer groot aantal requests stuurt met behulp van een groot aantal machines (ook wel een botnet genoemd). De server probeert tevergeefs al deze verzoeken te beantwoorden, maar bezwijkt onder de druk. Als dit gebeurt, is de aanval succesvol.

Het enige verschil tussen de HTTP/1.1 en HTTP/2 aanval is dat het HTTP/2 protocol parallelle verzoeken accepteert door het ondersteunen van meerdere streams.

Nu komt de belangrijke eigenschap die de Rapid Reset aanval zo sterk maakt: in beide voorgaande aanvallen, antwoordde de server en wachtte de client op de request. **Bij Rapid Reset is dit niet het geval!**

### Waarom werkt deze aanval

De Rapid Reset aanval steunt op een eenvoudig principe: de client gaat een zeer groot aantal streams creëren. Alleen hoeft de client niet te wachten op antwoord van de server. In plaats van te wachten, wordt gewoon nog een verzoek gestuurd voor een nieuwe stream en daarna nog één en nog één ...

Deze werkwijze is mogelijke doordat de client meteen na het sturen van zijn verzoek een een RST_STEAM uitstuurt. Een pakket dat aangeeft dat de client de connectie wilt afbreken.
Het "resetten" van deze stream kost echter resources voor de server, maar beïnvloedt de client niet. Als gevolg hiervan kan de client zeer veel streams starten en resetten. Vandaar de naam "HTTP/2 Rapid Reset"

Een bijkomend voordeel is dat het annuleren van een connectie ervoor zorgt dat het aantal open streams niet verhoogt. Waardoor de limiet van max toegelaten streams niet wordt overschreven.

Tot slot is er nog een voordeel dat deze aanval met zich meeneemt: door het direct annuleren van de verzoeken, stuurt de reverse proxy van de server geen antwoord. Dit zorgt ervoor dat de hacker niet moet beschikken over grote bandbreedtes. Dit zou namelijk wel nodig zijn om de antwoorden van de server te kunnen slikken in klassieke aanvallen.
## Wat kunnen we tegen deze aanval doen

## Demonstratie

In onze demonstratie maken we gebruiken van een oudere nginx versie namelijk 1.23.2 deze is ongeveer 1 jaar oud.

**De POC voor deze aanval gebruikt de code van se**

Op de onderstaande foto kan u zien dat de aanval 
![POC attack output](Images/attackerView.png)

De onderstaande foto laar zien dat de aanval veel resources van de computer heeft gevraagd. Doordat de swap partitie in actie is moeten schieten, betekent dat de aanval minstens 15 GB aan ram heeft gekost. Daarnaast waren ook hoge CPU waarden aanwezig tijdens de aanval zelf.


![Indication high RAM Usage](Images/swapUsed.png)

Tijdens sommige momenten ondervonden we zelfs dat de onze computer de DOS uitschakelde om de CPU minder te belasten.

De onderstaande foto laat het effect zien dat 1 computer heeft op een nginx webserver die HTTP/2 draait. We zien dat de container zijn CPU resource tot +- 20 procent werden bezet. We kunnen dus aannemen, wanneer we beschikken over meer computerkracht (circa 5 tot 7 computers) dat de aanval succesvol de container kan overbelasten.

![Alt text](Images/cpuUsage.png)

We kunnen dus aannemen dat de DOS correct heeft gerund, aangezien één computer in normale omstandigheden nooit zo veel resources van een webserver vereist. Tot slot kunnen we vaststellen dat de DOS enkel impact had op CPU, wat onze bevindingen in de vorige onderdelen van ons onderzoek bevestigt. 
## Bronnen

- Snellman, J., & Iamartino, D. (2023, 10 oktober). How it works: the novel HTTP/2 ‘Rapid Reset’ DDoS attack. _Google Cloud Blog_. https://cloud.google.com/blog/products/identity-security/how-it-works-the-novel-http2-rapid-reset-ddos-attack

- Pardue, L. (2023, 27 oktober). _HTTP/2 rapid reset: Deconstructing the record-breaking attack_. The Cloudflare Blog. https://blog.cloudflare.com/technical-breakdown-http2-rapid-reset-ddos-attack/
