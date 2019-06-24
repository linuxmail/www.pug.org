---
title: MariaDB Galera Cluster
toc: true
tags:
  - datenbank
  - mariadb
---
# Inhalt
Für viele Dienste und Anwendungen benötigt es eine Datenbank. Mit Hilfe von MySQL (bzw. dem Fork MariaDB) oder PostgreSQL lässt sich schnell etwas herbeischaffen und auch befüllen.\
Leider steht und fällt vieles, wenn die Datenbank mal nicht zur Verfügung steht, daher ist es sinnvoll einen Datenbank Cluster zu konfigurieren, damit beim Ausfall von einer Node nicht alles steht.

Dieser Aspekt soll hier mittel MariaDB 10.x und der zugrunde liegenden Erweiterung [Galera Cluster](http://galeracluster.com/products/) umgesetzt werden.\
Der Vorteil von MariaDB gegenüber von z.B. PostgreSQL liegt darin, dass wir einen Master-Master Cluster erstellen können, bei dem auf alle Instanzen schreibend zugegriffen werden kann. Bei PostgreSQL ist dies nur mittels doch recht teurer Plugins möglich.

## Aufbau

 Der Klassiker besteht aus drei Knoten. Wenn es nicht die Möglichkeit gibt, drei Knoten aufzusetzen (VMs / Container / physisch), sollte als Quorum der [Galera Arbitrator](http://galeracluster.com/library/documentation/arbitrator.html) verwendet werden. Dieser kleine Dienst hat die Aufgabe eine weitere Stimme für die Wahl bereitzustellen (voting), wenn ein Knoten ausfällt und die Frage im Raum steht, wer ist der neue Master.

 ![Galera Cluster](/doc/uploads/datenbank/galera_cluster.png?height=400px)

### Voraussetzungen

 Der Autor verwendet Debian Stretch, allerdings ist die Vorgehensweise auf allen Distributionen sehr ähnlich, sobald die MariaDB installiert wurde und die Pfade klar sind.\

Als Vorbereitung habe ich **drei** Debian Stretch VMs installiert:

* 10GB LVM Festplattenplatz
* 1GB Ram

Da viele Kommandos auf allen Knoten ausgeführt werden müssen, empfehle ich die Kombination aus "tmux" und [xpanes](https://github.com/greymd/tmux-xpanes)

Das Einloggen sieht dann ungefähr so aus:

```sh
$ xpanes --ssh office-ffm-db-0{1..3}
```

Daraufhin öffnen sich drei "panes" und kann nun Kommandos an alle drei (oder mehr) Server senden.
Der Autor verwendet des weiteren [diese](https://github.com/gpakosz/.tmux) tmux Konfiguration.

### MariaDB 10.x

#### Installation

Als erstes fügen wir das [Repository](https://downloads.mariadb.org/mariadb/repositories/#mirror=host-europe) von MariaDB hinzu, da das von Debian schon etwas älter ist:

```sh
$ sudo apt install software-properties-common dirmngr
$ sudo apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 0xF1656F24C74CD1D8
$ sudo add-apt-repository 'deb [arch=amd64] http://ftp.hosteurope.de/mirror/mariadb.org/repo/10.4/debian stretch main'
```

Bevor die Pakete installiert werden, bereiten wir ein paar Verzeichnisse vor. In der Standardeinstellung werden die Datenbanken in `/var/lib/mysql` gespeichert. Hier wollen wir sie aber nach `/opt/mariadb/mysql` speichern.
Damit haben wir eine bessere Kontrolle und können im Bedarfsfall Snapshots von nur diesem Verzeichnis (Mountpoint) erstellen:

```sh
$ sudo mkdir -p /opt/mariadb/
$ sudo lvcreate -n mariadbfs -L 1G db-vg
$ sudo mkfs.ext4 /dev/db-vg/mariadbfs
$ echo '/dev/mapper/db--vg-mariadbfs /opt/mariadb ext4 noatime,defaults 0 2' | sudo tee -a /etc/fstab > /dev/null
$ sudo mount /opt/mariadb
$ sudo mkdir /opt/mariadb/{tmp,mysql}
```

* Hier passiert nun folgendes:
  * `/opt/mariadb/` Verzeichnis erstellen
  * Logical Volume mit der Größe von 1GB erzeugen mit dem Namen "`mariadbfs`" aus der Volume Gruppe `db-vg`
  * Das neu erstellte LV mit dem Dateisysten EXT4 formatieren
  * Den dazu passenden `/etc/fstab` Eintrag erzeugen
  * Das LV einhängen
  * Das Verzeichnis `tmp` erstellen, welches von MariaDB genutzt wird
  * das Verzeichnis `mysql` erstellen, in dem dann alle Datenbanken landen

Wichtig ist an dieser Stelle nur, dass das neue Dateisystem **vorher** eingehangen wurde, andernfalls wären die neu erstellten Ordner nicht mehr greifbar.\
Nun reichen zwei Pakete aus, um alle wichtigen Dienste und Programme auf die Server zu bringen: 

```sh
$ sudo apt update
$ sudo apt install mariadb-server galera-4
```

Wurde alles installiert, kann man bereits darauf zugreifen:

```sh
denny@office-ffm-db-01:~$ sudo mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 15
Server version: 10.4.6-MariaDB-1:10.4.6+maria~stretch-log mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

Allerdings wird die Konfiguration kräftig geändert und die bereits erstellten Datenbanken verschoben:

#### Konfiguration

Bevor die Konfiguration angepasst wird, stoppen wir die Datenbanken (ffm-office-db-0\{1..3\}) und verschieben die bereits erstellte:

```sh
$ sudo chown mysql: -R /opt/mariadb/
$ sudo systemctl stop mariadb 
$ sudo mv /var/lib/mysql/* /opt/mariadb/mysql/
```
Nun folgt die Konfiguration, welche von [hier entliehen](https://community.icinga.com/t/galera-mysql-cluster-with-vips-and-haproxy-for-ido-mysql-and-more/407#configure-the-galera-cluster) wurde, aber ein wenig angepasst und von Kommentaren befreit. Es gibt nur vier Änderungen:

- bind-address - externe IP Adresse
- wsrep\_node\_address - interne Cluster IP Adresse
- datadir - wo landen die Datenbanken
- tmpdir - wo ist das temporäre Verzeichnis

```cfg
# office-ffm-db-01:/etc/mysql/my.cnf

# based on https://community.icinga.com/t/galera-mysql-cluster-with-vips-and-haproxy-for-ido-mysql-and-more/407#install-mariadb--galera-cluster-filesets

[server]
[mysqld]

bind-address=192.168.1.30
max_allowed_packet = 16M

datadir=/opt/mariadb/mysql
tmpdir=/opt/mariadb/tmp

[galera]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
query_cache_size=16M
query_cache_type=0
innodb_flush_log_at_trx_commit=0
innodb_buffer_pool_size=256M

# Galera settings
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_sst_method=rsync
wsrep_slave_threads=2

# Which node to choose, if SST is required
# default should be null, or a dedicated node (name, NOT IP)
# wsrep_sst_donor=office-ffm-db-03 
# wsrep_sst_donor=null

# SSL for Galera, but disabled
# wsrep_provider_options="socket.ssl_key=/etc/mysql/ssl/  server-key.pem;socket.ssl_cert=/etc/mysql/ssl/server-cert.pem;socket.ssl_ca=/etc/mysql/ssl/ca-cert.pem"

# Node name and ip
wsrep_node_address='172.25.50.30'
wsrep_node_name='ffm-office-db-01'

#Gluster name and node ips
wsrep_cluster_name='MariaDB_Cluster'
wsrep_cluster_address='gcomm://172.25.50.30,172.25.50.31,172.25.50.32'

[embedded]
[mariadb]
[mariadb-10.4]
```

Für die anderen Knoten muss entsprechend die IP Adresse (bind-address / wsrep\_node\_address) und der Name (wsrep\_node\_name) angepasst werden.

#### Cluster Initialisierung

Nachdem nun die Konfiguration auf allen drei Knoten hinterlegt wurde, kann auf einem Knoten MariaDB gestartet werden, allerdings mit einem speziellen Kommando, welches den Galera Cluster sozusagen initialisiert: `galera_new_cluster`

Das Kommando forciert den Start des ersten Cluster Knotens. Würde das reguläre `systemctl start mariadb` verwendet werden, würde nach den in der Konfiguration hinterlegten anderen Nodes geschaut werden, ob es einen "Donor" gibt. Da das zu diesem Zeitpunkt nicht der Fall ist, würde der Start abbrechen.

Ein erfolgreicher Start sieht dann zumeist so aus:

```bash
Jun 23 19:28:47 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:47 0 [Note] WSREP: Member 0.0 (ffm-office-db-01) synced with group.
```

Starten wir nun einen weitere beliebigen Knoten, wird dieser den Donor finden und sich bei ihm melden:

```bash
Jun 23 19:28:47 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:47 0 [Note] WSREP: Shifting JOINED -> SYNCED (TO: 2)
Jun 23 19:28:47 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:47 3 [Note] WSREP: Server ffm-office-db-01 synced with group
Jun 23 19:28:47 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:47 3 [Note] WSREP: Server status change joined -> synced
Jun 23 19:28:47 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:47 3 [Note] WSREP: Synchronized with group, ready for connections
Jun 23 19:28:47 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:47 3 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
Jun 23 19:28:48 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:48 0 [Note] WSREP: (4e0d80e8, 'tcp://0.0.0.0:4567') turning message relay requesting off
Jun 23 19:28:49 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:49 0 [Note] WSREP: async IST sender served
Jun 23 19:28:49 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:49 0 [Note] WSREP: 1.0 (ffm-office-db-02): State transfer from 0.0 (ffm-office-db-01) complete.
Jun 23 19:28:49 office-ffm-db-01 mysqld[4857]: 2019-06-23 19:28:49 0 [Note] WSREP: Member 1.0 (ffm-office-db-02) synced with group.
```

Der Knoten `ffm-office-db-02` hat sich also angemeldet und die bestehenden Datenbanken wurden per `rsync` übertragen.

Damit haben wir nun einen funktionieren Datenbank Cluster

#### Cluster SST per mariabackup

Wenn ein Datenbank Knoten aus dem Cluster "herausgefallen" ist und erneut eingebunden wird, gibt es zwei Möglichkeiten um den aktuellen Stand zu syncronisieren:

* IST - [Incremental State Transfers](http://galeracluster.com/library/documentation/state-transfer.html#state-transfer-ist)
* SST - [State Snapshot Transfers](http://galeracluster.com/library/documentation/state-transfer.html#state-transfer-sst)

MariaDB hält in einem [Zwischenspeicher](http://galeracluster.com/library/documentation/galera-parameters.html#gcache-size) die geänderten Daten vor, welche für einen IST Transfer verwendet werden. Ist dieser Zwischenspeicher voll, wird stattdessen ein SST durchgeführt.\
Ein SST überträgt erneut alle Datenbanken. Je nach Umfang der Daten können das von wenige Sekunden, bis hin zu vielen Stunden dauern, abhängig von der Bandbreite, verbaute Festplatten etc. pp.\
Ein großes Problem bei der Methode über `rsync` ist der, dass der genutzte Donor vollständig blockiert wird. Der Donor Knoten wird als Quelle für den Transfer der Daten verwendet. In einem drei Knoten Cluster, wäre im Fall der Fälle nur noch ein Knoten für Applikationen verfügbar.

Dieses Problem lässt sich entschärfen, wenn als Methode `mariabackup` eingesetzt wird. Diese Methode ist ein Fork von xtrabackup-v2. Damit werden die Daten nicht per `rsync`, sondern mittels `socat` übertragen (sozusagen `cat` für das Netzwerk) und per `mariabackup` übergeben.

Bei einem SST wird dann nur die jeweilige Datenbank blockiert, nicht jedoch der gesamte Knoten.\
Dafür bedarf es eines speziellen Datenbank Nutzers und die jeweiligen Pakete auf alles Knoten. Um es einfach zu halten, verwenden wir die Benutzer:Passwort Variation:

* Anpassen der Konfiguration

```cfg
[mariadb]
...
wsrep_sst_method = mariabackup
wsrep_sst_auth = mariabackup:mypassword

[sst]
sst-syslog=1
```

* MariaDB Benutzer mit passenden Rechten erstellen

```mysql
root@office-ffm-db-01:[~]: mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 14
Server version: 10.4.6-MariaDB-1:10.4.6+maria~stretch mariadb.org binary distribution

MariaDB [(none)]> CREATE USER 'mariabackup'@'localhost' IDENTIFIED BY 'mypassword';

MariaDB [(none)]> GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'mariabackup'@'localhost';
```

* Fehlende Pakete installieren

```bash
$ sudo apt install socat mariadb-backup
```
Nach einem `sudo systemctl reload mariadb` wird nun wir einen SST statt `rsync` `mariabackup` verwendet.
