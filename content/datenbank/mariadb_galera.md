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

Als erstes fügen wir das Repository von MariaDB hinzu, da das von Debian schon etwas älter ist:


