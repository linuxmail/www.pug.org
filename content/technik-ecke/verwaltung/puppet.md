---
title: Puppet
type: page
publishdate: "2019-08-23T23:04:00+02:00" 
toc: true
creatordisplayname: Denny Fuchs
creatoremail: "linuxmail at 4 lin dot net"
tags:
  - puppet
---

# Puppet

# Das Ziel

{{% toc %}}

In diesem Artikel geht es nicht darum zu erläutern was Puppet ist oder kann, sondern wie es eingerichtet wird.\
Wer wissen möchte, was Puppet ist, schaut am Besten hier vorbei: https://www.dev-insider.de/was-ist-puppet-a-720552/

Es dient als Grundlage für die Icinga2 Installation und wie sie aktuell bei mir umgesetzt wurde. Das Ziel ist es, Puppet und Icinga2 zusammen zu bringen und eine (im Ansatz) automatisierte Monitorlösung zu haben. Werden also neue Hosts hinzugefügt und lässt sich mit einem Puppet Agent ausstatten, dann sollte dieser Host auch im Icinga2 auftauchen.\
Auch das Gegenteil ist wünschenswert: fällt ein Host weg (weil dieser nicht mehr benötigt wird), kann dieser auch aus dem Monitoring verschwinden. 

## Konzept

Das Konzept sieht so aus, dass jedem Host eine(!) Rolle (Webserver / Mailserver / Jabber ..) zugewiesen wird, welche aus mehreren Profilen bestehen kann. Ein Beispiel:

* **role::webserver::nginx**
  * common -> Enthält Standardpakete (tmux/htop/fail2ban/ntpd...) / Konfigurationen (NTP Server / tmuxrc ..)
  * profile::webserver -> Kann z.B. für passende Firewallregeln sorgen ...
  * profile::webserver::nginx -> Wir wollen den Nginx als Webserver
  * profile::webserver::nginx::php -> Alles was für PHP nötig ist

Aufgrund dieser Rolle und deren Zusammensetzung, können wir später diese Informationen für unser Monitoring verwenden.

## Voraussetzungen

* Man sollte mit der Kommandozeile umgehen können
* Debian mögen
* Einen Rechner mit Git und SSH Zugriff auf die VMs/Hosts
* Zeit
* Ein wenig Ahnung von Puppet
* Drei VMs für:
    - PuppetMaster VM
    - VM1 zum testen (später für Icinga2 Master)
    - VM2 zum testen (später für Icinga2 client)

Klar, theoretisch würde es auch mit einem Host klappen, aber so sind die Zusammenhänge besser zu verstehen.\
Für die Anleitung habe ich mir drei VMs (LXC Container mit Debian Buster) erzeugt (Dank an [ProxMox](https://www.proxmox.com/de/)) und mittels Standard Debian 10.0 befüllen lassen. 
Es empfiehlt sich außerdem [tmux](https://www.grund-wissen.de/linux/shell/tmux.html) mit [xpanes](https://github.com/greymd/tmux-xpanes) zu verwenden, um zügig Aufgaben parallel abarbeiten zu können.

{{% alert theme="info" %}}Wir werden alles im Git ablegen, daher sollten auch die allerwichtigsten Git Kommandos bekannt sein{{% /alert %}}

### Git

Bevor es richtig losgeht, benötigen wir eine Grundstruktur und initialisieren auch gleich unser Git Repository. Dies kann entweder auf dem Arbeitsrechner passieren, oder  -- wie in diesem Beispiel --  auf dem Puppet Master Host.\
Ich persönlich bevorzuge es, das Git auf dem Puppet Master zu initialisieren und dann per Git clone auf den Arbeitsrechner zu holen. Dafür schaffen wir nun die Grundlage.\
Damit `pull`  und `push` auf beiden Seiten funktioniert, benötigen wir ein "bare" Git Repository:

* **User, Verzeichnisse und Git:**

```bash
root@office-ffm-srv-puppet:~# apt install git vim 
root@office-ffm-srv-puppet:~# mkdir /opt/{puppet.git,puppg}
root@office-ffm-srv-puppet:~# useradd -r -s /bin/bash -d /opt/puppet puppet
root@office-ffm-srv-puppet:~# chown puppet: /opt/puppet*
root@office-ffm-srv-puppet:~# su - puppet

puppet@office-ffm-srv-puppet:/opt/puppet$ cd ../puppet.git
puppet@office-ffm-srv-puppet:/opt/puppet.git$ git init --bare
```
Damit bei einem `git push` unser späteres `/opt/puppet/` aktualisiert wird wenn wir vom heimischen Rechner arbeiten, legen wir ein Git **hook** an:

```bash
puppet@office-ffm-srv-puppet:/opt/puppet.git$ echo -e '#!/bin/sh\necho "Update Puppermaster"\ncd /opt/puppet\nenv -i git pull' > hooks/post-receive
puppet@office-ffm-srv-puppet:/opt/puppet.git chmod +x hooks/post-receive
```
Damit wird im Ordner `/opt/puppet` automatisch ein `git pull` ausgeführt.

* **Git clone:**

Damit der Hook klappt, müssen wir zuvor ein `git clone` ausführen und als Ziel unser `/opt/puppet/` angeben:

```bash
puppet@office-ffm-srv-puppet:/opt/puppet.git$ cd ../
puppet@office-ffm-srv-puppet:/opt$ git clone /opt/puppet.git /opt/puppet/
```
Damit hätten wir nun ein leeres Verzeichnis mit Git Struktur. Im Anschluss richten wir ein paar Basis Informationen ein:

```bash
puppet@office-ffm-srv-puppet:/opt/puppet.git$ cd
puppet@office-ffm-srv-puppet:/opt/puppet$ git config --local user.name "Puppetmaster"
puppet@office-ffm-srv-puppet:/opt/puppet$ git config --local user.email "Puppetmaster@example.com"
puppet@office-ffm-srv-puppet:/opt/puppet$ git config --local merge.tool vimdiff
puppet@office-ffm-srv-puppet:/opt/puppet$ echo '.viminfo' >> .gitignore ; git add .gitignore ; git commit -m "Add Git ignore file" ; git push
```

Da wir per SSH arbeiten, muss der eigene öffentliche SSH Schlüssel auf den Puppet Master Host, sodass wir uns als Benutzer "puppet" einloggen können. Daher muss der öffentliche Schlüssel passend abgelegt werden:

* **/opt/puppet/.ssh/authorized_keys**

```bash
puppet@office-ffm-srv-puppet:/opt/puppet$ mkdir .ssh ; chmod 700 .ssh ; cd .ssh
puppet@office-ffm-srv-puppet:/opt/puppet/.ssh$ echo 'ssh-rsa AAAAB3NzaC1yc2EA...' > authorized_keys ; chmod 600 authorized_keys
```
* **SSH**

Ich habe in diesem Fall alles per Hand erledigt, da ssh-copyid nur funktionieren würde, wenn der Benutzer "puppet" sich per Kennwort einloggen dürfte, was nicht der Fall ist.

{{% alert theme="warning" %}}Noch besser wäre es, wenn SSH so konfiguriert wurde, dass die autorisierten Schlüssel (siehe https://www.ssh.com/ssh/authorized_keys/openssh#sec-Location-of-the-Authorized-Keys-File) geschützt abgelegt werden, z.B. in /etc/ssh/keys/%u {{% /alert %}}

* **.gitignore**

Wenn wir nun ein `git status` ausführen, erhalten wir die Meldung, dass `.ssh` Git nicht bekannt ist.\
Entweder wird das Verzeichnis hinzugefügt, oder wir setzen es in die Ignore Datei von Git.\
Wer möchte, kann das `.ssh/`` Verzeichnis ebenfalls zum Git hinzufügen:

```bash
puppet@office-ffm-srv-puppet:/opt/puppet/$ git add .ssh
puppet@office-ffm-srv-puppet:/opt/puppet/$ git commit -m "Add ssh folder" ; git push
```

Da im Verlauf der Anleitung noch ein paar Verzeichnisse hinzukommen, sieht meine `.gitignore` so aus:

```
.viminfo
.bash_history
.eyaml/
.gem/
.gnupg/
.selected_editor
```

Damit wäre das Wichtigste abgedeckt und wir können das Repository auf dem Puppetmaster auf den Arbeitsrechner holen:

```bash
$ denny@home:~$
$ mkdir -p git ; cd git
$ git clone puppet@office-ffm-srv-puppet.4lin.net:/opt/puppet.git puppet
```

Ist SSH korrekt konfiguriert, wird Git SSH verwenden.

Für Icinga und Puppet YAML Dateien gibt es ein paar schöne Vim Plugins, die das Leben einfacher gestalten. Die Jungs von Netways haben da schon eine nette [Anleitung](https://blog.netways.de/2012/10/30/puppet-und-vim/) bereit gestellt.

## Puppet Master

Ein großer Fortschritt der mit Puppet4 kam, war, dass sich im Bereich von [Hiera](https://puppet.com/docs/puppet/5.4/hiera_intro.html) sehr viel getan hat. Während früher(tm) (<Puppet3) viel in Manifest Dateien definiert wurde, ist man dazu übergegangen Werte zum "Nachschauen" (keys), in Hiera abzulegen. Hiera ist also eine Key Value Datenstruktur.\
Damit ist es möglich einen Wert an vielen Stellen zu verwenden, oder ihn an anderer Stelle zu überschreiben.\
Ein einfaches Beispiel:\
Überall soll der NTP Server 192.168.1.10 verwendet werden, aber die Nodes, die die Rolle "NTP" Server innehaben, sollen "ptbtime1.ptb.de" und "ptbtime2.ptb.de" verwenden, mit der Ausnahme, dass der host "office-ffm-ntp-03" die IPv6 Adresse "2001:638:610:be01::103" als Quelle verwenden soll.

### Apt

Aktuell gibt es nur Puppet 5.5 von Debian, da Puppet.org selbst noch keine Pakete für Buster hat (11.08.2019). Da dies aber die [nächsten Tage](https://tickets.puppetlabs.com/browse/PA-2410) passieren wird, binden wir schon einmal das Apt Repo ein, allerdings nicht nur das von Buster, **sondern auch auch das von Stretch**. Das bedeutet, wir installieren die Stretch Version von Puppet6 auf Buster.\
Es gibt lediglich ein Paket, welches aus Stretch nachgezogen muss, und dies ist [openjdk-8-jre-headless](https://packages.debian.org/stretch/amd64/openjdk-8-jre-headless/download). Sobald die Buster Version erscheint, wird Debian die Pakete automatisch aktualisieren und das JDK Paket kann wieder entfernt werden.\
Das Debian Paket von Puppet (Version 5.5) wird nicht verwendet, dazu viele Kleinigkeiten wie Pfade etc. anders sind.

```bash
root@office-ffm-srv-puppet:~# wget https://apt.puppetlabs.com/puppet6-release-buster.deb; dpkg -i puppet6-release-buster.deb && apt-get update

root@office-ffm-srv-puppet:~# wget https://apt.puppetlabs.com/puppet6-release-stretch.deb; dpkg -i puppet6-release-stretch.deb && apt-get update

root@office-ffm-srv-puppet:~# wget http://security.debian.org/debian-security/pool/updates/main/o/openjdk-8/openjdk-8-jre-headless_8u222-b10-1~deb9u1_amd64.deb
root@office-ffm-srv-puppet:~# dpkg -i openjdk-8-jre-headless_8u222-b10-1~deb9u1_amd64.deb
```

### Installation

Dann werden wir nun den Puppetmaster zzgl. benötigter Tools installieren:

```bash
root@office-ffm-srv-puppet:~# apt install puppetmaster augeas-tools
```

Puppetmaster wird sich beim Start 2GB Arbeitsspeicher nehmen, wer nicht so viel hat, muss an der passenden Schraube drehen. Wir wollen z.B: nur 512M dafür verwenden:

* **/etc/default/puppetserver:**

```config
# Modify this if you'd like to change the memory allocation, enable JMX, etc
 JAVA_ARGS="-Xms512m -Xmx512m"
```

### Konfiguration

Ein wichtige Grundvoraussetzung für Puppet ist, dass DNS funktioniert. Dies kann entweder über `/etc/hosts` laufen, oder in Kombination mit dem einfachen DNS Server **dnsmasq**.

Die Basiskonfiguration kann wie folgt aussehen:

* /etc/puppetlabs/puppet/puppet.conf

```config
[master]
    vardir = /opt/puppetlabs/server/data/puppetserver
    logdir = /var/log/puppetlabs/puppetserver
    rundir = /var/run/puppetlabs/puppetserver
    pidfile = /var/run/puppetlabs/puppetserver/puppetserver.pid
    # codedir = /opt/puppet/environments
    # Default is $codedir
    environmentpath = /opt/puppet/environments
    ssl_client_header = SSL_CLIENT_S_DN
    ssl_client_verify_header = SSL_CLIENT_VERIFY
    storeconfigs = false
    storeconfigs_backend = puppetdb
    hiera_config = /etc/puppetlabs/puppet/hiera.yaml

[agent]
report=false
ca_server = office-ffm-srv-puppet.4lin.net
server = office-ffm-srv-puppet.4lin.net
environment = dev
```

Vor dem ersten Start korrigieren wir noch das SSL Verzeichnis, da der Puppetmaster sonst nicht starten würde:

```bash
root@office-ffm-srv-puppet:~# chown puppet: -R /etc/puppetlabs/
```

Im Anschluss kann der Puppetserver gestartet werden. Der Dienst benötigt meist etwas Zeit, bis dieser gestartet wurde. Also Geduld haben und ein Blick in die Logs werfen.

```bash
root@office-ffm-srv-puppet:~# service puppetserver start ; tail -F /var/log/puppetlabs/puppetserver/*.log
```

Eventuell kann es sein, dass der Puppetserver sich kein zweites Mal starten lässt, wegen fehlender Berechtigungen für das SSL Verzeichnis und dem User "puppet". In diesem Fall wie oben angemerkt, das angegebene Verzeichnis korrigieren: 

```bash
...
Caused by: org.jruby.exceptions.RuntimeError: (RuntimeError) Got 1 failure(s) while initializing: File[/etc/puppetlabs/puppet/ssl]: change from 'absent' to 'directory' failed: Could not set 'directory' on ensure: Permission denied - /etc/puppetlabs/puppet/ssl
```

### PuppetDB

Als nächstes benötigen wir PostgreSQL für die Puppetmaster Datenbank "puppetdb". Der Puppetserver dient nur dafür um die Ressourcen bereitzustellen, die dann von dem Puppet Agent abgerufen werden. Die Eigenschaften (Facts) des Hosts die der Agent auswertet, wird nirgends zentral gespeichert. Genau das ist die Aufgabe vom "puppetdb". Alle Facts werden vom Puppetmaster an Puppetdb weitergegeben und liegen dann dort zentral vor. Damit ist es dann zum Beispiel möglich mittels "@@" auf diese Informationen von jedem Puppet Agent zu zugreifen.\
In unserm Fall benötigen wir diese Informationen allerdings für Icingaweb und das Director Modul.\
Leider unterstützt PuppetDB im Grunde nur PSQL, daher muss dieser ebenfalls bereitgestellt werden:

```bash
root@office-ffm-srv-puppet:~# apt install postgresql postgresql-contrib
```

Folgt die Basiskonfiguration für PostgreSQL. Da wir UTF8 verwenden, müssen wir zuerst das Template1 löschen und mit UTF8 neu erzeugen, bevor wir die Datenbank für PuppetDB anlegen können:

```bash
root@office-ffm-srv-puppet:~# su - postgres
postgres@office-ffm-srv-puppet:~$ createuser -DRSP puppetdb
Enter password for new role:
Enter it again:

postgres@office-ffm-srv-puppet:~$ psql

postgres=# UPDATE pg_database SET datistemplate = FALSE WHERE datname = 'template1';
UPDATE 1
postgres=# DROP DATABASE template1;
DROP DATABASE
postgres=# CREATE DATABASE template1 WITH TEMPLATE = template0 ENCODING = 'UNICODE';
CREATE DATABASE
postgres=# UPDATE pg_database SET datistemplate = TRUE WHERE datname = 'template1';
UPDATE 1
postgres=# \c template1
You are now connected to database "template1" as user "postgres".
template1=# VACUUM FREEZE;
VACUUM
template1=# \q
postgres@office-ffm-srv-puppet:~$ createdb -E UTF8 -O puppetdb puppetd
```

Danach kann die Datenbank "puppetdb" wie oben gezeigt, erstellt werden. Nach der Datenbank wird als nächstes eine Erweiterung hinzugefügt, welche sich im jeweiligen "postgresql-contrib-x.y" Paket verbirgt: 

```bash
root@office-ffm-srv-puppet:~# su - postgres
postgres@office-ffm-srv-puppet:~$ psql puppetdb -c 'create extension pg_trgm'
CREATE EXTENSION
postgres@office-ffm-srv-puppet:~$ exit
```

Nun erlauben wir noch eine Verbindung per Passwort:

```bash
root@office-ffm-srv-puppet:~# echo 'local all all md5'  >> /etc/postgresql/11/main/pg_hba.conf
```

Nun können wir PostgreSQL neustarten und das Kennwort prüfen:

```bash
root@office-ffm-srv-puppet:~# service postgresql restart
root@office-ffm-srv-puppet:~# psql -h localhost puppetdb puppetdb
Password for user puppetdb:
psql (11.5 (Debian 11.5-1+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

puppetdb=> \q
root@office-ffm-srv-puppet:~#
```

Damit ist der PostgreSQL Teil abgeschlossen, folgt PuppetDB selbst:

```bash
root@office-ffm-srv-puppet:~# apt install puppetdb puppetdb-termini
```
Bei der Ersteinrichtung werden die SSL Zertifikate die zuvor von der Installation von Puppetserver bzw. puppet-agent erstellt worden sind, nach `/etc/puppetlabs/puppetdb/ssl` kopiert. Wurde zuerst puppetdb installiert oder eine neue CA aufgesetzt, kann das Setup mittels `puppetdb ssl-setup` erneut aufgerufen werden.\
PuppetDB befüllt die Datenbank automatisch, es müssen lediglich die Daten für die Authentifizierung hinterlegt werden.

* **/etc/puppetlabs/puppetdb/conf.d/database.ini**

```config
[database]
subname = //localhost:5432/puppetdb
username = puppetdb
password = secret
gc-interval = 60
log-slow-statements = 10

# expired nodes after 90days with no puppet run
node-ttl = 90d

# purge nodes after 100 days with no puppet run
node-purge-ttl = 100d
```

```bash
root@office-ffm-srv-puppet:~# service puppetdb start ; tail -F /var/log/puppetlabs/puppetdb/*.log
```

Der Start dauert auch hier ein wenig ...

Nun kommt der letzte Teil für PuppetDB. Wir müssen dem Puppetmaster noch mitteilen, dass die Daten an PuppetDB übergeben werden sollen. Dazu müssen die passenden Zeilen in der puppet.conf hinterlegt werden. Dazu bennötigt es in der Datei `/etc/puppetlabs/puppet/puppet.conf` den Wert `storeconfigs = false` auf `storeconfigs = true` zu setzen ... :

```bash
root@office-ffm-srv-puppet:~# sed -i 's/storeconfigs = false/storeconfigs = true/g' /etc/puppetlabs/puppet/puppet.conf
```

...  und die Datei `/etc/puppetlabs/puppet/routes.yaml` anzulegen, mit folgendem Inhalt:

```yaml
---
master:
  facts:
    terminus: puppetdb
    cache: yaml
```

Wird nun der Puppetmaster neugestartet, werden zwar alle Tabelle angelegt und mit ersten Informationen gefüttert. Das reicht allerdings noch nicht ganz und es benötigt noch die Datei `puppetdb.conf`, welche in der Regel noch nicht existiert:

* **/etc/puppetlabs/puppet/puppetdb.conf**

```config
[main]
server_urls = https://office-ffm-srv-puppet.4lin.net:8081
```

```bash
root@office-ffm-srv-puppet:~# service puppetserver restart
```
Damit wäre die PuppetDB Konfiguration abgeschlossen.

### Hiera

Normalerweise würde man nun damit beginnen, die Manifests für Module (Apt/Webserver...) anzulegen, um diese dann für Nodes zu verwenden.\
**Das Problem:** schon bei nur wenigen Nodes (aka Hosts) würden früher oder später Zeilen doppelt geschrieben werden müssen, weil es Ausnahmen gibt. Zum Beispiel soll der Webserver den internen NTP Server verwenden, der NTP Server selbst jedoch von pool.ntp.org. Das wäre an dieser Stelle noch "einfach" zu lösen, doch je länger eine Puppet Infrastruktur "lebt", desto komplizierter wird es Ausnahmen zu erstellen, ohne ganze Ketten von Profilen, Modulen und Nodes umzuschreiben. Diesem Problem soll Hiera entgegenwirken, in dem Redundanzen vermieden, oder zumindest stark reduziert werden. Es werden nur noch Module mit Eigenschaften gepflegt; die Attribute kommen aus einer anderen Quelle.\
Doch wo viel Licht .... ist der Schatten nicht weit entfernt. Ein Problem welches sich recht schnell einstellen kann: Wo wurde welche Wert definiert !? Unter Umständen kann es bei einer gewachsenen Struktur passieren, dass nicht mehr schnell ersichtlich wird, an welcher Stelle welcher Wert verwendet wird, da dies in Hiera nicht gekennzeichnet wird. Dies ist besonders der Fall, wenn aus unterschiedlichen Orten die Werte verschmolzen werden.\
Des weiteren muss einem immer klar sein, dass Hiera nur eine "Nachschau- Tabelle" (lookup table) ist, es gibt **keine Fallunterscheidungen**.

#### Konfiguration

Hiera verwendet als Sprache [YAML](https://de.wikipedia.org/wiki/YAML) die gleich viele Verfechter, wie auch Gegner hat :-)\
Wichtig ist wie bei Python auch: Auf Einrückung achten (Leerzeichen).

Zuerst muss definiert werden, wo sich die `hiera.yaml` Datei befindet. Das wiederum wird in der puppet.conf definiert:

```bash
root@office-ffm-srv-puppet:~# grep hiera /etc/puppetlabs/puppet/puppet.conf
    hiera_config = /etc/puppetlabs/puppet/hiera.yaml
```

* **/etc/puppetlabs/puppet/hiera.yaml**

```config
---
# Hiera 5 Global configuration file

version: 5
defaults:
  datadir: "/opt/puppet/environments/%{environment}/hieradata"
  data_hash: yaml_data
hierarchy:
  - name: "Settings and secrets Per-node level"
    path: "node/%{::fqdn}.eyaml"
    lookup_key: eyaml_lookup_key
    options:
      gpg_gnupghome: /opt/puppetlabs/server/data/puppetserver/.gnupg
      gpg_recipients: 'denny@example.com,puppet@office-ffm-srv-puppet.4lin.net'
  - name: "Settings per server rack. No secrets here"
    path: "rack/%{::rack}.yaml"
  - name: "Setting on the Datacenter level. No secrets available here"
    path: "datacenter/%{::datacenter}.yaml"
  - name: "Settings per server role. No secrets here"
    path: "role/%{::role}.yaml"
  - name: "Common settings. Shared secrets could be saved here"
    path: "common.eyaml"
    lookup_key: eyaml_lookup_key
    options:
      gpg_gnupghome: /opt/puppet/.gnupg
      gpg_recipients: 'denny@example.com,puppet@office-ffm-srv-puppet.4lin.net'
```

Diese Konfiguration hat eine Besonderheit, auf die wir später noch eingehen werden: per GPG verschlüsselte Werte.\
In jedem der Pfade lassen sich Werte definieren und auch überschreiben. Die Abfolge lässt sich am Besten von unten nach oben lesen. Der obere Wert kann von einem Wert darunter überschrieben werden:

1. common.eyaml -> Global und verschlüsselte Werte (eyaml) möglich
2. role/%{::role}.yaml -> Nutzt das Fact "role", zum Beispiel `mongodb.yaml`
3. datacenter/%{::datacenter}.yaml zum Beispiel `ffm.yaml`
4. rack/%{::rack}.yaml zum Beispiel `office.yaml
5. node/%{::fqdn}.eyaml zum Beispiel `office-ffm-srv-puppet.4lin.net.eyaml`

Grob gesprochen ist die `common.eyaml` die Gieskanne und `node/` die Pipette.

#### Verschlüsselung

Um Passwörter, Token und Co. sicher abzuspeichern, gibt es bei Puppet die Möglichkeit diese [Verschlüsseln zu lassen](https://github.com/voxpupuli/hiera-eyaml). Dafür wird das Kommando `eyaml` verwendet.\
Es gibt die Variante PKCS7, oder mittels GnuPG. In diesem Fall verwenden wir die GnuPG Variante, da bereits relativ viele ein Schlüsselpaar für E-Mails etc. haben dürften. Der Charme liegt darin, dass mehrere Signaturen hinzugefügt werden können.

Dazu müssen ein paar weitere Pakete auf dem Puppetmaster installiert werden. Dies wird allerdings nicht über Debian erledigt, sondern wir installieren die Ruby Pakete mittels `puppetserver gem`:

```bash
root@office-ffm-srv-puppet:~# su - puppet
puppet@office-ffm-srv-puppet:~$ puppetserver gem install hiera-eyaml ruby_gpg hiera-eyaml-gpg ; exit
root@office-ffm-srv-puppet:~# service puppetserver restart
```

Damit der Puppetmaster die Werte selbst wieder entschlüsseln kann, muss auch für ihn ein Schlüsselpaar erzeugt werden:

{{% alert theme="warning" %}}**Um es einfach zu halten, wird der private Schlüssel ohne Passphrase erzeugt, da andernfalls bei jedem Neustart der gnupg-agent mit der Passphrase gestartet werden müsste. In einer "unsicheren" Umgrbung sollte dies angepasst werden.**{{% /alert %}}

```bash
root@office-ffm-srv-puppet:~# apt install gpg2 -y
```

[Interessanterweise](https://unix.stackexchange.com/questions/477445/www-data-user-cannot-generate-gpg-key) kann es beim Erzeugen des Schlüsselpaars zu einem "permission denied" führen, wenn mittels "su - puppet" zum User gewechselt wird. Dieser Fehler tritt nicht auf, wenn per ssh puppet@office-ffm-srv-puppet.4lin.net eingeloggt wird:

**Wichtig: den Schlüssel ohne Passphrase erzeugen**

```bash
denny: $ ssh puppet@office-ffm-srv-puppet.4lin.net

puppet@office-ffm-srv-puppet:~$ gpg --full-generate-key

   (1) RSA and RSA (default)

Key is valid for? (0) 0

Real name: Puppetmaster
Email address: puppet@office-ffm-srv-puppet.4lin.net
Comment: Puppetmaster Hiera encryption
You selected this USER-ID:
    "Puppetmaster (Puppetmaster Hiera encryption) <puppet@office-ffm-srv-puppet.4lin.net>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
```

Am Ende sollte ein Schlüsselpaar vorhanden sein:

```bash
puppet@office-ffm-srv-puppet:~$ gpg -K
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
/opt/puppet//.gnupg/pubring.kbx
-------------------------------
sec   rsa4096 2019-08-19 [SC]
      C212BBC91BD7924A739CE150FE99C1DAC528D188
uid           [ultimate] Puppetmaster (Puppetmaster Hiera encryption) <puppet@office-ffm-srv-puppet.4lin.net>
ssb   rsa4096 2019-08-19 [E]

puppet@office-ffm-srv-puppet:~$
```
Die Adresse muss mit der in der hiera.yaml Konfiguration übereinstimmen.\
Dieses Schlüsselpaar ist einzig für den Puppetmaster vorgesehen. Damit nun auch reguläre Personen (Sysops etc.) Daten ver- und entschlüsseln können, müssen die öffentlichen Schlüssel in den Schlüsselbund (keyring) importiert werden:

```bash
denny: $ ssh puppet@office-ffm-srv-puppet.4lin.net
puppet@office-ffm-srv-puppet:~$ gpg --import denny.asc
```
In denny.asc ist der öffentliche Teil meines Schlüssels enthalten und importiere diesen in den Schlüsselbund vom user `puppet`. Die ID muss ebenfalls in der hiera.yaml aufgelistet sein, damit `eyaml` auch diesen Schlüssel zum **verschlüsseln** benutzt. Dies muss für jede weitere Signatur getan werden. Es empfiehlt sich auch eventuell ein Schlüsselpaar als Backup anzulegen und diesen ebenfalls in hiera.yaml einzutragen, damit im Fall der Fälle mindestens ein Schlüssel für die Wiederherstellung genutzt werden kann.

Anschließend schadet ein Neustart des Masters nicht:

```bash
root@office-ffm-srv-puppet:~# service puppetserver restart
```
Ob auch alles klappt, wird sich im weiteren Verlauf zeigen.

## Basis

Für die ersten Schritte verwenden wir die `common.eyaml` und lassen die `/etc/apt/sources.list` von Puppet verwalten.\
Dazu installieren wir das puppetlabs-apt Modul und erstellen die Grundstruktur. Damit die Übersicht nicht zu sehr leidet, lasse ich den Arbeitspfad ($PWD) in den Shells weg. Der Hauptarbeitsordner ist: `/opt/puppet/environments`. 

Es empfiehlt sich folgende Struktur:

* production/ -> Puppet Module die per "puppet module install" installiert werden
* live/ -> Arbeitsumgebung für die produktive (live) Umgebung
* dev/ -> Arbeitsumgebung zum testen und entwickeln 

```bash
$ mkdir production/modules
$ puppet module install puppetlabs-apt --modulepath production/modules/
Notice: Preparing to install into /opt/puppet/environments/production/modules ...
Notice: Downloading from https://forgeapi.puppet.com ...
Notice: Installing -- do not interrupt ...
/opt/puppet/environments/production/modules
└─┬ puppetlabs-apt (v7.1.0)
  ├── puppetlabs-stdlib (v6.0.0)
  └── puppetlabs-translate (v2.0.0)
puppet@office-ffm-srv-puppet:/opt/puppet/environments$ ls production/modules/
apt  stdlib  translate
```

Danach erstellen wir unsere `dev` Umgebung und erstellen sowohl das Verzeichnis, als auch die Abarbeitung der Module:

* **/opt/puppet/environments/dev/environment.conf:**

```bash
$ mkdir -p dev/{modules,files,templates,manifests,hieradata}
$ echo 'modulepath = /opt/puppet/environments/dev/modules:/opt/puppet/environments/live/modules:/opt/puppet/environments/production/modules' > /opt/puppet/environments/dev/environment.conf
```
Die Reihenfolge ist wichtig. Damit kann ein Modul von "production/modules" nach z.B. "dev/modules" kopiert **und verändert** werden.

Nun erstellen wir eine **site.pp**, um die Klassen über Hiera einbinden zu können:

* **/opt/puppet/environments/dev/manifests/site.pp**

```puppet
# Default site manifests

hiera_include('classes','')
node default {
}
```

Nun folgt unsere YAML Datei, die für alle gilt:

* **/opt/puppet/environments/dev/hieradata/common.eyaml**

Kurz anzumerken ist, dass wir die Endung .**e**yaml, da sie später auch verschlüsselte Werte enthalten soll. Für den Augenblick belassen wir es noch und füllen die Datei mit folgenden Werten:

{{% alert theme="warning" %}} **Auf die Einrückung achten !** {{% /alert %}}

```yaml
---
classes:
  - 'apt'
apt::purge:
  sources.list.d: true
  sources.list: true
apt::sources:
  'debian_stable':
    comment: 'This is the Debian stable mirror'
    location: 'http://ftp.debian.org/debian'
    release: 'buster'
    repos: 'main contrib non-free'
    include:
      src: false
      deb: true
  'debian_security':
    comment: 'This is the Debian stable security mirror'
    location: 'http://security.debian.org'
    release: 'buster/updates'
    repos: 'main contrib'
    include:
      src: false
      deb: true
  'debian_updates':
    comment: 'This is the Debian stable update mirror'
    location: 'http://ftp.debian.org/debian'
    release: 'buster-updates'
    repos: 'main contrib'
    include:
      src: false
      deb: true
  'puppetlabs_stretch':
    location: 'http://apt.puppetlabs.com'
    repos: 'puppet6'
    release: 'stretch'
    key:
      id: '8735F5AF62A99A628EC13377B8F999C007BB6C57'
      server: 'pgp.mit.edu'
```

Damit können wir auf dem Puppetmaster den Puppet Agent ausführen und schauen, ob alles klappt:

```bash
root@office-ffm-srv-puppet:~# puppet agent -t --noop
Info: Using configured environment 'dev'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Notice: /File[/opt/puppetlabs/puppet/cache/lib/facter]/ensure: created
...
Notice: /Stage[main]/Apt/File[sources.list]/content:
--- /etc/apt/sources.list   2019-07-08 07:33:13.000000000 +0200
+++ /tmp/puppet-file20190821-27485-14z7pa8  2019-08-21 22:05:26.393166073 +0200
@@ -1,6 +1 @@
-deb http://ftp.debian.org/debian buster main contrib
-
-deb http://ftp.debian.org/debian buster-updates main contrib
-
-deb http://security.debian.org buster/updates main contrib
-
+# Repos managed by puppet.
```

Damit wäre die Puppet soweit eingerichtet, dass es funktionsfähig ist. Zum Schluss werden wir alles in Git einchecken und prüfen, ob auch verschlüsselte Werte funktionieren.

### Abschlussarbeiten - Git / Eyaml

Bevor wir die neuen Dateien in Git einchecken, gehen wir auf Nummer sicher und ändern rekursiv den Eigentümer:

```bash
root@office-ffm-srv-puppet:~# chown puppet: -R /opt/puppet/

puppet@office-ffm-srv-puppet:/opt/puppet$ git add environments/
puppet@office-ffm-srv-puppet:/opt/puppet$ git commit -m "Add Initial environments" environments/ ; git push
```
Dann können wir ein `git pull` auf dem heimischen Rechner ausführen:

```bash
$ denny@home:~$ cd git/puppet
$ denny@home:puppet$ git pull
remote: Enumerating objects: 673, done.
remote: Counting objects: 100% (673/673), done.
remote: Compressing objects: 100% (636/636), done.
remote: Total 672 (delta 149), reused 0 (delta 0)
Empfange Objekte: 100% (672/672), 445.29 KiB | 10.12 MiB/s, Fertig.
Löse Unterschiede auf: 100% (149/149), Fertig.
Von puppet:/opt/puppet
   6312fd2..6f867b5  master     -> origin/master
Aktualisiere 6312fd2..6f867b5
Fast-forward
 environments/dev/environment.conf                                                                   |    1 +
 environments/dev/hieradata/common.eyaml                                                             |   40 ++
 environments/dev/manifests/site.pp                                                                 |    3 +
...
```

Damit liegen die Änderungen nun vor und wir können die Pakete installieren, um das Kommando `eyaml edit` verwenden zu können.
Da dies aber stark vom Betriebssystem abhängig ist, heißt es ein wenig zu recherchieren.

#### MacOS

Unter MacOS waren so einige Klimmzüge notwendig:

* Pfad für die Ruby Module definieren und einlesen

```bash
$ echo -e "export GEM_HOME=/Users/${USER}/.gem\n export PATH=\"\$GEM_HOME/bin:\$PATH\"" >> $HOME/.bash_profile && source $HOME/.bash_profile
```

* Module installieren:

```bash
$ gem install hiera-eyaml hiera-eyaml-gpg optimist highline -v 1.6.19 require puppet CFPropertyList -v 2.2
$ gem pristine gpgme --version 2.0.18
```

In `~/.gem/bin` finden sich nun die ausführbaren Dateien.\
Damit Eyaml weiß, welche Schlüssel zum signieren / verschlüsseln verwendet werden sollen, wird eine dazu passende Datei angelegt:

### Eyaml konfigurieren und testen

* **~/.eyaml/config.yaml**
```yaml
gpg_recipients: 'linuxmail@4lin.net,puppet@office-ffm-srv-puppet.4lin.net'
```
Es ist der gleiche Inhalt, wie auch auf dem Puppetmaster in `/opt/puppet/.eyaml/config.yaml`.

Das bedingt natürlich, dass der öffentliche Schlüssel vom Puppetmaster im eigenen Schlüsselbund vorzufinden ist. Daher exportieren wir ihn, um ihn dann zu importieren:

* **Export auf Puppetmaster**

```bash
puppet@office-ffm-srv-puppet:/opt/puppet$ gpg --export  --armour  puppet@office-ffm-srv-puppet.4lin.net > puppet.asc
```

* **Import in den eigenen Schlüsselbund**

Entweder die Datei per scp kopieren, oder klassisch per kopieren/einfügen:

```bash
$ gpg --import puppet.asc
$ gpg --edit-key puppet@office-ffm-srv-puppet.4lin.net
gpg> trust
Ihre Auswahl? 5
...
```

Wir müssen dem Schlüssel vertrauen, da wir andernfalls Probleme beim Eyaml bekommen.

* **Ausprobieren**

```bash
$ eyaml edit git/puppet/environments/dev/hieradata/common.eyaml
```

```yaml
foo: DEC::GPG[DeBiAn]!
```
Damit haben wir nun einen verschlüsselten Wert erstellt:

```
grep foo  environments/dev/hieradata/common.eyaml
foo: ENC[GPG,hQIOA0d1s4aH....==]
```

Auf diesen Wert lässt sich später zugreifen, sofern der Puppetmaster korrekt funktioniert.\
Am Einfachsten ist es, wir rufen den verschlüsselten Wert innerhalb von Hiera auf. Dazu werden wir einfach die bereits hinterlegen Apt Quellen "missbrauchen"

```yaml
...
  'debian_security':
    comment: "This is the %{hiera('foo')} stable security mirror"
...
```
Wenn es klappt, sollte im Kommentar in einer Datei verändert werden.
Der geänderte Wert lässt sich nun in Git einchecken und wir können prüfen, ob es klappt.

```bash
denny@home:puppet$ git commit -m "Add first secret"  environments/dev/hieradata/common.eyaml ; git push
```

Nun lassen wir Puppet mit `--noop` auf dem Puppetmaster laufen:

```bash
root@office-ffm-srv-puppet:/etc/apt/sources.list.d# puppet agent -t --noop
Info: Using configured environment 'dev'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Retrieving locales
Info: Loading facts
Info: Applying configuration version '1566589098'
Notice: /Stage[main]/Apt/Apt::Source[debian_security]/Apt::Setting[list-debian_security]/File[/etc/apt/sources.list.d/debian_security.list]/content:
--- /etc/apt/sources.list.d/debian_security.list    2019-08-21 22:05:37.193238953 +0200
+++ /tmp/puppet-file20190823-16703-1wxi356  2019-08-23 21:38:20.106406275 +0200
@@ -1,3 +1,3 @@
 # This file is managed by Puppet. DO NOT EDIT.
-# This is the Debian stable security mirror
+# This is the DeBiAn stable security mirror
 deb http://security.debian.org buster/updates main contrib
...
```
... und es hat geklappt. Eine Zeile wird gelöscht und durch eine andere (mit unserem Wert) ersetzt.

## Eigene Facts

Mit das Wichtigste an Puppet sind die Facts. Sie sind die Gewürze im Eintopf ! Es gibt bereits jede Menge Facts, diese lassen sich mit `facter -p` anzeigen. Einzelne Werte lassen sich dabei herausgreifen:

```bash
root@office-ffm-srv-puppet:~# facter -p virtual
lxc
```

Spannend wird es allerdings, wenn wir unsere eigenen Facts erstellen. Diese benötigen wir besonders für das Monitoring.\
Den Hostnamen "office-ffm-srv-puppet" habe ich bewusst so gewählt, denn dieser lässt sich sehr gut zerlegen:

* office -> rack
* ffm -> datacenter
* srv -> role

Damit diese Facts vorhanden sind, benötigen wir ein eigenes Puppet Modul, welches wir unter `environments/dev/modules/custom/` ablegen:

```bash
denny@home:puppet$ mkdir -p environments/dev/modules/custom/lib/facter/
```

In dem `facter` Ordner erzeugen wir zwei Dateien:
* role.rb
* datacenter.rb

* **role.rb**

```ruby
# Set the fact "role": facter -p role
# modules/custom/lib/facter/role.rb

# https://rnelson0.com/2014/07/14/intro-to-roles-and-profiles-with-puppet-and-hiera/
# hostname like office-ffm-srv-puppet has a role "srv"
if Facter.value(:hostname) =~ /^([a-z]+)-([a-z]+)-([a-z]+)-([a-z])+$/
  Facter.add('role') do
  setcode do
    $3
  end
  end

# ([a-z]+), i.e. www or logger have a role of www or logger
elsif Facter.value(:hostname) =~ /^([a-z]+)$/
  Facter.add('role') do
  setcode do
    $1
  end
  end

# Set to hostname if no patterns match
else
  Facter.add('role') do
    setcode do
      'default'
    end
  end
end
```

* **datacenter.rb**

```ruby
# Set fact for datacenter and rack
# https://rnelson0.com/2014/07/14/intro-to-datacenters-and-profiles-with-puppet-and-hiera/
# modules/custom/lib/facter/datacenter.rb

# ([a-z]+), i.e. hostname like office-ffm-srv-puppet has a datacenter "ffm"
if Facter.value(:hostname) =~ /^([a-z]+)-([a-z]+)-([a-z]+)-([a-z])+$/
  Facter.add('datacenter') do
  setcode do
  $2
  end
  end

# ([a-z]+), i.e. fra-corp-srv-debian have a datacenter of fra
elsif Facter.value(:hostname) =~ /^([a-z]+)-([a-z]+)-([a-z]+)-([a-z])+$/
  fact=$1
  Facter.add('datacenter') do
  setcode do
  fact
  end
  end

# Set to default if no patterns match
else
  Facter.add('datacenter') do
    setcode do
      'default'
    end
  end
end

######### Set rack #########

# hostname like office-ffm-srv-puppet has a rack "office"
if Facter.value(:hostname) =~ /^([a-z]+)-([a-z]+)-([a-z]+)-([a-z])+$/
  Facter.add('rack') do
  setcode do
  $1
  end
  end

# ([a-z]+), i.e. qh-a07-pmox-02 have a rack of a07
elsif Facter.value(:hostname) =~ /^([a-z]+)-([a-z]+)([0-9]+)-([a-z]+)-([0-9])+$/
  Facter.add('rack') do
  setcode do
  $2+$3
  end
  end

# Set to hostname if no patterns match
else
  Facter.add('rack') do
    setcode do
      'default'
    end
  end
end
```

Diese Ruby elsif Konstrukte lassen sich beliebig erweitern bzw. verändern, sodass sie auf eigene Bedürfnisse angepasst werden können.\

```bash
 denny@home:puppet$ git add environments/dev/modules/ ; git commit -m "Add custom facts"  environments/dev/modules/ ; git push 
```

Auf dem Puppetmaster sollte das nun klappt:

```bash
root@office-ffm-srv-puppet:~# puppet agent -t --noop ; facter -p datacenter
ffm

root@office-ffm-srv-puppet:~# puppet agent -t --noop ; facter -p rack
office

root@office-ffm-srv-puppet:~# puppet agent -t --noop ; facter -p role
srv
```

Als nächstes bringen wir zwei weitere Facts in Spiel:

* puppet_modules -> Listet alle Puppet Module auf, die für die Node verwendet werden
* puppet_classes -> Listet alle Puppet Klassen auf, die für diese Node aufgerufen werden

Damit ist es zum Beispiel möglich dem Icinga2 zu sagen: "Bitte führe die Service Checks für Nginx aus, wenn Nginx in puppet_classes auftaucht".

Damit das klappt, erstellen wir in dem `facter` Ordner eine Datei:

* puppet_classes.rb


```ruby
classes_file  = '/opt/puppetlabs/puppet/cache/state/classes.txt'
classes_hash  = {}
modules_array = []
File.foreach(classes_file) do |l|
  modules_array << l.chomp.gsub(/::.*/, '')
end
modules_array = modules_array.sort.uniq
modules_array.each do |i|
  classes_array = []
  classes_array << i
  File.foreach(classes_file) do |l|
    classes_array << l.chomp if l =~ /^#{i}/
      classes_array = classes_array.sort.uniq
  end
  classes_hash[i] = classes_array
end

Facter.add(
  :puppet_modules) do
  confine :kernel => 'Linux'
  setcode do
    modules_array.sort.uniq.join(', ').to_s
  end
  end
Facter.add(
  :puppet_classes) do
  confine :kernel => 'Linux'
  setcode do
    classes_hash.map { |_k, v| v }.sort.uniq.join(', ').to_s
  end
  end
```

Wurde das ins Git eingecheckt und der Puppet Agent auf der Node (z.B. Puppet Master) ausgeführt, sind die zwei neuen Facts verfügbar.

```bash
root@office-ffm-srv-puppet:~# facter -p puppet_modules
apt, default, settings
```

```bash
root@office-ffm-srv-puppet:~# facter -p puppet_classes
apt, apt::params, apt::update, default, settings
```

## Ende

Damit haben wir nun alle Voraussetzungen, um zum einen unsere Hosts mit Puppet zu konfigurieren, als auch später unser Monitoring mittels Icinga2 auf die Beine zu stellen.
