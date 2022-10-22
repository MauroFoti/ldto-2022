# Introduzione

## Cos'è Nextcloud?

Nextcloud è un server cloud open source, praticamente un'alternativa gratuita e
free a servizi proprietari come Google Drive o OneDrive.

In una sola applicazione, permette di sincronizzare file, calendari e rubriche
di contatti. Inoltre è facilmente personalizzabile e ha uno store con molte
applicazioni per avere funzionalità aggiuntive, come la gestione di progetti
con deck e la creazione di form e questionari.

è disponibile sia come servizio enterprise, con un canone annuale, che self-
hosted, completamente gratuito. Noi sceglieremo questa soluzione ovviamente.

## Perchè un cloud self-hosted?

1. Totale controllo dei propri dati. Possiamo immagazzinare file privati e/o
confidenziali senza il rischio che vengano usati per scopi a cui non
acconsentiamo. Nessuno potrà vedere i nostri contatti e impegni riservati, come
le visite dal medico

2. Convenienza economica. Con poche centinaia di euro si può avere un cloud
affidabile con diversi TB di spazio. Per confronto, 2TB di spazio su google
drive possono costare fino a 50€/mese

3. Maggiore sicurezza. Se mantenuto costantemente aggiornato, un software
open-source con uno sviluppo così attivo diventa quasi impenetrabile. Nextcloud
offre anche uno scanner che permette di rilevare problemi e vulnerabilità note
della tua installazione e indica come correggerle.
Inoltre, una piattaforma centralizzata, se vulnerabile, può esporre i dati di
decine di milioni di utenti in una volta sola, mentre un'installazione
casalinga di nextcloud ne esporrà una decina al massimo. Per un attaccante che
vuole fare più danni possibile, è molto più conveniente profare a infiltrarsi
nelle grande piattaforme.

4. è più divertente e si impara un sacco di cose. Inoltre si può personalizzare
il sistema come meglio si crede

## Prerequisiti

1. Avere un PC come server, o un computer usato, o una VPS fornita da provider
di hosting
2. Avere un dominio registrato, esistono domini gratuiti, ma non sono molto
affdabili. Un dominio .it, .eu o .com costa pochi €/anno
3. Se teniamo il server in casa, serve un indirizzo ip statico, generalmente
fornito dal proprio Internet Provider. In laternativa, è possibile usare dei
DNS dinamici

### Questa macchina

La macchina di test è una Virtual Machine con Linux Mint 20.3 cinnamon (don't
judge pls). Tutte le istruzioni che seguono sono adatte a una macchina ubuntu-
based. I concetti sono validi per ogni sistema linux, ma occorre adattare di
conseguenza i comandi e i valori di impostazione.

---------------------------------------

# Installazione dei software

## ssh

Un server ssh permette di controllare la macchina da remoto

```sh
sudo apt-get install openssh-server
```

Il server è accessibile alla porta 22. Ci si può autenticare con password o,
meglio ancora, tramite chiave pubblica-privata.

Maggiori informazioni a riguardo al seguente (link)[https://www.ssh.com/academy/ssh/public-key-authentication]

Per caricare la propria chiave pubblica, copiare il file `~/.ssh/is_rsa.pub`
dal computer client in `~/.ssh/authorized_keys`. Questo file può avere un
numero illimitato di chiavi pubbliche, una per linea.

Per rendere la configurazione più sicura, occorre configurare i seguenti
parametri nel file di configurazione `/etc/ssh/sshd_config`

```sh
PasswordAuthentication no
PermitRootLogin no
```

Questo impedisce l'autenticazione con password (meno sicura di quella con
chiave pubblica) e impedisce l'accesso ssh all'utente root, impedendo di
ottenere il controllo del sistema con la sola chiave privata (occorre anche
conoscere la password dell'utente root).

## Docker

Docker è un gestore di _container_. I container sono delle sandbox in cui
eseguire programmi in modo sicuro e compartimentalizzato, in più è molto facile
automatizzare la creazione e il deployment dei container, a partire da immagini
pre costruite e file di configurazione appositi.

La [guida ufficiale di docker](https://docs.docker.com/engine/install/)
spiega come installarlo su una macchina. Sostanzialmente bisogna aggiungere il
repository e scaricare i pacchetti con `apt-get`.

```sh
# scarica la chiave di firma
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg

# aggiunge il repository per ubuntu focal
echo "deb [arch=amd64 signed-by=/etc/apt/trusted.gpg.d/docker.gpg] https://download.docker.com/linux/ubuntu \
      focal stable" | sudo tee /etc/apt/sources.list.d/docker.list

# aggiorna la cache e installa docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# aggiunge l'utente al gruppo docker
sudo usermod -aG docker $USER
```

In seguito, riavviare la macchina.

### docker compose

Docker compose è un _compositore_ per docker, in pratica è in grado di gestire
e creare docker a partire da un singolo file (`docker-compose.yml`), di questo
container si può impostare il comportamento, le risorse a cui accede (porte di
rete, cartelle condivise e variabili d'ambiente).

```sh
sudo apt-get install docker-compose-plugin
```

### ctop

`ctop` è un semplice ma potente pannello di controllo per docker. Permette di
vedere lo stato dei container sul sistema, fermarli, metterli in pausa,
cancellarli, vedere a quale risorse (porte di rete, cartelle condivise e
variabili d'ambiente) accede, leggere i log e creare una shell interattiva
dentro il container.

Si può scaricare l'eseguibile per la maggior parte delle architettura dal
[repository github](https://github.com/bcicen/ctop)

```sh
cp ctop-0.7.7-linux-amd64 ~/bin/ctop-0.7.7-linux-amd64
cd ~/bin
chmod +x ctop-0.7.7-linux-amd64
ln -s ctop-0.7.7-linux-amd64 ctop
```

**nota**: stiamo mettendo il programma in `~/bin`, per eseguirlo è necessario
aggiungere la cartella home al `$PATH`, aggiungendo queste linee al file
`~/.bashrc`

```sh
# add home/bin to path
# cambiare $HOME con il percorso della propria cartella home
export PATH=$PATH:$HOME/bin
```

e ricaricare il file, o riaprire la shell

```sh
source ~/.bashrc
```

---------------------------------------

# Creazione dei container

prima di creare un container, bisogna scaricare l'_immagine_ docker, che
contiene tutti i file necessari al funzionamento del programma. I file dei
dati, invece, verranno salvati su una cartella del sistema operativo, `/srv`, a
cui l'utente deve avere accesso, quindi gli diamo la proprietà.

```sh
docker compose pull
sudo chown $USER:$USER /srv
```

## Nginx

[Nginx](https://www.nginx.com/) è un server web, ovvero riceve richeste web
(http e https) e restituisce file html al browser, che le usa per costruire le
pagine web da mostrare all'utente. Nel nostro caso verrà usato come
"intermediatore" tra l'utente e il server di Nextcloud.

Questa configurazione si chiama _reverse proxy_, e ha diversi vantaggi:

1. permette di isolare il server web e il server nextcloud in due diversi
container, in modo che instrusioni in uno non compromettano l'altro

2. semplifica la configurazione, dato che ci permette di usare due container
mantenuti dagli stessi sviluppatori, invece di un container con i software
integrati, magari da terze parti.

3. permette di semplificare alcuni aspetti legati alla comunicazione con il
client, soprattutto quando installiamo diversi servizi e tutti hanno lo stesso
intermediatore. Per esempio, in questo caso se vogliamo mettere l'https (e lo
vogliamo) sarà necessario solo farlo sul server intermediatore, senza replicare
la configurazione su tutti i servizi installati.

I file di nginx (configurazione dei siti, pagine web, certificati di cifratura
e file di log) andranno dentro la cartella `/srv/nginx`.

Eseguiamo il container, controlliamo il log per la presenza di errori e, se
tutto va bene, eseguiamo il container in modalità _demone_, ovvero in
background, liberando la shell per altri programmi.

```sh
cd /srv
mkdir nginx
docker compose up www

# is there any error?
docker compose up www -d
```

Provare a collegarsi all'indirizzo ip (oppure hostname) della macchina
`http://192.168.xxx.xxx/`

## Mariadb

[Mariadb](https://mariadb.org/) è un _DataBase Management System_, versione
open-source di _MySQL_, gestisce database relazionali, ed è utilizzato da
Nextcloud per salvare le configurazoni degli utenti e i permessi ai file.

Quando creiamo un database, dobbiamo indicare il nome del primo database (ne
potremo creare altri per altre applicazioni), una password di root (che serve a
controllare tutti i database) e almeno un utente con la relativa password.
Questo utente dovrà essere indicato a Nextcloud in fase di configurazione per
accedere al database.

Tutti i file del database andranno dentro la cartella `/srv/mariadb`.

```sh
cd /srv
mkdir mariadb
docker compose up db

# is there any error?
docker compose up db -d
```

Al primo avvio il database ci metterà qualche minuto per creare la
configurazione iniziale. Se non ci sono errori, possiamo fare la stessa cosa
che con Nginx ed eseguirlo in remoto. Se dovessi mandargli altri comandi,
potremo farlo con `ctop`.

## Nextcloud

[Nextcloud](https://nextcloud.com/) è il cuore del nostro sistema di cloud. La
sua interfaccia permette di gestire i file del nostro utente, i calendari, i
contatti, usare e installare applicazioni. Il suo docker è semplicemente un
servizio web che, come abbiamo discusso prima, sarà "mascherato" dal nostro
intermediatore.

Tutti i file del database andranno dentro la cartella `/srv/nextcloud`. In
`data` ci saranno i dati del cloud (inclusi tutti i file), in `config` tutti i
file di configurazione del server, e in `apps` i dati delle applicazioni, sia
quelle di default che quelle che scaricheremo dallo store.

```sh
cd /srv
mkdir nextcloud
cd nextcloud
mkdir data
mkdir config
mdkir apps
docker compose up cloud
docker compose up cloud -d
```

Colleghiamo all'indirizzo del server alla porta 8080
`http://192.168.xxx.xxx:8080/`. Il motivo per cui non possiamo collegarci
direttamente con la porta 80 (quella di default di http, che non è necessario
specificare) è perché Nginx è in ascolto su quella porta, e solo un
programma alla volta può stare in ascolto su una porta. Successivamente
modificheremo la configurazione di nginx per risolvere questo problema.

La prima volta che viene eseguito, chiederà di creare un utente amministratore
(che può modificare ogni configurazione del server) e le credenziali di
collegamento al database, quello che abbiamo creato in pecedenza. Volendo
possiamo salvare i dati su un database `SQLite`, ma ciò è generalmente
sconsigliato, visto che un database su file è meno affidabile e performante di
un _DBMS_.

```
utente: admin
password: ldto2022
database: mysql/mariadb
    utente: nextcloud
    password: ldto2022
    nome database: nextcloud
    host database: ldto2022-mariadb:3306
```

Si noti che l'hostname del server mariadb è uguale al _container name_ scelto
in fase di configurazione del container. Questa è una funzionalità di docker
che semplifica la comunicazione tra i container, indipendentemente da come sono
assegnati gli indirizzi ip delle reti interne. Alternativamente, sarebbe stato
possibile indiare l'indirizzo ip del container, ma questo potrebbe cambiare nel
tempo e tra diverse configurazioni. Il _container name_, invece, lo scegliamo
noi,
    
Ci sarà un errore di permessi, infatti nextcloud usa l'utente `www-data` per
accedere alle cartelle dei dati, delle quali deve essere proprietario. Con
ulteriori configurazioni, potremmo dire a nextcloud di usare lo stesso utente
del sistema. Per ora, utilizziamo `ctop` per eseguire una shell dentro al
container e dare manualmente l'accesso alla cartella dei dati. Questa
operazione va fatta una solta volta.
```sh
chown -R www-data:www-data /var/www/html/data
```

Ci vorrà qualche minuto perché Nextcloud si configuri, ma dopo sarà
completamente funzionante. Da qui possiamo installare le nostre applicazioni e
creare nuovi utenti.

### Nginx reverse proxy

Il file `099-ldto.conf` contiene la configurazione per usare Nginx come reverse
proxy, installarlo in `/srv/nginx/nginx/site-confs` ci permette di utilizzare
l'hostname corretto.

### Trusted domains

Avendo cambiato l'host del server (prima ci siamo collegati a
`http://192.168.xxx.xxx:8080/` e Nextcloud lo ha salvato come host autorizzato
del server, adesso ci stiamo collegando a `http://192.168.xxx.xxx/`m con una
porta diversa), dobbiamo modificare i file di configurazione (in
`/srv/nextcloud/config/config.php`)

```php
  'trusted_domains' => 
  array (
    0 => '****',
    1 => '****',
  ),
  
  'overwrite.cli.url' => 'http://****',
```

`overwrite.cli.url` viene usato per gestire i redirect delle pagine (se
volessimo per esempio usare Nextcloud in una sottocartella del dominio),
conviene specificarlo correttamente o Nextcloud potrebbe rimandarci al vecchio
host, ora inaccessibile.

---------------------------------------

# Extra

## https

https (http secure) è un protocollo che crea una layer di protezione sopra al
protocollo http, che permette di verificare che il sito a cui ci stiamo
collegando sia effettivamente chi dichiara di essere. Per abilitare https è
necessario ricevere un certificato da una _certification authority_, che
certifica che il server e il dominio coincidono.

Per fare ciò, nel certificato è indicato il dominio (o i domini) per cui quel
certificato è valido. Inoltre, tutto il traffico sarà cifrato con una chiave
privata, la cui corrispondete chiave pubblica sarà inserita all'interno del
certificato. In questo modo il client che ci connette può verificare che il
certificato sia correttamente associato al server. In caso contrario, il
browser impedirà l'accesso.

Una certification authority che fornisce certificati gratuitamente è
[let's encrypt](https://letsencrypt.org/). La procedura per verificare e
rilasciare un certificato è la seguente:

1. specifichi per quale dominio (o domini) vuoi richiedere il certificato
2. let's encrypt fornisce dei token, e indica dove inserirli all'interno della
webroot, in modo che siano pubblicamente verificabili
3. let's encrypt verifica che i token che abbiamo inseriti siano quelli che ci
ha fornito, in questo modo sa che abbiamo la "proprietà" del server

Possiamo usare programmi che fanno questo lavoro in automatico, uno di questi è
[getssl](https://github.com/srvrco/getssl).

```sh
# crea la configurazion per il nostro certificato
getssl -c $hostname

# modifichiamo il file di configurazione del certificato, indicando
# indicando i domini, dove copiare i token ricevuti, e come installare
# i certificati
cd ~/.getssl/$hostname

# genera il certificato
getssl $hostname
```

## scanner webserver e nextcloud

Sono presenti molti scanner online per verificare che le nostre impostazioni
siano sicure. Tipicamente controllano che non si utilizzino versioni del
software obsolete o con vulnerabilità note, oppure che le configurazioni siano
corrette e sicure.

- [scanner dell'installazione nextcloud](https://scan.nextcloud.com/)
- [scanner della configurazione https](https://www.ssllabs.com/ssltest)
- [scanner degli header https](https://securityheaders.com/)

## onlyoffice

[OnlyOffice](https://www.onlyoffice.com/it/) è un software di scrittura
documenti online, simile a Office Online, ma open source. Oltre alle
soluzioni commerciali, permette di scaricare una versione self-hosted del
_document server_, e pure un'[integrazione nextcloud](https://apps.nextcloud.com/apps/onlyoffice).

Oltre all'applicazione Nextcloud, è necessario installare il _document server_,
anche come [docker](https://hub.docker.com/r/onlyoffice/documentserver/) e un
editor stand alone per modificare desktop offline.


