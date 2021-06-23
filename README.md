# UnixImpl-exposing-with-wireguard
Semestrálna práca pre Implementácie Linuxu / UNIXu, tunelovanie služieb z LAN cez NAT na internet.

Autor: Matúš Prančík

# Obsah - TOC

- [UnixImpl-exposing-with-wireguard](#uniximpl-exposing-with-wireguard)
- [Obsah - TOC](#obsah---toc)
- [Koncept a cieľ(e)](#koncept-a-cieľe)
- [Prístup do siete okľukou](#prístup-do-siete-okľukou)
-   [Môj VPS](#môj-vps)
-   [Základný koncept fungovania](#základný-koncept-fungovania)
- [Samotná konfigurácia v praxi](#samotná-konfigurácia-v-praxi)
-   [Konfigurácia servera / VPS](#konfigurácia-servera--vps)
-   [Príprava na forwardovanie a routovanie paketov](#príprava-na-forwardovanie-a-routovanie-paketov)
-   [Inštalácia WireGuard](#inštalácia-wireguard)
-   [Konfigurácia WireGuard na VPS](#konfigurácia-wireguard-na-vps)
-   [Vytváranie konfigurácií pre klientov (nodes)](#prvé-spustenie-systému-a-inštalácia-aktualizácií)
-   [Test kofigurácie](#na-tomto-mieste-je-dobrý-nápad-konfiguráciu-wireguard-otestovať)
- [NAT, IPTABLES, ROUTING, FORWARDING, ...](#nat-iptables-routing-forwarding-)
-   [iptables na CentOS](#iptables-na-centos)
-   [iptables na Raspberry Pi](#iptables-na-raspberry-pi)
-   [Test kofigurácie](#test-forwardingu-a-nat-ovania-na-vps)
- [Konfigurácia iptables pravidiel pre NAT-ovanie na serveri (a otváranie portov na internet)](#Konfigurácia-iptables-pravidiel-pre-nat-ovanie-na-serveri-a-otváranie-portov-na-internet)
- [Záver](#záver)


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

![Exposing local services through WireGuard](https://user-images.githubusercontent.com/18015488/120908703-3520a980-c66d-11eb-8ed8-f3b5e0e812db.png)

Kvôli double-NATu, musíme akékoľvek spojenie začať z LAN do internetu, naopak to jednoducho nebude fungovať - NAT u ISP nám to nedovolí. Toto spojenie taktiež budeme musieť nejako udržať otvorené (keep alive), inak nám ho NAT po chvíli neaktivity jednoducho uzavrie, čo by spôsovilo reset všetkých, namä TCP, spojení.

Na spojenie medzi RPi4 v LAN a VPS sa celkom dobre hodí VPN, jednak z hľadiska bezpečnosti, ale aj kvôli flexibilite. OpenVPN je v tomto ohľade asi najznámejšie riešenie, avšak nie je úplne ideálne. Jednak nie je práve najjednoduchšie čo sa konfigurácie týka, a taktiež má históriu, nie naozaj ideálnej kompatibility, čo sa rôznych platoform týka, či už je to Microsoft Windows, MacOS, alebo dokonca aj Android.

Nedávno sa začala ukazovať potenciálna náhrada OpenVPN: WireGuard (wg) sľubuje prakticky všetko lepšie než OpenVPN - jednoduchšia konfigurácia, vyší výkon / nižšia latencia, lepšie zabezpečenie (dokonca aj post-kvantum), lepšia kompatibilita, ... No, uvidí sa ;)

Nakoniec, musíme nejako identifikovať, ku ktorému zariadeniu v LAN chceme z internetu pristupovať, keďže vlastne budeme mať iba jednu verejnú IPv4 adresu (tú ktorou disponuje VPS), ale potenciálne viacero zariadení v LAN.
Toto sa dá riešiť taktiež viacerými spôsobmi, napríklad cez reverznú proxy, ale ja to budem riešiť, paradoxne, opäť pridaním ďalšieho NATu, konkrétne PNATu (resp. PATu) takže budem mapovať interné zariadenia / adresy v podstate na porty.


# Samotná konfigurácia v praxi
## Konfigurácia servera / VPS

Ako už bolo spomenuté, na serveri v mojom prípade beží CentOS 7.

Nebudem tu demonštrovať nič z úvodnej konfigurácie, ako napríklad aktualizácie systému či bezpečné nastavenie SSH servera.

### Príprava na forwardovanie a routovanie paketov

Na koniec súboru, nachádzajúceho sa v prípade CentOS 7 v `/etc/sysctl.d/99-sysctl.conf`, pridáme `net.ipv4.ip_forward = 1` a `net.ipv6.conf.all.forwarding = 1`, čo zapne packet forwarding.

**Tento súbor môže byť podľa distribúcie na inom mieste. Dajte si na to pozor!**

Ekvivalentnú konfiguráciu treba urobiť aj na Raspberry Pi.



### Inštalácia WireGuard


Ako je uvedené na [oficiálnych stránkach inštalácie WireGuard](https://www.wireguard.com/install/#centos-7-module-plus-module-kmod-module-dkms-tools), pre CentOS 7 tu sú 3 metódy inštalácie. Dôvodom je, že CentOS 7 používa staré jadro, ktoré WireGuard natívne nepodporuje. Použijeme metódu 1 ktorá nám pre toto nainštaluje kernel CentOS kernel-plus.


```
  # Method 1: a signed module is available as built-in to CentOS's kernel-plus:
  
  sudo yum install yum-utils epel-release
  sudo yum-config-manager --setopt=centosplus.includepkgs=kernel-plus --enablerepo=centosplus --save
  sudo sed -e 's/^DEFAULTKERNEL=kernel$/DEFAULTKERNEL=kernel-plus/' -i /etc/sysconfig/kernel
  sudo yum install kernel-plus wireguard-tools
  sudo reboot
```

Po reštarte by sme už mali bežať na novom jadre, vrátane automaticky aktívnej služby WireGuard, takže aj ten už bude po reštarte spustený.


### Konfigurácia WireGuard na VPS


WireGuard "nepodporuje" štandardný model klient-server. Namiesto toho, existuje iba konfigurácia použitého WireGuard rozhrania, a uzly (nodes). Napriek tomu, pokiaľ by som použil výraz "WireGuard server", myslím tým CentOS 7 VPS.


Veľmi užitočná aj dobre napísaná dokumentácia, je, možno prekvapivo, na stránkach dokumentácie pre projekt Pi-Hole, čo je vlastne projekt na blokovanie reklám na úrovni DNS, navrhnutý pre beh na Raspberry Pi, ale to teraz nie je podstatné. V dokumentácií Pi-Hole je totižto zdokumentovaného oveľa viac než len Pi-Hole. [Dokumenácia pre nastavenie WireGuard na stránkach Pi-Hole](https://docs.pi-hole.net/guides/vpn/wireguard/overview/)


Určitú časť tejto dokumentácie, ktorú som sám používal, budem parafrázovať, avšak niektoré časti, pre ktoré to bude vhodné, určite pozmením.



1. Ako úplne prvé, vojdeme do interaktívnej relácie root shellu `sudo -i`, to aby sme nemuseli teraz všade pred každý príkaz pridávať sudo. (teda, pokiaľ je to potrebné, ja som bol prihlásený ako root).
2. Vojdeme do konfiguračného adresára WireGurad `cd /etc/wireguard`.
3. Budeme vytvárať veľa súborov, preto použijeme `umask 077`, nech potom neskôr nemáme problémy s oprávneniami.
4. Vygenerujeme pár kryptografických kľúčov `wg genkey | tee server.key | wg pubkey > server.pub`. Každý jeden WireGuard node bude potrebovať takýto kľúčový pár.
5. Vytvoríme si konfiguračný textový súbor wg0.conf `nano /etc/wireguard/wg0.conf`. Kde podľa názvov týchto súborov bude WireGurat pomenovávať vytvárané interfaces.
6. Na vrch konfiguračného súboru, vložíme definícu interface:
```
[Interface]
Address = 10.100.0.1/24, fd08:4711::1/64  #Povolené adresné rozsahy v tomto wg0 interface, a zároveň aj adresa tohoto node
ListenPort = 47111                        #Port cez ktorý sa bude dať ku tomuto interface pripojiť s ostatnými nodes
MTU = 1380                                #WireGuard defaultne používa MTU 1412, kým drvivá väčšina intefaces používa "štandarne" 1500 MTU. Nižšia hodnota MTU vo WireGuard tuneli často spôsobuje nadbytočnú fragmentáciu paketov, čo znižuje výkonnosť. Toto by malo pomôcť, ale mojim problémom s výkonom to veľmi nepomohlo...
```
7. Teraz do konfiguračného súboru vložíme súkromný kľúč. Šikovne to môžeme urobiť pomocou `echo "PrivateKey = $(cat server.key)" >> /etc/wireguard/wg0.conf`.
8. Konfiguráciu WireGuard interface na serveri máme týmto hotovú. Môže (ale nemusí) vyzerať nejako takto:

```
[Interface]
Address = 10.100.0.1/24, fd08:4711::1/64 #Address of the VPS inside Wireguard
ListenPort = 47111
MTU = 1380
PrivateKey = UOREQ/yam+jnDSns8sdyfmoDs5sds8sd5DS85s1pO7M=
```

Ale teraz ešte musíme vytvoriť konfiguráciu pre ostatné zariadenia, ktoré sa budú ku serveru pripájať. Minimálne teda pre RPi 4.


### Vytváranie konfigurácií pre klientov (nodes)

**Tu pokračujeme ešte stále na serveri, v adresári `/etc/wireguard/`!**


0. Ak je to potrebné, vráťte sa do adresára `/etc/wiregurad/` v privilegovanom shelli:
```
sudo -i
cd /etc/wireguard
umask 077
```

1. Pre jednoduhosť príkazov, a možnosť ich opakovaného použitia pre viacero klientov, si vytvoríme premennú pre meno klienta: `name="client_name"`.
2. Vygenerujeme kryptografické kľúče:
  2.1 Asymetrický (Private-Public pair): `wg genkey | tee "${name}.key" | wg pubkey > "${name}.pub"`,
  2.2 Symetrický (pre-shared key): `wg genpsk > "${name}.psk"`
  
3. Pridáme záznam o peerovi (klientovi, node) do konfiguračného súboru:
```
echo "[Peer]" >> /etc/wireguard/wg0.conf
echo "PublicKey = $(cat "${name}.pub")" >> /etc/wireguard/wg0.conf
echo "PresharedKey = $(cat "${name}.psk")" >> /etc/wireguard/wg0.conf
```

4. Pridáme mu rozsah poveloených adries, `echo "AllowedIPs = 10.100.0.2/32, fd08:4711::2/128" >> /etc/wireguard/wg0.conf` kde:
  IPv4 `10.100.0.2/32` je adresa konkrétneho peera, ideálne z rozsahu definovanom v časti `[Interface]` na riadku
  `Address = ...`.
  IPv6 `fd08:4711::2/128` je IPv6 ekvivalent predchádzajúcej IPv4 adresy.
  IPv4 `192.168.0.0/24` je adresný rozsah, našej LAN, do ktorej budeme pristupovať. Pre jednoduchosť sa adresy sietí
  zhodujú, ale aj tak ju budeme NAT-ovať, takže by to mohla byť akákoľvek iná adresa, ideálne z privátnych rozsahov...

Konfiguračný súbor teraz môže (ale nemusí) vyzerať nejako takto:

```
[Interface]
Address = 10.100.0.1/24, fd08:4711::1/64 #Address of the VPS inside Wireguard
ListenPort = 47111
MTU = 1380
PrivateKey = UOREQ/yam+jnDSns8sdyfmoDs5sds8sd5DS85s1pO7M=

[Peer]
PublicKey = JsJHf5sd7S7SDPhwjuPfb5ss730H1LQtrslzw57gie5=
PresharedKey = jFs5sZ6584VS0SR5W9n86jbvR+bhOIdnsPs58aI4sIy=
AllowedIPs = 10.100.0.2/32, fd08:4711::2/128, 192.168.0.0/24
```


5. Použitím `systemctl restart wg-quick@wg0` môžeme slúžbu WireGuard reštartovať, ktorá si načíta novú konfiguráciu s novými peermi. Pokiaľ nenastane žiadna chyba, a po spustení `wg` bude vo výpise vidno všetkých peerov so správnymi parametrami, môžeme pokračovať ďalej.


Teraz ešte budeme potrebovať konfiguračný súbor ktorý dáme danému node / peerovi.

6. Vytvoríme časť `[Interface]`:
```
echo "[Interface]" > "${name}.conf"
echo "Address = 10.100.0.2/32, fd08:4711::2/128" >> "${name}.conf" # May need editing
# (Optional)
echo "MTU = 1360" >> "${name}.conf"                                # Add same MTU as is in the other peers config's
```

7. Pridáme do konfigurácie vygenerovaný kľúč pre tohto peera: `echo "PrivateKey = $(cat "${name}.key")" >> "${name}.conf"`

8. Tu sa nám to už začína prekrývať s NAT-om. Pre NAT-ovanie medzi lokálnou sieťou v ktorej je Raspberry Pi, a sieťou 192.168.0.0/24 ktorá bude iba medzi WireGurad peermi, v tuneli, do konfigurácie pridáme:
```
echo "#Forwarding (NAT)" >> "${name}.conf"
echo "PostUp = iptables -w -t nat -A POSTROUTING -o bond0 -j MASQUERADE; ip6tables -w -t nat -A POSTROUTING -o bond0 -j MASQUERADE" >> "${name}.conf"
echo "PostDown = iptables -w -t nat -D POSTROUTING -o bond0 -j MASQUERADE; ip6tables -w -t nat -D POSTROUTING -o bond0 -j MASQUERADE" >> "${name}.conf"
```
  Kde:
  `-o bond0` vo všetkkých štyroch príkazoch, je `bond0` rozhranie ktoré je priamo pripojené do lokálnej siete. Obvykle je to skôr `eth0`, `wlan0`, alebo niečo úplne iné, pokiaľ sú zapnuté "deskriptívne popisy" sieťových interfaces.

9. Pre druhé zariadenia, pridáme tento server (VPS) ako peera, aby vedeli kam sa vlastne majú pripojiť:
```
[Peer]
AllowedIPs = 10.100.0.0/24, fd08::/64       #IPs that are allowed to access the Pi, in this case everyone in Wireguard
Endpoint = [your public IP or domain]:47111 #IP/domain where the WireGuard "server" runs.
PersistentKeepalive = 25
```

10. Ako posledné, pridáme na koniec súboru, teda do časti `[Peer]`, kľúče potrebné pre pripojenie na server:
```
echo "PublicKey = $(cat server.pub)" >> "${name}.conf"
echo "PresharedKey = $(cat "${name}.psk")" >> "${name}.conf"
```

Týmto máme konfiguračný súbor pre klienta (peera) pripravený. Môže (ale nemusí) vyzerať nejako takto:
```
[Interface]
Address = 10.100.0.2/32, fd08:4711::2/128 #Address of the Pi inside Wireguard
PrivateKey = 8JSdp82DFKskSoP79K14SUS6Xzbc06MQeWertyzuioP=
MTU = 1360

#Forwarding (NAT)
PostUp = iptables -w -t nat -A POSTROUTING -o bond0 -j MASQUERADE; ip6tables -w -t nat -A POSTROUTING -o bond0 -j MASQUERADE
PostDown = iptables -w -t nat -D POSTROUTING -o bond0 -j MASQUERADE; ip6tables -w -t nat -D POSTROUTING -o bond0 -j MASQUERADE

#VPS
[Peer]
AllowedIPs = 10.100.0.0/24, fd08::/64 #IPs that are allowed to access the Pi, in this case everyone in Wireguard
Endpoint = 80.211.207.110:47111 #IP of the VPS
PersistentKeepalive = 25
PublicKey = p3s1/O+Ifra7poIUpMLopFX5/+IQnE87kdsOns1gJUD=
PresharedKey = KUZFžťRU76ztS57DoíáhS74WE+yťčwE6týváZtýXE0F=
```

Nakoniec tento konfiguračný súbor budeme nejako musieť dostať na zariadenie peera, pre ktoré bolo vytvorené. Kým `scp` by bola jedna možnosť, nie je problém ani skopírovať text a vložiť ho do prázdenho súboru na node.


### Na tomto mieste je dobrý nápad konfiguráciu WireGuard otestovať

Keď sú konfiguračné súbory na svojich miestach, pomocou `systemctl start wg-quick@wg0` (kde `@wg0` je `wg0` názov konfiguračného súboru, v tomto prípade `wg0.conf`, ktorý bude aj názvom WireGuard interface).

Potom, spustením `wg`, by sme mali vidieť či sa spojenie podarilo nadviazať. Napríklad pre výstup:
```
interface: wg0
  public key: Kzh3/O+bQRXUTZx6WZlI5ajv/+RSytlo43g4gOLqZgQ=
  private key: (hidden)
  listening port: 47111

peer: FKeRl37dsHbFgpemRiXoXBGoFaTpkmq0JMi34w7TR1k=
  preshared key: (hidden)
  endpoint: 88.212.41.116:46576
  allowed ips: 10.100.0.2/32, fd08:4711::2/128, 192.168.0.0/24
  latest handshake: 43 seconds ago
  transfer: 149.32 MiB received, 13.30 MiB sent

```

Môžeme vidieť, že interface `wg0` počúva na porte 47111, a `peer` má `latest handshake: 43 seconds ago`, čo znamená že spojenie cez tunel funguje. V tomto prípade môžeme skúsiť ešte `ping 10.100.0.2` na otestovanie konektivity peera, cez tunel.


# NAT, IPTABLES, ROUTING, FORWARDING, ...

## iptables na CentOS

Teraz nám ešte bude treba čosi pre NATovanie paketov. CentOS 7 natívne používa `Firewalld`, ale ja, byť tvrdohlavý aký som, chcem napriek tomu použiť `iptables`.
"Prepnúť" na iptables, je prekavpivo bezbolestné:
```
  # Vypneme službu firewalld, nech sa nezapína pri boote:
  systemctl disable firewalld

  # Nainštalujeme si cez yum iptables ako akýkoľvek iný balíček:
  yum install -y iptables-services
  
  # Aktivujeme službu iptables, nech sa nám spúšťa pri boote:
  systemctl enable iptables
```

A to je v podstate všetko, čo sa inštalácie iptables týka. Ak by sme službu iptables chceli spustiť hneď teraz, bez reštartu, `systemctl start iptables`.

Neskôr si ale určite budeme chcieť konfiguráciu iptables uložiť pomocou `service iptables save`, inak sa nám po reštarte zmaže.


## iptables na Raspberry Pi

Vzhľadom na to, že na mojom RPi beží Raspbian (Raspberry Pi OS, ale ja ho volám starým menom) z debianovej vetvy, iptables netreba inštalovať.

Musíme však zabezpečiť, aby, keď systém bude NAT-ovať a forwardovať pakety medzi interfaces wg0 (WireGuard tunel) a bond0 (interface priamo pripojené v LAN), iptables neblokovali tento forwarding a komunikáciu medzi týmito intefaces.

To sa dá jednoducho pomocou:
```
iptables -I FORWARD -i wg0 -o bond0 -j ACCEPT
iptables -I FORWARD -i bond0 -o wg0 -j ACCEPT
```
Ideálne je si po otestovaní tieto zmeny aj hneď uložiť. Pre Raspbian by mal fungovať príkaz `service netfilter-persistent save`.

Každopádne, to by na Raspberry Pi malo byť zatiaľ jediné z iptables čo treba nakonfiguraovať.


## Test forwardingu a NAT-ovania (na VPS)

Teraz by sa opäť oplatilo otestovať konfiguráciu.

Ak si opäť pozrieme výstup príkazu `wg`, zistíme, že v povolených IP pre peera, `allowed ips: 10.100.0.2/32, fd08:4711::2/128, 192.168.0.0/24` je okrem jeho adries, aj adresný rozsah lokálnej siete: `192.168.0.0/24`.

Ak je toto nastavené správne, WireGurad by mal pri spojení automagicky nastaviť NAT aj routing tak, že by už teraz mala fungovať konektivita medzi zariadeniami v LAN kde je Raspberry Pi, z VPS. Napríklad `ping 192.168.0.1` by mal postupne cez tunel, RPi, LAN, a spať, úspešne dostať ICMP odpoved, v tomto prípade od môjho edge routera.
```
# ping 192.168.0.1
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=63 time=33.0 ms
64 bytes from 192.168.0.1: icmp_seq=2 ttl=63 time=33.1 ms
^C
--- 192.168.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 33.077/33.104/33.131/0.027 ms
```

## Konfigurácia iptables pravidiel pre NAT-ovanie na serveri (a otváranie portov na internet)

Keď už máme správne nakonfigurovaný WireGuard, kde z VPS môžeme pristupovať ku zariadeniam v LAN, pokiaľ by sme urobili  obdobný konfiguračný súbor pre daľšie zariadenia, ktoré by sa dali pripojiť ku WireGuard na VPS, ako napríklad telefón, z každého takéhoto zariadenia by sme po pripojení získali ku adresnému rozsahu 192.168.0.0/24 rovnaký prístup, akoby sme boli pripojení ku LAN. Pre osobné použitie sa to určite zíde, ale pokiaľ by sme chceli neajkú službu "otvorene" poskytovať na Internete, tak okrem toho, že nemôžeme každého nútiť nainštalovať a nakonfigurovať WireGuard, tak navyše aj otvoriť prístup do celej LAN na internete by bolo veľmi hlúpe.

Našťastie stačí iba pár iptables pravidiel, ktoré otvoria na internet iba konkrétny port, konkrétneho zariadenia.


Napríklad, ak by sme chceli forwardovať pakety protokolu `TCP`, prichádzajúce na interface `eth0` na port `5111`, na IP `192.168.0.200`, na port `5001`, použijeme nasledujúci pár príkazov:
```
iptables --table nat --append PREROUTING --in-interface eth0 --destination 80.211.207.110 --protocol tcp --dport 5111 --jump DNAT --to-destination 192.168.0.200:5001

iptables --table nat --append POSTROUTING --protocol tcp --destination 192.168.0.200 --dport 5001 --jump SNAT --to-source 10.100.0.1
```
Tieto dva príkazy, pomocou NAT-u presmerujú všetky pakety prichádzajúce na interface eth0 port 5111, na IP 192.168.0.200 na port 5001. Iptables si taktiež zapamätajú, že tento paket bol forwardovaný a NAT-ovaný, takže keď server pošle odpoveď, iptables opäť zmenia port aj IP odpovede "naspäť", tak, aby opäť prišiel k pôvodnému odosielateľovi.

Ako je spomenuté vyššie, konfigurácia iptables sa dá na CentOS 7 uložiť pomocou `service iptables save`, takže keď si urobíme pravidlá, a otvoríme porty ktoré potrebujeme, uložíme si ich, pretoŽe inak sa pri reštarte nezachovajú.

Teraz sme už ale vlastne dosiahli cieľ - pomocou dvoch iptables príkazov, môžeme ku službám bežiacim na zariadeniach v LAN pristpovať cez VPS server podobne, ako cez lokálnu sieť.

# Záver

Začali sme prakticky s dvoma čistými linuxovými distribúciami, na ktorých sme postupne nakonfigurovali VPN tunel, NAT-ovanie a na koniec aj forwardovanie paketov z Internetu, až do zariadenia v LAN. Toto funguje celkom dobre, minimálne pre Webservery respektíve možno aj nejaké zložitejšie web stránky.










