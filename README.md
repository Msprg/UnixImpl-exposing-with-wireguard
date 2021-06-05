# UnixImpl-exposing-with-wireguard
Semestrálna práca pre Implementácie Linuxu / UNIXu, tunelovanie služieb z LAN cez NAT na internet.


# Koncept a cieľ(e)
Pred tým, než sa internet rozšíril do sveta tak ako je tomu teraz, sa s takouto veľkou popularitou pri nárhu niektorých dôležitých protokolov pre fungovanie internetu, nepočítalo. Avšak postupne sa ukázalo že je adresný priestor IPv4 nedostačujúci. Riešenie pre tento problém v skutočnosti existuje už celkom dlho - IPv6, avšak jeho adpocia je zatiaľ naozaj veľmi chabá / sporadická, aj keď sa to postupne pomaly zlepšuje.

Veľa ISP ešte stále poskytuje pripojenie do internetu iba pomocou IPv4, namiesto toho, aby implementovali IPv6, ktorého adresný priestor sa nateraz označuje ako nevyčerpateľný, aj keď podobne to bolo aj pri začiatkoch implementácie IPv4, a predsa sa nám už prakticky minuli.

NAT - Network Address Translation, konkrétne PAT (Port-A-T) alebo PNAT (Port-N-A-T), je skôr "hackom" než reálnym riešením nedostatku IPv4 adries, avšak funguje dosť dobre (a je jednoduchší na implementáciu) na to, aby ho ISP používali radšej, než prešli na IPv6, alebo dual-stack IPv4 + IPv6.

Jednoducho povedané, NAT dokáže "skryť" viacero IP adries, (obvykle v privátnom rozsahnu, aj keď nie nutne), za jednu "verejnú" (opäť, nie nutne), IP adresu. Konkrétne PNAT, to robí použitím portov pre jednotlivé IP adresy, respektíve ich spojenia. To sa nazýva "Overload"-ing.

"Wi-Fi routre" alebo ako sa týmto zariadeniam nadáva, sú vlastne "all-in-one" sieťové jednotky, kde je v jednom zariadení minimálne router, switch, WAP, firewall, managment, a často krát napríklad aj modem. Tieto zariadenia sú v predvolenej konfigurácií už nastavené na používanie NAT-u, pričom v mnohých sa NAT-ovanie ani nedá vypnúť!
Spomínam to preto, lebo takto je prakticky garantované, že už takto ste za jedným NAT-om, a z pohľadu ISP máte pripojené iba jedno zariedenie, s jednou IP adresou! Pokiaľ vám potom ISP takto pridelí rovno adresu z verejného rozsahu, t.j. Verejnú IP adresu, tak je medzi vami a doslova internetom iba jeden NAT, ku ktorého konfigurácií máte prístup. Takže môžete pristupovať z internetu cez svoju verejnú (verejne prístupnú IP), ku zariadeniam v lokálnej sieti. Stačí permanentne otvoriť niektorý port. Tomu sa hovorí port forwarding...

Takže to je pre ISP jedna verejná IPv4 adresa na jedného zákazníka. Lenže dokonca ešte aj na to je ich málo, a tak sa niektorí ISP rozhodli pre použitie ďaľšieho NATu. Toto sa často nazýva ako ISP-Grade NAT, alebo jednoducho double-NAT. Znamená to, že každý zákazník nedostane verejnú IPv4 adresu, ale jednu z adries vo WAN v sieti ISP. ISP potom túto celú WAN sieť opäť NAT-uje, na jednu alebo viac verejných IPv4 adries.

Teraz si už ale môžete port-forwardovať na svojom NAT-e u seba doma koľko chcete, ale otvoríte tým prístup maximálne tak cez WAN u ISP. Čo ak teraz chcete pristupovať do svojej lokálnej siete? Existuje na to niekoľko spôsobov, najlepšie je obvykle skúsiť od ISP získať vlastnú verejnú IPv4 adresu, čo ale môže priniesť aj nejaké nevýhody, tomu sa ja už ale venovať nebudem.



# Prístup do siete okľukou
## Konkrétne použitím externého servera

Ja budem obchádzať toto obmedzenie double-NATu pomocou nejakého ďalšieho servera, ktorý má verejnú IPv4 adresu. Konkrétne pomocou VPS - Virutal Private Server. Je to virtualizovaný server, principiálne identický s behom linuxovej distribúcie (alebo aj iného OS, na tom prakticky nezáleží) vo VirtualBox, alebo VMware, s tým, že na svojej virtuálnej sieťovej karte bude mať pridelenú verejnú IPv4 adresu!

## Môj VPS

Konkrétne môj VPS bude bežať na distribúcií CentOS 7, ktorá je síce už dosť stará, ale stále dostáva nejaké tie aktualizácie, a na tento účel bez problémov poslúži.

Okrem toho budeme potrebovať ešte nejaké zariadenie u nás, v LAN do ktorej budeme chcieť pristupovať, respektíve v ktorej sa nachádzajú zariadenia ktoré budeme chcieť "vystrčiť" na internet (expose), ktoré poslúži ako server / node.

Teoreticky sa to dá spraviť aj tak, že by toto robil každý server v LAN samostatne, tak ako je mu to potrebné, ale to by výrazne skomplikovalo konfiguráciu - ako počiatočné nasadenie, tak aj rekonfiguráciu. Namiesto toho, tak budem mať v LAN iba Raspebrry PI 4, ktoré bude zastrešovať prístup a komunikáciu medzi zariadeniami LAN a VPS.

## Základný koncept fungovania

Kvôli double-NATu, musíme akékoľvek spojenie začať z LAN do internetu, naopak to jednoducho nebude fungovať - NAT u ISP nám to nedovolí. Toto spojenie taktiež budeme musieť nejako udržať otvorené (keep alive), inak nám ho NAT po chvíli neaktivity jednoducho uzavrie, čo by spôsovilo pád všetkých, namä TCP, spojení.
Nakoniec, musíme nejako identifikovať, ku ktorému zariadeniu v LAN chceme z internetu pristupovať, keďže vlastne budeme mať iba jednu verejnú IPv4 adresu (tú ktorou disponuje VPS), ale potenciálne viacero zariadení v LAN. 
