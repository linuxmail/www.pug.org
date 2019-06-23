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

Als Vorbereitung habe ich drei Debian Stretch VMs installiert:

* 10GB LVM Festplattenplatz
* 1GB Ram

### MariaDB 10.x

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

