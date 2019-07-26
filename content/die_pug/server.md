---
title: Der PUG Server
---

## Der PUG Server

Wie bei so vielen Dingen auch, hat sich der PUG Server auch in den letzten Jahren sehr verändert. Der Server musste leider vom Netz, da die Firma ([http://www.eswe-versorgung.de/]) die ihn für viele -- dankenswerte -- Jahre beheimatet hat, ihn nicht mehr an Ort und Stelle belassen konnte. Daher wurde für die erste Zeit ein neue, virtuelle Hardware bei OVH (gesponsort von Johannes Maus) gefunden, und läuft nun beim Provider First-Colo in Frankfurt, unter der Obhut von Denny Fuchs.

### Hardware
####  Physisch
* CPU:	Intel(R) Xeon(R) CPU E3-1230 v6 @ 3.50GHz
* RAM:	32GB
* Festplatten:	1 x Zpool (Raidz) mit SSD von Crucial, 1 x Zpool (Raidz) mit WD-Red 2TB
* Ethernet:	I210 Gigabit Network Connection
* Anbindung/Hosting:	First-Colo
* Gehäuse:	https://www.supermicro.com/products/chassis/1U/819/SC819TQ-R700WB
* Mainboard:	https://www.supermicro.com/products/motherboard/Xeon/C236_C232/X11SSH-F.cfm
* Hypervisor:	Proxmox 5.x (Debian + KVM)

#### Virtuelle VM

* CPU:	KVM 3.5 Ghz
* RAM:	4096MByte
* Festplatten:	/dev/vdb mit 25GB
* Ethernet:	2 x VirtIO
* Anbindung/Hosting: [FirstColo](https://www.First-Colo.de)
* IPv6 connection provided by [FirstColo](https://www.First-Colo.de)

### Software
* Debian GNU/Linux - http://www.debian.org/
* Apache HTTP-Server - http://httpd.apache.org/
* Hugo Content Generator - https://gohugo.io/
* Bareos Backup: https://www.bareos.org/en/
