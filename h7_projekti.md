Tarkoituksena oli automatisoida ntfy -palvelimen asennus käyttäen saltia. 
NTFY on yksinkertainen HTTP-pohjainen pub-sub (publish-subscribe) -ilmoituspalvelu.

## Asennus saltilla

Vuokrasin linodelta palvelimen ja tein api avaimen. Api avaimelle annoin kaikki oikeudet, koska en löytänyt sopivaa komboa, jolla saisin tämän muuten toimimaan. 

![kuva](images/h7/1.png)

![kuva](images/h7/2.png)

Otin ssh yhteyden linode palvelin koneelle. Asensin siihen salt-masterin ja salt-cloudin.

Tein `/etc/salt/cloud.providers` -hakemistoon `linode.conf` -tiedoston.
```
linode:
  provider: linode
  password: <linode_password
  apikey: <linode_api_key>
  api_version: v4
  driver: linode
```
`linode:`: Alustaa Linode-palveluntarjoajan konfiguraation.
`provider: linode`: Määrittelee palveluntarjoajan tyyppinä Linoden.
`password: <linode_password>`: Linode-tilin root salasana.
`apikey: <linode_api_key>`: Linode-tiliin liitetty API-avain.
`api_version: v4`: Määrittää Linoden API:n version 4.
`driver: linode`: Määrittelee Linoden Cloud-palvelimen käyttöön otettavan driverin (ajurin).

Seuraavaksi tein `/etc/salt/cloud.profiles` -hakemistoon `minion.conf` -tiedoston.

```
minion:
  provider: linode-provider
  size: g6-nanode-1
  image: linode/debian11
  location: se-sto
  minion:
    master: <master_ip> 
```

`minion:`: Määrittelee profiilin nimen, tässä tapauksessa "minion".
`provider: linode-provider`: Kertoo, että tämä profiili käyttää Linode-palveluntarjoajaa Salt Cloudissa.
`size: g6-nanode-1`: Määrittelee palvelimen kokoluokan, tässä käytetään "g6-nanode-1", eli pienintä mahdollista.
`image: linode/debian11`: Valitsee palvelimelle käytettävän kuvan, tässä Debian 11 Linode-kuvan.
`location: se-sto`: Määrittelee palvelimen sijainnin, tässä käytetään "se-sto".
`minion: master: <master_ip>`: Määrittää minionille masterin IP-osoitteen, johon se liittyy. 

Ajoin komennon `curl -fsSL https://archive.heckel.io/apt/pubkey.txt | sudo gpg --dearmor -o /srv/salt/ntfy/archive.heckel.io.gpg`,  
tämä komento lataa GPG-avaimen tiedoston verkosta, muuntaa sen binäärimuotoon ja tallentaa sen `/srv/salt/ntfy/archive.heckel.io.gpg` -tiedostoon.

`/srv/salt/ntfy` tein vielä tiedoston `archive.heckel.io.list` johon laitoin `deb [arch=amd64 signed-by=/etc/apt/keyrings/archive.heckel.io.gpg] https://archive.heckel.io/apt debian main`

Tein `/srv/salt/ntfy` -hakemistoon `init.sls` -tiedoston, joka asentaa tulevalle minion -koneelle ntfy -palvelimen.
```yaml
 /etc/apt/keyrings:
  file.directory:
    - makedirs: True

apt-transport-https:
  pkg.installed

/etc/apt/keyrings/archive.heckel.io.gpg:
  file.managed:
    - source: salt://archive.heckel.io.gpg

/etc/apt/sources.list.d/archive.heckel.io.list:
  file.managed:
    - source: salt://archive.heckel.io.list

install_ntfy:
  pkg.installed:
    - refresh: True
    - name: ntfy
```
 `/etc/apt/keyrings:`: Tarkistaa, että `/etc/apt/keyrings` -hakemisto on olemassa.
 
`file.directory:`: Salt State -moduuli, joka varmistaa hakemiston olemassaolon.

`apt-transport-https:`: Asentaa `apt-transport-https` -paketin.

`pkg.installed:`: Salt State -moduuli, joka varmistaa paketin asennuksen.
`/etc/apt/keyrings/archive.heckel.io.gpg:`: Tarkistaa, että `/etc/apt/keyrings/archive.heckel.io.gpg` -tiedosto on olemassa.

 `file.managed:`: Salt State -moduuli, joka varmistaa tiedoston olemassaolon ja hallinnan.

`/etc/apt/sources.list.d/archive.heckel.io.list:`: Tarkistaa, että `/etc/apt/sources.list.d/archive.heckel.io.list` -tiedosto on olemassa.

`install_ntfy:`: tilan id.
`pkg.installed:`: Salt State -moduuli, joka varmistaa paketin asennuksen.
`refresh: True`: Paketin asennuksen yhteydessä päivittää pakettilistat.
`name: ntfy`: Määrittelee asennettavan paketin nimen.

Tein `/srv/cloud` -hakemiston johon tein `ntfy` -tiedoston. 
```yaml
minion:
  - ntfy-1:
      location: se-sto
```
`minion:`: Määrittelee profiilin nimen, tässä tapauksessa "minion".
 `- ntfy-1:`: Määrittelee minionin nimen, tässä "ntfy-1". 
`location: se-sto`: Määrittää minionin sijainnin, tässä "se-sto" eli Tukholma, Ruotsi.

Tämä profiilitiedosto antaa Salt Cloudille ohjeet siitä, miten se luo ja asentaa minionin Linode-palvelimelle. Tässä määritellään yksi minion, nimeltään "ntfy-1", ja määritellään sen sijainti "se-sto".

Komennolla `sudo salt-cloud  -m /srv/cloud/ntfy` sain automaattisesti vuokrattua minion koneen.

Master -koneella näkyi automaattisesti minionin avain hyväksyttynä.

![kuva](images/h7/8.png)

Komennolla: `sudo salt 'ntfy*' state.apply ntfy` ajoin ntfy -palvelimen asennuksen vuokratulle minion -koneelle.

```bash
sudo salt '*' state.apply ntfy
ntfy-1:
----------
          ID: /etc/apt/keyrings
    Function: file.directory
      Result: True
     Comment: The directory /etc/apt/keyrings is in the correct state
     Started: 15:04:47.397052
    Duration: 4.645 ms
     Changes:
----------
          ID: apt-transport-https
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 15:04:48.152220
    Duration: 22.831 ms
     Changes:
----------
          ID: /etc/apt/keyrings/archive.heckel.io.gpg
    Function: file.managed
      Result: True
     Comment: File /etc/apt/keyrings/archive.heckel.io.gpg is in the correct state
     Started: 15:04:48.175365
    Duration: 26.874 ms
     Changes:
----------
          ID: /etc/apt/sources.list.d/archive.heckel.io.list
    Function: file.managed
      Result: True
     Comment: File /etc/apt/sources.list.d/archive.heckel.io.list is in the correct state
     Started: 15:04:48.202549
    Duration: 24.537 ms
     Changes:
----------
          ID: install_ntfy
    Function: pkg.installed
        Name: ntfy
      Result: True
     Comment: All specified packages are already installed
     Started: 15:04:48.227265
    Duration: 10.547 ms
     Changes:
----------
          ID: ntfyService
    Function: service.running
        Name: ntfy
      Result: True
     Comment: The service ntfy is already running
     Started: 15:04:48.238784
    Duration: 19.738 ms
     Changes:

Summary for ntfy-1
------------
Succeeded: 6
Failed:    0
------------
Total states run:     6
Total run time: 109.172 ms
```

## NTFY testaus cronjobilla

Kävin tekemässä ntfy:n sovelluksella palvelimen osoitteeseen ilmoitusten tilauksen. Testasin ajaa `curl -d "successful" http://172.234.120.86/ntfyTopic`,  joka lähetti puhelimeeni ilmoituksen. 


Tein ensin testimielessä virtuaalikoneella cronjobiin määrityksen, joka lähettää ilmoituksia joka 2. minuutti. Komennolla `crontab -e` avasin cronjobin asetustiedoston. Sinne laitoin 
```
*/2 * * * * curl -d "jeejee cronjob" http://172.234.120.86/ntfyTopic
```

Kävin vaihtamassa tämän, että cronjob päivittää virtuaalikoneen jokaisen tunnin viides minuutti ja lähettää curlilla ntfy -palvelimelle "updates successful", jos päivityksen onnistuvat:
`crontab -e`
```
5 * * * *  sudo apt update -y && sudo apt upgrade -y && curl -d "updates successful" http://172.234.120.86/ntfyTopic
```

![kuva](images/h7/11.png)

## Palomuuri saltilla

Lisäsin vielä saltilla palomuuriasetukset `/srv/salt/ntfy` -kansiossa sijaitsevaan `init.sls` tiedostoon. 
```yaml
/etc/apt/keyrings:
  file.directory:
    - makedirs: True

apt-transport-https:
  pkg.installed

/etc/apt/keyrings/archive.heckel.io.gpg:
  file.managed:
    - source: salt://ntfy/archive.heckel.io.gpg

/etc/apt/sources.list.d/archive.heckel.io.list:
  file.managed:
    - source: salt://ntfy/archive.heckel.io.list

install_ntfy:
  pkg.installed:
    - refresh: True
    - name: ntfy

ntfyService:
  service.running:
    - name: ntfy

ufw:
  pkg.installed:
    - pkgs:
      - ufw

ufw allow 22/tcp:
  cmd.run:
    - unless: "ufw status verbose | grep '22/tcp'"
ufw allow 80/tcp:
  cmd.run:
    - unless: "ufw status verbose | grep '80/tcp'"

ufw enable:
  cmd.run:
    - unless: "ufw status | grep 'Status: active'"
```

`ufw:`: Tämä määrittelee Salt State -moduulin, joka hallitsee `ufw`-paketin asennusta.

`pkg.installed:`: Salt State -moduuli, joka varmistaa, että `ufw` on asennettu.
`pkgs:`: Määrittelee paketit, jotka tulee asentaa. Tässä tapauksessa vain `ufw`.

`ufw allow 22/tcp:`: Tämä osa määrittelee Salt State -moduulin, joka ajaa komennon (`cmd.run`), joka sallii liikenteen portissa 22 (SSH-portti) palomuurissa, jos se ei ole jo sallittu.

`unless: "ufw status verbose | grep '22/tcp'"`: Määrittelee ehtolauseen, joka tarkistaa, onko portti 22 sallittu. Jos se ei ole sallittu, komento suoritetaan.

`ufw allow 80/tcp:`: Tämä osa määrittelee Salt State -moduulin, joka sallii liikenteen portissa 80 (HTTP-portti) palomuurissa, jos se ei ole jo sallittu.

`unless: "ufw status verbose | grep '80/tcp'"`: Määrittelee ehtolauseen, joka tarkistaa, onko portti 80 sallittu. Jos se ei ole sallittu, komento suoritetaan.

`ufw enable:`: Tämä osa määrittelee Salt State -moduulin, joka ajaa komennon (`cmd.run`), joka ottaa palomuurin käyttöön, jos se ei ole jo käytössä.

`unless: "ufw status | grep 'Status: active'"`: Määrittelee ehtolauseen, joka tarkistaa, onko palomuuri jo aktiivinen. Jos se ei ole aktiivinen, komento suoritetaan.

Ajoin `sudo salt 'ntfy*' state.apply ntfy`, joka palautti palomuuriasetusten onnistuneen.

![kuva](images/h7/10.png)

## Lähteet

https://zesc.wiki/articles/basics/secure-debian-salt/


https://help.dreamhost.com/hc/en-us/articles/215767047-Creating-a-custom-Cron-Job

https://docs.saltproject.io/en/latest/topics/cloud/qs.html#salt-cloud-qs

https://docs.saltproject.io/en/latest/topics/cloud/linode.html

https://docs.ntfy.sh/install/

https://docs.ntfy.sh/config/
