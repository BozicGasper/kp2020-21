# kp2020-21
Projekt za seminarsko nalogo pri predmetu Komunikacijski Protokoli

## Shema omrežja

<img src="networkScheme.png">

Omrežje je razdeljeno na 3 segmente, z zunanjim internetom pa ga povezuje **vyos** router z javnim IPv4 naslovom **88.200.24.237** in IPv6 naslovom **2001:1470:fffd:98::2/64**

Med segmenti delimo:
- **javno** podomrežje ```subnet4 192.168.7.0/24``` ter ```subnet6 2001:1470:fffd:99::/64``` do katerega dostopamo preko priključka ```eth1```
- **interno** podomrežje ```subnet4 10.0.7.0/24``` ter ```subnet6 2001:1470:fffd:9a::/64``` do katerega dostopamo preko priključka ```eth2```
- **ipv6only** podomrežje ```subnet6 2001:1470:fffd:9b::/64``` do katerega dostopamo preko priključka ```eth3```

Priključki (interfaces):
- **eth0** je povezan z zunanjim internetom, in sicer na naslovu ```IPv4 88.200.24.237/24``` in ```IPv6 2001:1470:fffd:98::2/64```
- **eth1** je povezan s stikalom ```sk07-dmz``` in sicer na naslovu ```IPv4 192.168.7.1/24``` in ```IPv6 2001:1470:fffd:99::2/64```
- **eth2** je povezan s stikalom ```sk07-internal``` in sicer na naslovu ```10.0.7.1/24``` in ```IPv6 2001:1470:fffd:9a::2/64```
- **eth3** je povezan s stikalom ```sk07-ipv6only``` in sicer na naslovu ```IPv6 2001:1470:fffd:9b::2/64```

## Uporaba SSH protokola za zunanji dostop do naprav
Za lažji in manj okreten nadzor sta **vyos** usmerjevalnik in **javni Ubuntu strežnik** dostopna za upravljanje preko ssh protokola, kjer se za dostop preverja uporabniško ime in geslo uporabnika za ta sistem.

**dostopnosti:**
- do usmerjevalnika vyos lahko dostopamo z ukazom ```ssh vyos@88.200.24.237 -p 312```
- do strežnika Ubuntu, priklopljenega na *sk07-dmz* stikalo, lahko dostopamo z ukazom ```ssh gazic@88.200.24.237 -p 3001```

Seveda ne gre brez omembe, da je pri obeh potrebno vedeti tudi geslo za uporabnika, s katerim se želimo prijaviti

## NAT konfiguracija usmerjevalnika
### DNAT (Destination NAT)
#### SSH dostop
V prejšnem razdelku kofniguracije SSH protokola lahko opazimo, da se za vzpostavitev povezave pri nobeni izmed naprav ne uporabljajo standardna vrata za ssh protokol (**22**).
To smo omogočili s konfiguriranjem preslikovanja naslovov, kjer smo ves promet vrat **3001** pa smo usmerili na priključek ```eth1``` preko ```sk07-dmz``` stikala na naslov ```192.168.7.100:22```, na katerem je dostopen naš **javni Ubuntu strežnik**, ki posluša za promet na vratih 22 (nameščen je <a href="https://www.openssh.com/">**openssh**</a>).
```bash
rule 10 {
    description "Server ssh"
    destination {
        port 3001
    }
    inbound-interface eth0
    protocol tcp
    translation {
        address 192.168.7.100
        port 22
    }
}
```
#### HTTP dostop
Naš ubuntu strežnik zunanjemu svetu nudi **RESTful API**, in sicer na vratih 3000. Da lahko do le teh dejansko dostopamo iz zunanjega sveta, je na usmerjevalniku dodana konfiguracija za preusmeritev prometa iz iz vrat **3000** na priključek ```eth1``` na naslov ```192.168.7.100:3000```.
```bash
rule 420 {
    description "server forwarding"
    destination {
        port 3000
    }
    inbound-interface eth0
    protocol tcp
    translation {
        address 192.168.7.100
    }
}
```
Skonfigurirana je tudi preusmeritev za ves http promet na vratih 80, ki se prav tako preusmeri do strežnika, kjer je na istih vratih s pomočjo **Apache2** podpore servirana **Cacti** nadzorna stran za **SNMP monitoring**
```bash
rule 30 {
    description "80 for cacti"
    destination {
        port 80
    }
    inbound-interface eth0
    protocol tcp
    translation {
        address 192.168.7.100
    }
}
```
#### SNMP dostop
Na ubuntu sistemu, ki priključen na interno stikalo (```sk07-internal```) so nameščena osnovna <a href="http://www.net-snmp.org/docs/man/">**snmp**</a> orodja za izvajanje poizvedb, na ubuntu strežniku, ki je priklopljen na dmz stikalo (```sk07-dmz```) pa je poleg osnovnih orodij nameščen tudi **snmp agent** (<a href="https://linux.die.net/man/8/snmpd">**snmpd**</a>), ki odgovarja na poizvedbe (npr. ```snmpwalk```).
```bash
rule 20 {
    description "161 from internal to dmz"
    destination {
        port 161
    }
    inbound-interface eth2
    protocol udp
    translation {
        address 192.168.7.100
    }
}
```
Kot je možno razbrati, ta konfiguracija ne vpliva na prihod paketov iz zunanjega sveta, temveč le iz priključka ```eth2``` preko protokola udp (*161/udp*)
### SNAT (Source NAT)

//todo matic

### Konfiguracija Požarnih zidov

#### Vyos

//todo matic
 
#### Javni Ubuntu strežnik
Za Ubuntu strežnik, ki ga uporabljamo za serviranje **RESTApi-ja** ter statične spletne strani za monitoring (**cacti**), je konfiguracija potekala takole:

najprej smo skonfiguriali dostop za ssh
```bash
gazic@gazic:~$ sudo ufw allow ssh
```
kar nam je omogočilo promet skozi vrata **22** s protokolom **tcp** za IPv4 in IPv6 promet.
Nato je bilo potrebno dovoliti dostop do statične spletne strani, ki jo serviramo s pomočjo **Apache2** in sicer na vratih **80**
```bash
gazic@gazic:~$ sudo ufw allow 80/tcp
```
Tako smo omogočili komunikacijo preko protokola tcp skozi vrata 80 s katerega koli naslova.
nato je bilo potrebno odpreti še vrata za RESTApi, ki ga serviramo na vratih **3000**, prav tako preko **tcp** protokola.
```bash
gazic@gazic:~$ sudo ufw allow 3000/tcp
```
Spomnimo se, da za princip testiranja poskušamo izvajati vzorčne klice s *snmp* orodjem iz ubuntu sistema, ki je priključen v internem podomrežju, zato bomo vrata **161** preko protokola **udp** (snmp komunicira preko udp), odrpli le za ves promet, ki prihaja iz podomrežja ```10.0.7.0/24```
```bash
gazic@gazic:~$ sudo ufw allow from 10.0.7.0/24 to any port 161 proto udp
```
Po končani konfiguraciji lahko z ukazom ```sudo ufw status numbered``` prikažemo vsa nastavljena pravila za naš Ubuntu sistem
```bash
gazic@gazic:~$ sudo ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 3000/tcp                   ALLOW IN    Anywhere
[ 3] 80/tcp                     ALLOW IN    Anywhere
[ 4] 161/udp                    ALLOW IN    10.0.7.0/24
[ 5] 22/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 6] 3000/tcp (v6)              ALLOW IN    Anywhere (v6)
[ 7] 80/tcp (v6)                ALLOW IN    Anywhere (v6)
```

