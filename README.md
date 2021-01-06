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
### RESTful API storitev
Na Ubuntu strežniku, ki je dostopen javnosti, na vratih 3000 serviramo Stateless mikrostoritev **Sledilnik števila obiskovalcev**, ki služi kot pripomoček za sledenje števila oseb v zaprtih prostorih ter za kreiranje statistike in poročil o trendih zasedenosti. Ponuja vmesnik, s katerim lahko zaposleni ročno beležijo vstope in izstope, pri čemer lahko beleženje poteka na več vhodih hkrati. Določen uporabnik (pravna oseba) ima pod nadzorom enega ali več prostorov, vsak prostor pa ima enega ali več vhodov. Vsak prostor ima določeno tudi velikost.

Aplikacija med drugim omogoča tudi vnos omejitve števila obiskovalcev na kvadratni meter, na podlagi katere se izračuna dovoljeno število obiskovalcev v prostoru.
#### Nameščanje Aplikacije
**Javansko** aplikacijo bomo s pomočjo orodja **Docker** stregli v vsebniku, ki bo izpostavljen na vratih 3000, hkrati pa bomo morali v še enem dodatnem vsebniku poganjati **Postgresql** podatkovno bazo, ki je brezpogojna za celotno funckionalnost aplikacije.
##### Nameščanje okolja docker
Najprej zvedemo najbolj potreben ukaz za vse OCD razvijalce
```bash
gazic@gazic:~$ sudo apt update
```
Nato bomo dodali knjiznjice, ki bodo sistemu ```apt``` dovolile prenos novih knjiznjic preko HTTPS
```bash
gazic@gazic:~$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
Naslednji korak je ta, da dodamo GPG ključ, ki nam omogoča prenos datotek iz uradnega Docker repozitorija
```bash
gazic@gazic:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
Nato dodamo Docker repozitorij med repozitorije, ki so znani programu ```apt``` 
```bash
gazic@gazic:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```
ponovno zaženemo ocd ukaz (tokrat brezpogojno)
```bash
gazic@gazic:~$ sudo apt update
```
Nato lahko končno poženemo ukaz ta prenos in namestitev docker okolja
```bash
gazic@gazic:~$ sudo apt install docker-ce
```
Preverimo delovanje
```bash
gazic@gazic:~$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2021-01-06 12:01:47 CET; 3h 30min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1057 (dockerd)
      Tasks: 30
     Memory: 179.5M
     CGroup: /system.slice/docker.service
             ├─1057 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```
##### Namestitev dodatka Docker Compose
Da lahko našo aplikacijo zaženemo karseda lahko in brez dolgih spisov v ukazni vrstici, bomo namestili orodje **Docker Compose**, ki služi kot nekaksen dinamičen organizator docker vsebnikov.

Da bomo prenesli res najnovejšo verzijo orodja, bomo to storili preko njihovega uradnega <a href="https://github.com/docker/compose">Github repozitorija</a>.
```bash
gazic@gazic:~$ sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Slednji ukaz bo prenesel in shranil ```executable``` datoteko **docker-compose** v lokalno shrambo uporabnika ```/usr/local/bin/docker-compose```, kar bo omogočilo, da bomo lahko izvedli program kar z ukazom ```$ docker-compose```. Seveda je potrebno nastaviti tudi temu primerne pravice.
```bash
gazic@gazic:~$ sudo chmod +x /usr/local/bin/docker-compose
```
Namestitev preverimo z:
```bash
gazic@gazic:~$ docker-compose --version
docker-compose version 1.27.4, build 40524192
```
##### Postavitev aplikacije in strežba
Slika aplikacije se nahaja na DockerHub repozitoriju **mrzic/trendi**. Priraviti je potrebno ustrezeno docker-compose.yml datoteko, da bomo lahko efektivno pognali celotno aplikacijo.
```bash
cd ~
touch docker-compose.yml
nano docker-compose.yml
```
Prilepimo naslednjo vsebino in shranimo.
```yaml
version: "3"
services:
  database:
    container_name: postgres-trendi
    image: postgres:latest
    restart: always
    environment:
      POSTGRES_DB: trendi
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - database-data:/var/lib/postgresql/data/
    ports:
      - "5432:5432"
    networks:
      - omrezje
  obiskovalci:
    container_name: obiskovalci
    image: mrzic/trendi:latest
    restart: always
    ports:
      - "3000:8080"
    depends_on:
      - database
    links:
      - database
    networks:
      - omrezje
networks:
  omrezje:
volumes:
  database-data:
```
Nato poženemo docker-compose
```bash
gazic@gazic:~$ docker-compose up -d
Creating network "gazic_omrezje" with the default driver
Creating postgres-trendi ... done
Creating obiskovalci     ... done
gazic@gazic:~$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS                    NAMES
436b992f7c37   mrzic/trendi:latest   "java -jar api-1.0-S…"   7 seconds ago   Up 5 seconds   0.0.0.0:3000->8080/tcp   obiskovalci
5eb24d2d425c   postgres:latest       "docker-entrypoint.s…"   7 seconds ago   Up 6 seconds   0.0.0.0:5432->5432/tcp   postgres-trendi
```
Kot lahko vidimo je vsebnik, v katerem se streže RESTful API izpostavljen na vratih **3000** ( *vsebnik "obiskovalci"* ).

Sedaj lahko v brskalniku preverimo delovanje naše RESTful API točke tako, da poskušamo pridobiti **OpenApi** dokumentacijo na naslovu <a href="http://88.200.24.237:3000/api-specs/ui">http://88.200.24.237:3000/api-specs/ui</a>.

