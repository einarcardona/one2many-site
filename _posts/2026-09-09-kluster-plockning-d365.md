---
layout: post
title: "Klusterplockning i D365 SCM: så påverkar du inte den befintliga plockprocessen"
---

Klusterplockning gör det möjligt att plocka till flera order samtidigt från samma lagerplats — och kan spara betydande tid i en högvolymsmiljö. Men i ett nyligen genomfört WMS-projekt ville kunden bara aktivera klusterplockning för en avgränsad grupp artiklar, utan att på något sätt ändra hur den vanliga, manuella plockningen fungerar för allt annat.

Så här löste vi det — och två saker som var mindre uppenbara än de först verkade.

<!-- Skärmdump-förslag: enkel tvådelad bild — "Order släpps" som förgrenar sig i "Klustermall" respektive "Standardmall" -->

## Separat arbetsmall, inte en gemensam

Istället för att bygga om den befintliga arbetsmallen delade vi upp flödet i två separata arbetsmallar, åtskilda med en egen arbetsklassificering:

- En ny arbetsmall för klusterberättigade order, med lägre sekvensnummer (utvärderas först) och en egen arbetsklass.
- Den befintliga arbetsmallen förblir helt orörd och fångar upp allt annat.

Ställplatsen (put) för klusterordern styrs via en dirigeringskod på arbetsmallens ställ-rad, kopplad till en ny platsdirigering. En dirigeringskod gör att D365 söker platsdirigeringar efter kod istället för efter sekvensnummer — det är standardmekaniken, inte en workaround. Resultatet: klusterorder får en förutsägbar, systemresolverad ställplats, medan allt annat behåller dagens manuella val av packstation helt opåverkat.

## Fallgrop 1: en order med blandade artiklar

Den första versionen av urvalsfrågan (header query) filtrerade direkt på artikelnummer på arbetsmallens toppnivå. Det fungerade perfekt — så länge *alla* rader på en order var klusterberättigade. Men en order med en blandning av berättigade och icke-berättigade artiklar splittrades i **två separata arbeten**: den berättigade raden gick till klustermallen, resten föll igenom till standardmallen.

Orsaken: urvalsfrågan utvärderas per rad, inte per order. Ett enkelt filter på radens egen artikel kan aldrig fånga upp systerraderna på samma order.

<!-- Skärmdump-förslag: Edit query-dialogen på arbetsmallen, Joins-fliken, med kopplingsträdet synligt -->

Lösningen var att byta frågan mot en **Exists-join**: koppla en andra instans av orderraderna via ordertabellen, med artikelvillkoret på den kopplade raden — och med kopplingsläget satt till *Exists*, inte standardvalet *Inner Join*. Frågan blir då "finns det **någon** rad på ordern med en berättigad artikel", inte "är just den här radens artikel berättigad". Varje rad på en kvalificerande order matchar då mallen, oavsett vilken specifik rad som utlöste matchningen.

## Fallgrop 2: klustret startar inte om det inte är fullt

Klusterprofilen har en inställning, *Activate positions*, som är påslagen som standard. Med den påslagen kräver systemet att **alla** konfigurerade positioner har tillgängligt arbete innan ett kluster ens skapas — finns det färre kvalificerande order än antalet positioner får du felmeddelandet "Not enough work can be found for cluster", även om det finns gott om arbete för de positioner som faktiskt är tillgängliga.

Lösningen är att stänga av *Activate positions*. Det är en dokumenterad, känd begränsning — men lätt att missa om man bara läser den översiktliga beskrivningen av fältet.

## Slutsats

Inget av det här kräver anpassad kod — allt byggs med standardmekaniken för arbetsmallar, platsdirigeringar och klusterprofiler. Men klusterplockning som bara ska gälla en delmängd av artiklarna, utan att röra resten av flödet, kräver att man förstår exakt *var* i konfigurationen varje beslut faktiskt fattas — annars är det lätt att antingen störa den befintliga plockningen eller hamna i en av de här två fallgroparna.

Har du ett liknande behov i din WMS-miljö? Hör gärna av dig.
