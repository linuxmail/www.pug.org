---
title: Icinga2 Cluster mit Puppet
toc: true
tags:
  - icinga2
  - monitoring
  - Puppet
---

# Icinga2 und Puppet

{{% toc %}}

In dieser Anleitung wird **ein** Vorschlag unterbreitet, wie ein [Icinga2](https://icinga.com/docs/icinga2/latest/doc/01-about/) Cluster in Kombination mit [Icingaweb2](https://github.com/Icinga/icingaweb2) und Puppet installiert betrieben werden kann.\

{{% alert theme="warning" %}}Diese Anleitung wird größer als gedacht und enthält noch nicht den HA Teil, der aber demnächst nachgereicht wird. Des weiteren wird er fortlaufend verbessert.{{% /alert %}}

## Zwei Konzepte

Der Autor ([ich](https://www.denny-fuchs.de)) hat so einige Zeit damit zugebracht zu schauen, was wie am Besten klappt. Es gibt zwei grundlegende Methoden, die ich kurz anreißen werden.

### Puppet Ressource Export

Bei dieser Methode ist das zentrale Bindeglied die PuppetDB.\
Bei jedem Aufruf des Puppet Agents, werden alle Fakten (facts) eingesammelt und in die PuppetDB geschrieben. Wenn diese Daten einmal vorhanden sind, kann man sie natürlich auch wieder abrufen.\
Bei einer Icinga2 Installation kann es so aussehen, dass bei einer neuen Node (VM / physisch / Container) automatisch ein neuer Icinga2 Host erstellt und überwacht wird. Dank der Apply Regeln würde man recht schnell zu einem nahezu vollautomatischen Monitoring System kommen; immer unter der Voraussetzung, dass ein Puppet Agent darauf läuft.\
Wird eine Node aus dem System entfernt (weil nicht mehr benötigt), kann sie auch aus der Überwachung verschwinden. Dafür benötigt es allerdings ein wenig Feintuning.\
Was auf dem ersten Blick sehr verlockend erscheint, offenbart bei näherer Betrachtung allerdings einige Nachteile:

Es soll ein neuer Server mit in die Überwachung aufgenommen werden. Dazu muss natürlich erst einmal Puppet und Icinga eingerichtet worden sein, sodass beim initialen Aufruf des Puppet Agents der Icinga2 Agent auf dem neuen Server eingerichtet und gestartet wird. Danach sollte der Puppet Agent ein weiteres Mal gestartet werden, sodass die Fakten in der Datenbank auch wirklich alle vorhanden sind, auch die, die mit dem Icinga2 Puppet Modul mitkommen.\
Damit auch der Icinga2 Master darüber Bescheid weiß, dass eine neue Node zum überwachen eingetroffen ist, muss auf dem Icinga2 Master ebenfalls der Puppet Agent aufgerufen werden. Dies hat zur Folge, dass das Puppet Icinga2 Modul diese neue Node in der Datenbank findet, eine passende Icinga2 Host Konfiguration erstellt und anschließend den Icinga2 Master neustartet, bzw. ein Reload durchführt.\
Im Kern bedeutet dies, dass der Puppet Agent zwei Mal aufgerufen werden muss:

1. Auf dem neuen zu überwachenden Host -> Daten in PuppetDB schreiben
2. Auf dem Icinga2 Master -> Um neue Node aus der PuppetDB zu holen 

Je nach Komplexität der Puppet Konfiguration, kann so ein Puppet Aufruf schon einmal länger dauern, erst Recht, wenn wir es nicht nur mit einer ein- oder zweistelligen Anzahl von Hosts zu tun haben.

Kompliziert wird es auch dann, wenn wir es nicht nur mit einem (logischen) Ort zu tun haben. Stichwort: DMZ / VPN / Rechenzentren.

Das gesamte Konstrukt baut darauf, dass es nur eine PuppetDB gibt und alle auf die gleiche Datenbank zugreifen. Wenn es aber mehrere Puppet Installationen gibt, bricht das Konzept. An dieser Stelle würde man zum Datenbank Spezialisten gehen und Konzepte entwickeln, um die fehlenden Informationen zu importieren.\
Für kleinere und übersichtliche Installationen ist das Konzept großartig, für alles andere eher weniger.

Wer sich dafür interessiert, findet hier die nötigen [Puppet Ideen](https://github.com/Icinga/puppet-icinga2/tree/master/examples/example4).

### Icingaweb2 + PuppetDB

Methode Zwei hat sich im Laufe der Zeit als wirklich sehr durchdachtes Konzept entpuppt. Die Aufgabe des Puppet Agent besteht nur noch aus zwei Aspekten:

1. Icinga2 Agent installieren und einrichten ([incl. Zertifikate](https://github.com/Icinga/puppet-icinga2#clientsatellite-certificates))
3. Fakten in die PuppetDB eintragen

Um nun eine weitere Icinga2 Hostkonfiguration zu erstellen, wird das [PuppetDB](https://github.com/Icinga/icingaweb2-module-puppetdb) Modul vom [Icingaweb2 Director](https://github.com/Icinga/icingaweb2-module-director) verwendet. Es greift auf die PuppetDB (Postgresql) zu, und geht wie folgt vor:

1. Extrahieren der Nodes und Fakten aus der PostgreSQL und in die eigene Datenbank überführen
2. Daten nach definierten Regeln verändern, löschen oder neue hinzufügen
3. Neue Icinga2 Hostkonfiguration erstellen und über die API an den Icinga2 Master übergeben

Alle Schritte lassen sich automatisieren, sodass auch hier am Ende eine vollautomatische Überwachung stattfinden kann.\
Der Große Unterschied zu dem vorhergehenden Konzept liegt darin, dass nur einmalige Aufrufe des Puppet Agents notwendig sind und danach nur noch bei Änderungen. Auf dem Icinga2 Master wird kein Aufruf benötigt, da die Konfiguration vom Icingaweb2 Director übernommen wird.\
Des weiteren kann das PuppetDB Modul vom Icingaweb2 Director nicht nur eine PuppetDB abfragen, sondern beliebig viele. Die einzige Voraussetzung ist, dass der Datenbank Zugriff vom Icingaweb2 auf die jeweilige Postgresql Datenbank (TCP) gestattet ist.

Der Autor hat mit der ersten Methode angefangen und aufgrund der (Netzwerk / PuppetDB) Restriktionen nach und nach Methode Zwei implementiert, welche auch heute noch so genutzt und ausgebaut wird.

## Zutaten

Wir werden Methode **Zwei** verwenden, da diese am flexibelsten ist. Für den Anfang begnügen wir uns mit dem "einfachsten" Setup, ohne DMZ und Satelliten. Dazu benötigen wir folgendes:

* Eine funktionierende Puppet Infrastruktur
  * Siehe [hier](../../verwaltung/puppet)).
* Einen MariaDB Galera Cluster
    - Siehe [hier](../../datenbanken/mariadb_galera)
* Drei Debian Buster VMs oder Container
    - Icinga2 Master 1 / 2
    - Icinga2 Agent
    - (Alternativ können natürlich gleich die DB Hosts eingebunden werden)
* Puppet Kenntnisse

Beginnen wir mir dem Icinga2 Master. Um die Tipparbeit ein wenig in Grenzen zu halten, gelten folgende Konventionen:

* Puppet Dateien werden immer als eigener Benutzer editiert und per Git auf dem Puppetmaster hinterlegt
* Ein sehr großer Teil entstammt von einem fertigen Monitoring System, daher wird vieles per Copy/Paste übernommen, sofern sinnvoll.
* Unter Umständen müssen die Puppet Agents mehrfach aufgerufen werden, aber es wird versucht dies zu vermeiden
* Manifest/Konfigurations Dateien enthalten immer einen Kopf, damit klar wird, durch welches Modul eine Veränderung vorgenommen wurde
* **Alles** wird per Git gepflegt
* Der Ort ist, wenn nicht anders definiert: `puppet/environments/dev/`
* Die Unterordner sind:
    - hieradata/ -> Alles was mit Yaml zu tun hat
    - modules/ -> Wo alle eigenen Module liegen
* Es ist nicht perfekt, aber nachvollziehbar und Anregungen nehme ich sehr sehr gern an

## Icinga2 Master

### Vorbereitung

### Apache2

Für Icingaweb2 verwenden wir den Apache2 Webserver. Damit wir auch Hiera verwenden können, gibt es einige Hilfs- Manifests.

Zuerst müssen wir noch das Puppet Modul installieren:

```bash
$ puppet module install puppetlabs-apache --modulepath production/modules/
```
Nicht vergessen Git Bescheid zu geben.

* **modules/profile/manifests/webserver/apache2.pp:**

```puppet
# source modules/profile/manifests/webserver/apache2.pp
# == Class: profile::webserver::apache2
class profile::webserver::apache2 {
  class { 'apache':
    default_vhost        => false,
  }

  $myApache2Vhosts = hiera('apache::vhost', {})
  create_resources('apache::vhost', $myApache2Vhosts)

  contain '::apache::mod::alias'
  contain '::apache::mod::env'
  contain '::apache::mod::rewrite'
  contain '::apache::mod::headers'
  contain '::apache::mod::setenvif'
  contain '::apache::mod::deflate'
  contain '::apache::mod::status'
  contain '::apache::mod::auth_basic'
  contain '::apache::mod::authn_core'
  contain '::apache::mod::authz_user'
  contain '::apache::mod::authn_file'
}
```

* **profile/manifests/webserver/apache2_php.pp**
```puppet
# source modules/profile/manifests/webserver/apache2_php.pp
# == Class: profile::webserver::apache2_php
class profile::webserver::apache2_php {
  contain ::apache::mod::php
}
```

* **modules/profile/manifests/webserver/apache2_remoteip.pp**
```puppet
# source modules/profile/manifests/webserver/apache2_remoteip.pp
# == Class: profile::webserver::apache2_remopteip
class profile::webserver::apache2_remoteip {
  contain ::apache::mod::remoteip
}
```

#### SSL

Der Zugriff sollte ausschließlich per TLS erfolgen, daher entweder Zertifikate selbst erzeugen (z.B. per [xca](https://www.hohnstaedt.de/xca/)) oder LetsEncrypt verwenden.\
Für dieses Szenario verwende ich letzteres und habe die Test VMs per **IPv6** an das Welt-Weite-Netz angebunden. Als Client verwende ich gerne [dehydrated](https://github.com/lukas2511/dehydrated).

**Damit wir bei Puppet kein Henne-Ei Problem bekommen, ist im Hiera das Snakeoil Zertifikat angegeben.**

```bash
# apt install dehydrated dehydrated-apache2
```

Wir installieren auch das Paket für Apache, welches einfach nur eine Konfiguration mitbringt. Diese wird dann im Hiera Abschnitt eingebunden.

* **/etc/dehydrated/config**

```
CONFIG_D=/etc/dehydrated/conf.d
BASEDIR=/var/lib/dehydrated
WELLKNOWN="${BASEDIR}/acme-challenges"
DOMAINS_TXT="/etc/dehydrated/domains.txt"
IP_VERSION=6
```

* **/etc/dehydrated/domains.txt**

Hier muss natürlich auch ein CNAME oder A existieren, für office-ffm-master.

```
office-ffm-master-01.4lin.net office-ffm-master.4lin.net
```

Sobald der Apache2 installiert und eingerichtet wurde, kann mittels dehydrated das korrekte Zertifikat bezogen werden. 

```bash
# dehydrated --register --accept-terms ; dehydrated -c
```

Anschließend müssen um Hiera die Pfade für das SSL Zertifikat getauscht werden.

### MariaDB

An [dieser Stelle](../datenbanken/mariadb_galera) habe ich bereits aufgezeigt, wie ein MariaDB Galera Cluster erstellt werden kann.\
Ich gehe davon aus, dass ein funktionierendes MariaDB vorhanden ist und folgende User und leere(!) Datenbanken erstellt worden sind:

* icingaweb2_director_db
* icinga2_ido_db
* icingaweb2_director_db

{{% alert theme="warning" %}}**Leider kann man kein ed25519 aktuell unter Buster verwenden, da das php-mysql 7.3 aktuell nicht unterstützt** {{% /alert %}}

```sql
root@office-ffm-db-01:# mysql
MariaDB [(none)]> CREATE DATABASE icinga2_ido_db CHARACTER SET utf8mb4;
MariaDB [(none)]> CREATE DATABASE icingaweb2_db CHARACTER SET utf8mb4;
MariaDB [(none)]> CREATE DATABASE icingaweb2_director_db CHARACTER SET utf8mb4;

MariaDB [mysql]> GRANT SELECT INSERT UPDATE DELETE DROP CREATE VIEW CREATE INDEX EXECUTE ALTER REFERENCES ON icinga2_ido_db.* TO 'icinga2_ido_db'@'192.168.1.%' IDENTIFIED BY 'secret';

MariaDB [mysql]> GRANT SELECT INSERT UPDATE DELETE DROP CREATE VIEW CREATE INDEX EXECUTE ALTER REFERENCES ON icingaweb2_db.* TO 'icingaweb2_db'@'192.168.1.%' IDENTIFIED BY 'secret';

MariaDB [mysql]> GRANT SELECT INSERT UPDATE DELETE DROP CREATE VIEW CREATE INDEX EXECUTE ALTER REFERENCES ON icingaweb2_director_db.* TO 'icingaweb2_director_db'@'192.168.1.%' IDENTIFIED BY 'secret';
```

Man kann zwar auch von Puppet die Datenbank anlegen lassen, aber dafür müsste man noch das MySQL Puppet aufnehmen, was ich mir an dieser Stelle spare.

## Icinga2 Puppet Modul

```bash
$ puppet module install icinga-icinga2 --modulepath environments/production/modules/
$ puppet module install icinga-icingaweb2 --modulepath environments/production/
$ puppet module install camptocamp-systemd --modulepath environments/production/modules/
$ git add environments/production/modules/
$ git commit -m "Add Icinga2 module and dependencies" environments/production/modules/
```

* **Struktur**

```bash
$ mkdir -p environments/dev/modules/profile/{manifests,files,templates}/icinga2/
```

### Manifests

Wir schmeißen im Grunde die Standardkonfiguration weg und erzeugen sie neu. Die Datenbank Daten entnehmen wir Hiera sowie diverse andere Parameter.

* Nagios gehört der ssl-cert Gruppe an, um SSL Zertifikate lesen zu können
* Repo wird von Puppet verwaltet
* Die Datenbank wird regelmäßig aufgeräumt
* API wird mit TLS Minimum 1.2 abgesichert (default ab Icinga 1.11)
* Standard Zone global-templates und director-global werden erstellt
* Nur der Config Master erhält Apply Rules, Commands und Co
* create\_resource for Host und Service Objekte sowie API-User und Service Gruppen
* Bei 3rdparty Plugins wird zwischen Master und generischen Agent unterschieden
* Firewall wird geöffnet für Port 5665


#### Master

* **environments/dev/modules/profile/manifests/icinga2/master.pp**

```puppet
class profile::icinga2::master (
  $icinga_db_host = hiera('monitoring::mysql::ipaddress'),
  $icinga_db_name = hiera('icinga::mysql_db'),
  $icinga_db_user = hiera('icinga::mysql_user'),
  $icinga_db_password = Sensitive(hiera('icinga::mysql_password')),
  $ticketsalt = Sensitive(hiera('icinga::api::ticketsalt')),
  Boolean $icinga_config_master = hiera('icinga::config_master'),
){

user { 'nagios': groups =>  ssl-cert } ->

package {'mariadb-client':
  ensure => present,
}

class { '::icinga2':
  manage_repo    => true,
  purge_features => false,
  plugins        => ['plugins', 'plugins-contrib', 'windows-plugins', 'nscp', 'manubulon'],
  confd          => true,
  constants      => {
    'ZoneName'   => 'NodeName',
    'TicketSalt' => $ticketsalt.unwrap,
  }
}

class { '::icinga2::pki::ca': }

class{ '::icinga2::feature::idomysql':
  user          => "${icinga_db_user}",
  password      => "${icinga_db_password.unwrap}",
  database      => "${icinga_db_name}",
  host          => "${icinga_db_host}",
  import_schema => true,
  # require     => Mysql::Db["${icinga_db_name}"],
  cleanup       => {
    downtimehistory_age => '48h',
    contactnotifications_age => '31d',
    acknowledgements_age => '31d',
    logentries_age => '31d',
    statehistory_age => '183d',
  },
  require       => Package['mariadb-client'],
}

# Feature: api
class { '::icinga2::feature::api':
  pki             => 'none',
  accept_commands => true,
  accept_config   => true,
  ssl_protocolmin =>  'TLSv1.2',
  ssl_cipher_list =>  'ALL:!ADH:RC4+RSA:+HIGH:!MEDIUM:!LOW:!SSLv2:!EXPORT',
}

# Global Zones are on all Masters
icinga2::object::zone { ['global-templates', 'director-global']:
  global => true,
  order  => '47',
}

# Only create configuration, if values set to true, because, Icinga2 uses **only** one config master
# The changed configuration is copied via API to the 2nd master
if $icinga_config_master {

  # Zone directories
  file { ['/etc/icinga2/zones.d/master',
  '/etc/icinga2/zones.d/global-templates']:
    ensure => directory,
    owner  => 'nagios',
    group  => 'nagios',
    mode   => '0750',
    tag    => 'icinga2::config::file',
    purge  => true,
  }

  file { 'icinga2_services':
    path    => '/etc/icinga2/conf.d/services',
    ensure  =>  directory,
    purge   =>   true,
    recurse =>  true,
  }

  file { 'icinga2_notifications':
    path    => '/etc/icinga2/conf.d/notifications',
    ensure  => directory,
    purge   => true,
    recurse => true,
  }

  file { 'icinga2_commands':
    path    => '/etc/icinga2/conf.d/commands',
    ensure  => directory,
    purge   => true,
    recurse => true,
  }

# Define apply rules that
contain profile::icinga2::applyrules

# Create Icinga hosts from Hiera
$myicinga2hosts = hiera('icinga2::object::host', {})
create_resources( 'icinga2::object::host', $myicinga2hosts)

# Create API users from Hiera
$myicinga2apiuser = hiera('icinga2::object::apiuser', {})
create_resources( 'icinga2::object::apiuser', $myicinga2apiuser)

# Create Icinga servicegroups from Hiera
$myicinga2servicegroup = hiera('icinga2::object::servicegroup', {})
create_resources( 'icinga2::object::servicegroup', $myicinga2servicegroup)

contain profile::icinga2::notifications
contain profile::icinga2::templates
contain profile::icinga2::checkcommands

}

contain profile::icinga2::plugins

# Purge default config
file { [
  '/etc/icinga2/conf.d/notifications.conf',
  '/etc/icinga2/conf.d/groups.conf',
  '/etc/icinga2/conf.d/satellite.conf',
  '/etc/icinga2/conf.d/services.conf',
  '/etc/icinga2/conf.d/users.conf',
  '/etc/icinga2/conf.d/app.conf',
  '/etc/icinga2/conf.d/templates.conf',
  '/etc/icinga2/conf.d/downtimes.conf',
  '/etc/icinga2/conf.d/commands.conf',
  '/etc/icinga2/conf.d/hosts.conf',
]:
  ensure =>  absent,
}

    firewall { '201 allow Icinga2 connections':
      dport  => [5665],
      proto  => tcp,
      action => accept,
    }
}
```

* **environments/dev/modules/profile/manifests/icinga2/applyrules.pp**

Die Apply Regeln werden als Dateien gepflegt, da dies sehr viel einfacher und pflegeleichter ist, als sie zu exportieren.\
Ein Auszug:

```puppet
# This class is for service checks and apply rules
class profile::icinga2::applyrules {

  $templates = '/etc/icinga2/zones.d/global-templates'
  $master_confd = '/etc/icinga2/zones.d/master/conf.d'

  file { "${templates}/applyrules.d":
    ensure => directory,
    owner  => 'nagios',
    group  => 'nagios',
    mode   => '0750',
    purge  => true
  }

  file { "${master_confd}":
    ensure => directory,
    owner  => 'nagios',
    group  => 'nagios',
    mode   => '0750',
    purge  =>  true
  }

  -> file { "${templates}/applyrules.d/service_icinga_cluster_check.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    source =>  [
      'puppet:///modules/icinga2_files/applyrules/service_check_icinga2_cluster.conf',
    ],
  }

  file { "${templates}/applyrules.d/service_check_linux_base.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    source =>  [
      'puppet:///modules/icinga2_files/applyrules/service_check_linux_base.conf',
    ],
  }

  file { "${templates}/applyrules.d/service_check_ping.conf":
    ensure => file,
    owner  => nagios,
    group  => nagios,
    source => [
      'puppet:///modules/icinga2_files/applyrules/service_check_ping.conf',
    ],
  }

  file { "${templates}/applyrules.d/service_check_apt.conf":
    ensure => file,
    owner  => nagios,
    group  => nagios,
    source => [
      'puppet:///modules/icinga2_files/applyrules/service_check_apt.conf',
    ],
  }
}
```

Die Dateien liegen in einem **anderen Ordner** und kann bei Bedarf natürlich geändert werden.\
Alle "reinen" (also nicht Puppet Dateien) Icinga2 Dateien, Plugins, Scripte und Co. landen bei mir in **modules/icinga2_checks/**

```bash
$ mkdir -p environments/dev/modules/icinga2_files/files/{commands,plugins,plugins_agent,scripts,applyrules,templates}
```
Beispiel für service_check_apt.conf:

```
# Managed by Puppet
# modules/icinga2_checks/files/applyrules/service_check_apt.conf

apply Service "apt" {
  import "generic-service"
  check_command = "custom-apt"
    if (host.name != NodeName) {
        command_endpoint = host.name
    }
  assign where host.vars.distro == "Debian"
  enable_notifications = false
  ignore where host.vars.noagent
  if (host.vars.distro_name == "stretch") {
           vars.apt_list = true
     }
}
```

Beispiel für service_check_linux_base.conf:

```
# Mangaged by puppet
# modules/icinga2_files/files/applyrules/service_check_linux_base.conf

apply Service "icinga" {
  import "generic-service"
    check_command = "icinga"
    command_endpoint = host.vars.client_endpoint
    assign where host.name == NodeName
}

apply Service "basic diskspace " for (partition in host.vars.basic_partitions) {
  import "generic-service"
    display_name = "Check basic diskspace " + partition
    check_command = "disk"
    vars.disk_partitions = partition
    command_endpoint = host.vars.client_endpoint
    assign where host.vars.basic_partitions
    if (host.vars.notification_bereitschaft) {
      vars.sections = [ "bereitschaft","sysops" ]
    }
}

apply Service "Other diskpace " for (partition in host.vars.disk_partitions) {
  import "generic-service"
    display_name = "Check diskspace " + partition
    check_command = "disk"
    vars.disk_partitions = partition
    command_endpoint = host.vars.client_endpoint
    assign where host.vars.disk_partitions
    if (host.vars.notification_bereitschaft) {
      vars.sections = [ "bereitschaft","sysops" ]
    }
}

apply Service "load" {
  import "generic-service"
    command_endpoint = host.vars.client_endpoint
    check_command = "load"
    /* Used by the ScheduledDowntime apply rule in `downtimes.conf`. */
    vars.backup_downtime = "02:00-03:00"
    assign where host.vars.os == "Linux"
    ignore where host.vars.virtual_machine_type == "lxc"
}

apply Service "cpu" {
  import "generic-service"
    command_endpoint = host.vars.client_endpoint
    check_command = "check-cpu"
    assign where host.vars.os == "Linux"
    ignore where host.vars.virtual_machine_type == "lxc"
}

apply Service "procs" {
  import "generic-service"
    command_endpoint = host.vars.client_endpoint
    check_command = "procs"
    assign where host.vars.os == "Linux"
    ignore where host.vars.virtual_machine_type == "lxc"
}

apply Service "swap" {
  import "generic-service"
    command_endpoint = host.vars.client_endpoint
    check_command = "swap"
    assign where ( host.vars.os == "Linux" && host.vars.architecture != "armv7l")
}

apply Service "memory" {
  import "generic-service"
    command_endpoint = host.vars.client_endpoint
    check_command = "mem"
    vars.mem_used = true
    vars.mem_cache = true
    vars.mem_warning = 85
    vars.mem_critical = 95
    assign where host.vars.os == "Linux"
}

apply Service "users" {
  import "generic-service"
    check_command = "users"
    command_endpoint = host.vars.client_endpoint
    assign where host.vars.os == "Linux"
}
```

* **profile/manifests/icinga2/checkcommands.pp**

Hier hinterlegen wir die Kommandos. Historisch bedingt tatsächlich als Puppet Objekte. Auch hier wäre die ÜBerlegung wert, sie als reine Icinga2 Dateien zu hinterlegen. Diese Datei ist sehr umfangreich und sollte auf jeden Fall entschlackt werden. Allerdings hilft es dem einen oder anderen anhand der Syntax zu entnehmen, wie ich etwas umgesetzt habe.

```
# Custom checkcommands and may overwrites
class profile::icinga2::checkcommands {

  $templates = '/etc/icinga2/zones.d/global-templates'
  $commands = "${templates}/commands.d"
  $master_confd = '/etc/icinga2/zones.d/master/conf.d'

  file { $commands:
    ensure =>  directory,
    owner  =>  nagios,
    group  =>  nagios,
    mode   =>  '0750',
    purge  =>   true
  }

  # Extend check_mysql_health ITL
  -> file { "${commands}/check-custom-mysql-health.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    tag    =>  'icinga2::config::file',
    source =>  [
      'puppet:///modules/icinga2_files/commands/check_custom_mysql_health.conf',
    ],
  }

  # Extend check_mongodb.py ITL
  -> file { "${commands}/check-custom-mongodb.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    tag    =>  'icinga2::config::file',
    source =>  [
      'puppet:///modules/icinga2_files/commands/check_custom_mongodb.conf',
    ],
  }

  # Extend check_squid ITL
  -> file { "${commands}/check-custom-squid.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    tag    =>  'icinga2::config::file',
    source =>  [
      'puppet:///modules/icinga2_files/commands/check_custom_squid.conf',
    ],
  }

  # Extend check_apt ITL
  -> file { "${commands}/check-custom-apt.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    tag    =>  'icinga2::config::file',
    source =>  [
      'puppet:///modules/icinga2_files/commands/check_custom_apt.conf',
    ],
  }

  # Used for other disks
  icinga2::object::checkcommand { 'check-smart':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_smart',
    ],
    arguments => {
      '-d' => '$smart_device$',
      '-i' => '$smart_interface$',
      '-b' => '$smart_bad_threshold$',
    },
    vars      => {
      'smart_device'    => '/dev/sda',
      'smart_interface' => 'scsi',
    },
    target    => "${commands}/check-smart-command.conf",
  }

  # Crucial health check
  icinga2::object::checkcommand { 'check-crucial-ssd':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_crucial',
    ],
    arguments => {
      '-d'               => '$smart_device$',
      '-m'               => '$crucial_check$',
      '-w'               => '$crucial_warning$',
      '-c'               => '$crucial_critical$',
      '-s' => { 'set_if' => '$crucial_sudo$' },
    },
    vars      => {
      'smart_device'  => '/dev/sda',
      'crucial_check' => 'health',
      'crucial_sudo'  => true,
    },
    target    => "${commands}/check-smart-command.conf",
  }

  # Used for SSDs
  icinga2::object::checkcommand { 'check-smart-attributes':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_smart_attributes/check_smart_attributes',
    ],
    arguments => {
      '-d'      => '$smart_device$',
      '-dbj'    => '$smart_dbj$',
      '-ucfgj'  => '$smart_ucfgj$',
      '-nosudo' => { 'set_if' => '$smart_nosudo$' },
      '-ap' => { 'set_if' => '$smart_allperf$' },
    },
    vars      => {
      'smart_device'  => '/dev/sda',
      'smart_dbj'     => '/usr/lib/nagios/plugins/3dparty/check_smart_attributes/check_smartdb.json',
      'smart_allperf' => 'true',
    },
    target    =>  "${commands}/check-smart-attributes-command.conf",
  }

  # Ceph Checks
  icinga2::object::checkcommand { 'check-ceph-health':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /check_ceph_health',
    ],
    arguments => {
      '-e' => '$ceph_bin$',
      '-c' => '$ceph_conf$',
      '-m' => '$ceph_mon_address$',
      '-i' => '$ceph_client_id$',
      '-n' => '$ceph_client_name$',
      '-k' => '$ceph_client_keyring$',
      '-w' => '$ceph_whitelist_health_regex$',
      '-d' => '$ceph_health_detail$',
    },
    vars      => {
      'ceph_client_id'      => 'nagios',
      'ceph_client_keyring' => '/etc/icinga2/secrets/client.nagios.keyring',
    },
    target    => "${commands}/check-ceph-command.conf",

  }

  icinga2::object::checkcommand { 'check-ceph-mon':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /check_ceph_mon',
    ],
    arguments => {
      '-e' => '$ceph_bin$',
      '-c' => '$ceph_conf$',
      '-m' => '$ceph_mon_address$',
      '-I' => '$ceph_mon_id$',
      '-i' => '$ceph_client_id$',
      '-k' => '$ceph_client_keyring$',
    },
    vars      => {
      'ceph_client_id'      => 'nagios',
      'ceph_client_keyring' => '/etc/icinga2/secrets/client.nagios.keyring',
    },
    target    => "${commands}/check-ceph-command.conf",

  }

  icinga2::object::checkcommand { 'check-ceph-osd':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /check_ceph_osd',
    ],
    arguments => {
      '-e' => '$ceph_bin$',
      '-c' => '$ceph_conf$',
      '-m' => '$ceph_mon_address$',
      '-I' => '$ceph_osd_id$',
      '-H' => '$ceph_osd_host$',
      '-i' => '$ceph_client_id$',
      '-k' => '$ceph_client_keyring$',
      '-o' => '$ceph_osd_out$',
    },
    vars      => {
      'ceph_client_id'      => 'nagios',
      'ceph_client_keyring' => '/etc/icinga2/secrets/client.nagios.keyring',
      'ceph_osd_host'       => '$address$',
    },
    target    =>  "${commands}/check-ceph-command.conf",

  }

  icinga2::object::checkcommand { 'check-ceph-rgw':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /check_ceph_rgw',
    ],
    arguments => {
      '-c' => '$ceph_conf$',
      '-e' => '$radosgw_admin_bin$',
      '-d' => '$ceph_perf_detail$',
      '-B' => '$ceph_perf_byte$',
      '-i' => '$ceph_client_id$',
    },
    vars      => {
      'radosgw_admin_bin' => '/usr/bin/radosgw-admin',
      'ceph_conf'         => '/etc/ceph/ceph.conf',
    },
    target    => "${commands}/check-ceph-command.conf",

  }

  icinga2::object::checkcommand { 'check-ceph-df':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /check_ceph_df',
    ],
    arguments => {
      '-e' => '$ceph_bin$',
      '-c' => '$ceph_conf$',
      '-m' => '$ceph_mon_address$',
      '-i' => '$ceph_client_id$',
      '-n' => '$ceph_client_name$',
      '-k' => '$ceph_client_keyring$',
      '-d' => '$ceph_pool_detail$',
      '-W' => '$ceph_pool_warning$',
      '-C' => '$ceph_pool_critical$',
    },
    vars      => {
      'ceph_client_id'      => 'nagios',
      'ceph_client_keyring' => '/etc/icinga2/secrets/client.nagios.keyring',
      'ceph_pool_warning'   => '85',
      'ceph_pool_critical'  => '95',
    },
    target    =>  "${commands}/check-ceph-command.conf",

  }

  # Check Fortigate Firewall
  icinga2::object::checkcommand { 'check-fortigate':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_fortigate',
    ],
    arguments => {
      '-H' => '$snmp_address$',
      '-v' => '$snmp_version$',
      '-U' => '$snmpv3_user$',
      '-X' => '$snmpv3_priv_key$',
      '-A' => '$snmpv3_auth_key$',
      '-x' => '$snmpv3_priv_alg$',
      '-a' => '$snmpv3_auth_alg$',
      '-w' => '$snmp_warn$',
      '-c' => '$snmp_crit$',
      '-T' => '$snmp_fortigate_checktype$',
      '-S' => '$snmp_fortigate_serial$',
      '-V' => '$snmp_fortigate_vpnmode$',
      '-M' => '$snmp_fortigate_mode$',
      '-p' => '$snmp_fortigate_store_path$',
    },
    vars      => {
      'snmp_version'              => '3',
      'snmpv3_priv_alg'           => 'aes',
      'snmpv3_auth_alg'           => 'sha',
      'snmp_fortigate_store_path' => '/var/lib/nagios/',
    },
    target    => "${commands}/check-snmp-devices-command.conf",
  }

  # Check Apache2 status
  icinga2::object::checkcommand { 'check-apache2-status':
    import    => [
      'plugin-check-command',
      'ipv4-or-ipv6',
    ],
    command   => [
      'PluginDir + /3dparty/check_apache2',
    ],
    arguments => {
      '-H' => '$apache2_status_address$',
      '-p' => '$apache2_status_port$',
      '-u' => '$apache2_status_uri$',
      '-U' => '$apache2_status_username$',
      '-P' => '$apache2_status_password$',
      '-s' => {
        set_if => '$apache2_status_ssl$',
      },
      '-N' => '$apache2_status_no_validate$',
      '-w' => '$apache2_status_warning$',
      '-c' => '$apache2_status_critical$',
    },
    vars      => {
      'apache2_status_address' => '$check_address$',
    },
    target    => "${commands}/check-http-command.conf",
  }

  icinga2::object::checkcommand {'httpsping':
    import  => [
      'http',
    ],
    command => [
      'PluginDir + /check_http',
    ],
    vars    => {
      'http_ssl' => true,
    },
    target  => "${commands}/check-http-command.conf",
  }

  # Check Windows Powershell command
  icinga2::object::checkcommand { 'check-powershell':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'C:\\\Windows\\\system32\\\WindowsPowerShell\\\v1.0\\\powershell.exe'
    ],
    arguments => {
      '-command' => {
        required => true,
        'value'  => '$ps_command$',
        order    => -1,
      },
      '-args'    => {
        'value'  => '$ps_args$',
        order    => 97,
      },
      ';exit'    => {
        'value' => '$$LASTEXITCODE',
        order   => 99,
      },
    },
    target   => "${commands}/check-windows-plugins.conf",
  }

  icinga2::object::checkcommand { 'check-powershell-x64':
    import    => [
      'plugin-check-command',
    ],

    command   => [
      'C:\\\Windows\\\sysnative\\\WindowsPowerShell\\\v1.0\\\powershell.exe'
    ],

    arguments => {
      '-args'    => {
        order    => 97,
        'value'  => '$ps_args$',
      },
      '-command' => {
        order    => -1,
        required => true,
        'value'  => '$ps_command$',
      },
      ';exit'    => {
        order   => 99,
        'value' => '$$LASTEXITCODE',
      },
    },
    target   => "${commands}/check-windows-plugins.conf",
  }

  # Check mail flow -> Send -> Receive -> check
  icinga2::object::checkcommand { 'check-imap-receive':
    import    => [
      'ipv4-or-ipv6',
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_imap_receive',
    ],
    arguments => {
      '--plugin'              => {
        'value'  => '/usr/lib/nagios/plugins/3dparty/check_imap_receive',
        skip_key => true,
        required => true,
      },
      '-w'                    => {
        'value'               => '$imap_warn$',
      },
      '-c'                    => {
        'value'               => '$imap_crit$',
      },
      '-t'                    => {
        'value'               => '$imap_timeout$',
      },
      '--imap-check-interval' => {
        'value'               => '$imap_check_intervall$',
      },
      '--imap-check-retries'  => {
        'value'               => '$imap_check_retries$',
      },
      '--ssl'                 => {
        'set_if'              => '$imap_ssl$',
      },
      '-H'                    => {
        'value'               => '$check_address$',
      },
      '--port'                => {
        'value'               => '$port$',
      },
      '--username'            => {
        'value'               => '$imap_user$',
      },
      '--password'            => {
        'value'               => '$imap_password$',
      },
      '-s'                    => {
        'value'               => 'SUBJECT',
      },
      '-s'                    => {
        'value'               => 'Icinga Test %TOKEN1%.',
      },
    },
    vars      => {
      'imap_server'          => '$check_address$',
      'imap_user'            => '$mail_username$',
      'imap_password'        => '$mail_password$',
      'imap_check_intervall' => 5,
      'imap_check_retries'   => 10,
      'imap_search_string'   => 'SUBJECT',
      'imap_warn'            => '220',
      'imap_crit'            => '230',
      'imap_ssl'             => true,
    },
    target    => "${commands}/check-imap-receive-command.conf",
  }

  icinga2::object::checkcommand { 'check-smtp-send':
    import    => [
      'ipv4-or-ipv6',
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_smtp_send',
    ],
    arguments => {
      '--plugin'   => {
        'value'  => '/usr/lib/nagios/plugins/3dparty/check_smtp_send',
        skip_key => true,
        required => true,
      },
      '-w'         => {
        'value'               => '$smtp_warn$',
      },
      '-c'         => {
        'value'               => '$smtp_crit$',
      },
      '-t'         => {
        'value'               => '$smtp_timeout$',
      },
      '--tls'      => {
        'set_if'              => '$smtp_tls$',
      },
      '-H'         => {
        'value'               => '$check_address$',
      },
      '--port'     => {
        'value'               => '$port$',
      },
      '--username' => {
        'value'               => '$imap_user$',
      },
      '--password' => {
        'value'               => '$imap_password$',
      },
      '-auth'      => {
        'value'               => '$smtp_auth_type$',
      },
      '--header'   => {
        'value'               => '$smtp_header$',
      },
      '--mailto'   => {
        'value'               => '$smtp_mailto$',
      },
      '--mailfrom' => {
        'value'               => '$smtp_mailfrom$',
      },
    },
    vars      => {
      'smtp_server'    => '$check_address$',
      'smtp_user'      => '$mail_username$',
      'smtp_password'  => '$mail_password$',
      'smtp_auth_type' => 'plain',
      'smtp_header'    => 'Subject: Icinga2 check',
      'smtp_mailto'    => 'mailcheck@example.com',
      'smtp_mailfrom'  => 'mailcheck@example.com',
      'smtp_warn'      => '220',
      'smtp_crit'      => '230',
      'smtp_tls'       => true,
    },
    target    => "${commands}/check-smtp-send-command.conf",
  }

  icinga2::object::checkcommand { 'check-email-delivery':
    import    => [
      'ipv4-or-ipv6',
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_email_delivery',
    ],
    arguments => {
      '-p'              => {
        'value'  => '/usr/lib/nagios/plugins/3dparty/check_smtp_send',
        skip_key => true,
        order    => -1,
        required => true,
      },
      '-w'              => {
        'value' => '$smtp_warn$,$imap_warn$',
        order   => 0,
      },
      '-c'              => {
        'value' => '$smtp_crit$,$imap_crit$',
        order   => 2,
      },
      '--timeout'       => {
        'value' => '$smtp_timeout$,$imap_timout$',
        order   => 3,
      },
      '--token'         => {
        'value' => '$mail_token$',
        order   => 4,
      },
      '-H'              => {
        'value' => '$mail_host$',
        order   => 5,
      },
      '--smtp-server'   => {
        'value' => '$smtp_host$',
        order   => 6,
      },
      '--mailfrom'      => {
        'value' => '$smtp_mailfrom$',
        order   => 7,
      },
      '--mailto'        => {
        'value' => '$smtp_mailto$',
        order   => 8,
      },
      '--smtp-username' => {
        'value' => '$smtp_username$',
        order   => 9,
      },
      '--smtp-password' => {
        'value' => '$smtp_password$',
        order   => 10,
      },
      '-U'              => {
        'value' => '$mail_username$',
        order   => 11,
      },
      '-P'              => {
        'value' => '$mail_password$',
        order   => 12,
      },
      '--auth'          => {
        'value' => '$smtp_authtype$',
        order   => 13,
      },
      '--notls'         => {
        'set_if' => '$notls$',
        order    => 14,
      },
      '--smtptls'       => {
        'set_if' => '$smtptls$',
        order    => 15,
      },
      '--nosmtptls'     => {
        'set_if' => '$nosmtptls$',
        order    => 16,
      },
      '--tls'           => {
        'set_if' => '$tls$',
        order    => 17,
      },
      '--port'          => {
        'value' => '$smtp_port$',
        order   => 18,
      },
      '--header'        => {
        'value'  => 'Subject: Icinga Test %TOKEN1%.',
        skip_key => false,
        order    => 19,
      },
      '--body'          => {
        'value'  => '$mail_body_text$',
        skip_key => false,
        order    => 20,
      },
      '--plugin'        => {
        'value'  => '/usr/lib/nagios/plugins/3dparty/check_imap_receive',
        skip_key => true,
        required => true,
        order    => 21,
      },
      '--imap-server'   => {
        'value' => '$imap_host$',
        order   => 22,
      },
      '--ssl'           => {
        'set_if' => '$imapssl$',
        order    => 23,
      },
      '--imap-port'     => {
        'value' => '$imap_port$',
        order   => 24,
      },
      '--imap-username' => {
        'value' => '$imap_username$',
        order   => 25,
      },
      '--imap-password' => {
        'value' => '$imap_password$',
        order   => 26,
      },
    },

    vars      => {
      'smtp_mailfrom'  => '$mail_username$',
      'smtp_mailto'    => '$mail_username$',
      'smtp_host'      => '$check_address$',
      'imap_host'      => '$check_address$',
      'smtp_warn'      => 15,
      'smtp_crit'      => 20,
      'imap_warn'      => 220,
      'imap_crit'      => 230,
      'mail_token'     => 'U-X-Y',
      'mail_body_text' => 'Icinga mail flow check.',
      'nosmtptls'      => true,
      'imapssl'        => true,
    },
    target    =>  "${commands}/check-email-delivery-command.conf",
  }

  # Check with iftraffic
  icinga2::object::checkcommand { 'check-iftraffic-nrpe':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_iftraffic_nrpe',
    ],
    arguments => {
      '-i' => '$iftraffic_nrpe_include_regex$',
      '-X' => '$iftraffic_nrpe_exclude_regex$',
      '-p' => {
        'set_if' => '$iftraffic_nrpe_enable_perf$',
      },
      '-P' => '$iftraffic_nrpe_bits_s$',
      '-b' => '$iftraffic_nrpe_brief_output$',
      '-I' => '$iftraffic_nrpe_must_include$',
      '-a' => '$iftraffic_nrpe_semi_automatic$',
      '-d' => '$iftraffic_nrpe_must_exclude$',
      '-B' => '$iftraffic_nrpe_bits_s_output$',
      '-s' => '$iftraffic_nrpe_interface_speed$',
      '-w' => '$iftraffic_nrpe_warning$',
      '-c' => '$iftraffic_nrpe_critical$',
      '-u' => '$iftraffic_nrpe_unknown_to_warning$',
    },
    vars      => {
      'iftraffic_nrpe_enable_perf'     => true,
      'iftraffic_nrpe_interface_speed' => 1000,
    },
    target    => "${commands}/check-iftraffic-nrpe-command.conf",
  }

  # Check TCP connections
  icinga2::object::checkcommand { 'check-connections':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_connections',
    ],
    arguments => {
      '-s' => '$connections_state$',
      '-f' => '$connections_filter$',
      '-p' => '$connections_protocol_family$',
      '-w' => '$connections_warning$',
      '-c' => '$connections_critical$',
      '-e' => '$connections_exact_count$',
    },
    vars      => {
      'connections_state'    => 'established',
      'connections_warning'  => 800,
      'connections_critical' => 1200,
    },
    target    => "${commands}/check-connections-command.conf",
  }

  # Check Password expire
  icinga2::object::checkcommand { 'check-password-expire':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_password_expire',
    ],
    arguments => {
      '-x' => '$password_exclude$',
      '-w' => '$password_warning$',
      '-c' => '$password_critical$',
    },
    vars      => {
      'password_warning'  => 14,
      'password_critical' => 7,
    },
    target    => "${commands}/password-expire.conf",
  }

  icinga2::object::checkcommand { 'check-cpu':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_cpu',
    ],
    arguments => {
      '-w'  => '$cpu_global_percent_warning$',
      '-uw' => '$cpu_user_percent_warning$',
      '-iw' => '$cpu_iowait_percent_warning$',
      '-sw' => '$cpu_system_percent_warning$',
      '-c'  => '$cpu_global_percent_critical$',
      '-uc' => '$cpu_user_percent_critical$',
      '-ic' => '$cpu_iowait_percent_critical$',
      '-sc' => '$cpu_system_percent_critical$',
      '-i'  => '$cpu_iostat_interval_seconds$',
      '-n'  => '$cpu_iostat_number$',
    },
    target    => "${commands}/check-cpu.conf",
  }

  # Check Apach2 processes
  icinga2::object::checkcommand { 'check-apache-process':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /check_procs',
    ],
    arguments => {
      '-w' => '$apache_procs_warning$',
      '-c' => '$apache_procs_critical$',
      '-a' => '$apache_procs_argument$',
    },
    vars      => {
      'check_interval'        => 300,
      'apache_procs_warning'  => '1:15',
      'apache_procs_critical' => '1:20',
      'apache_procs_argument' => '/usr/sbin/apache2 -k start',
    },
    target    => "${commands}/check-http-command.conf",
  }

  # Check HTTP flow with HolyGhost  (phantomjs)
  icinga2::object::checkcommand { 'check-holyghost':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_holyghost/HolyGhost',
    ],
    arguments => {
      '--param' => '$holyghost_options$',
      '--test' => '$holyghost_testcase$',
      '--path' => '$holyghost_path$',
      '--proxy' => '$holyghost_proxy$',
      '--protocol' => '$holyghost_ssl_protocol$',
      '--html' => {
        set_if => '$holyghost_print_html$',
      },
      '--delete' => {
        set_if => '$holyghost_cleanup$',
      },
      '--keep' => {
        set_if => '$holyghost_keep$',
      },
    },
    vars      => {
      'check_interval'    => 300,
      'holyghost_path' => '/var/tmp',
    },
    target    => "${commands}/check-http-command.conf",
  }

  # Check APC UPS
  icinga2::object::checkcommand { 'check-ups-health':
    import    => [
      'ipv4-or-ipv6',
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_ups_health',
    ],
    arguments => {
      '-H'                 => '$ups_health_address$',
      '-t'                 => '$ups_health_timeout$',
      '--domain'           => '$ups_health_domain$',
      '--protocol'         => '$ups_health_protocol$',
      '--community'        => '$ups_health_community$',
      '--username'         => '$ups_health_username$',
      '--authpassword'     => '$ups_health_authpassword$',
      '--authprotocol'     => '$ups_health_authprotocol$',
      '--privpassword'     => '$ups_health_privpassword$,',
      '--privprotocol'     => '$ups_health_privprotocol$,',
      '--contextengineid'  => '$ups_health_contextengineid$,',
      '--contextname'      => '$ups_health_contextname$,',
      '--community2'       => '$ups_health_community2$,',
      '--servertype'       => '$ups_health_servertype$,',
      '--oids'             => '$ups_health_oids$',
      '--offline'          => '$ups_health_offline$',
      '--mode'             => '$ups_health_mode$',
      '--regexp'           => '$ups_health_regexp$',
      '--warning'          => '$ups_health_warning$',
      '--critical'         => '$ups_health_critical$',
      '--warningx'         => '$ups_health_warningx$',
      '--criticalx'        => '$ups_health_criticalx$',
      '--units'            => '$ups_health_units$',
      '--name'             => '$ups_health_name$',
      '--name2'            => '$ups_health_name2$',
      '--name3'            => '$ups_health_name3$',
      '--extra-opts'       => '$ups_health_extra-opts$',
      '--blacklist'        => '$ups_health_blacklist$',
      '--mitigation'       => {
        set_if => '$ups_health_mitigation$',
      },
      '--lookback'         => '$ups_health_lookback$',
      '--environment'      => '$ups_health_environment$',
      '--negate'           => '$ups_health_negate$',
      '--morphmessage'     => '$ups_health_morphmessage$',
      '--morphperfdata'    => '$ups_health_morphperfdata$',
      '--selectedperfdata' => '$ups_health_selectedperfdata$',
      '--report'           => '$ups_health_report$',
      '--statefilesdir'    => '$ups_health_statefilesdir$',
      '--isvalidtime'      => '$ups_health_isvalidtime$',
    },
    vars      =>  {
      'ups_health_address' => '$hostaddress$',
    },
    target    => "${commands}/check-ups-health-command.conf",
  }

  # Check MySQL Perf
  icinga2::object::checkcommand { 'check-mysql-perf':
    import    => [
      'ipv4-or-ipv6',
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/mysql/perf_mysql.pl',
    ],
    arguments => {
      '--host'     => '$mysql_perf_hostname$',
      '--user'     => '$mysql_username$',
      '--password' => '$mysql_password$',
      '--output'   => '$mysql_perf_output$',
      '--socket'   => {
        set_if => '$mysql_socket$',
      },
      '--port'     => {
        set_if => '$mysql_port$',
      },
      '--module'   => '$mysql_perf_module$',
    },
    vars      =>  {
      'mysql_perf_hostname' => '$hostaddress$',
      'mysql_perf_output'   => 'icinga',
    },
    target    => "${commands}/check-mysql-perf-command.conf",
  }

  # Check MySQL Galery state
  icinga2::object::checkcommand { 'check-mysql-galera':
    import    => [
      'ipv4-or-ipv6',
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/mysql/check_galera_nodes.pl',
    ],
    arguments => {
      '--hostname' => '$mysql_galera_hostname$',
      '--user'     => '$mysql_username$',
      '--password' => '$mysql_password$',
      '--nodes'    => '$mysql_galera_nodes$',
      '--socket'   => {
        set_if => '$mysql_perf_socket$',
      },
      '--port'     => {
        set_if => '$mysql_port$',
      },
    },
    vars      =>  {
      'mysql_perf_hostname' => '$hostaddress$',
    },
    target    => "${commands}/check-mysql-galera-command.conf",
  }

  # Used for APC USV
  icinga2::object::checkcommand { 'check-usv-apc':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_snmp_usv',
    ],
    arguments => {
      '-H' => '$snmp_address$',
      '-v' => '$snmp_version$',
      '-C' => '$snmp_community$',
      '-u' => '$snmpv3_user$',
      '-l' => '$snmpv3_seclevel$',
      '-x' => '$snmpv3_priv_alg$',
      '-X' => '$snmpv3_priv_key$',
      '-a' => '$snmpv3_auth_alg$',
      '-A' => '$snmpv3_auth_key$',
      '-t' => '$snmp_check_type$',
      '-w' => '$snmp_usv_warn$',
      '-c' => '$snmp_usv_crit$',
      '-S' => '$snmp_usv_separator$',
    },
    vars      => {
      'snmpv3_seclevel' => 'authPriv',
      'snmpv3_priv_alg' => 'AES',
      'snmpv3_auth_alg' => 'SHA',
      'snmp_check_type' => 'status',
      'snmp_version'    => '3',
      'snmp_address'    => '$address$',
    },
    target    =>  "${commands}/check-snmp-devices-command.conf",
  }

  # Check APC transfer switch
  # Used for APC ATS
  icinga2::object::checkcommand { 'check-ats-apc':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_apc_ats',
    ],
    arguments => {
      '-H' => '$snmp_address$',
      '-u' => '$snmpv3_user$',
      '-l' => '$snmpv3_seclevel$',
      '-x' => '$snmpv3_priv_alg$',
      '-X' => '$snmpv3_priv_key$',
      '-a' => '$snmpv3_auth_alg$',
      '-A' => '$snmpv3_auth_key$',
      '-t' => '$snmp_check_type$',
      '-w' => '$snmp_ats_warn$',
      '-c' => '$snmp_ats_crit$',
      '-S' => '$snmp_ats_selected_source$',
    },
    vars      => {
      'snmpv3_seclevel' => 'authPriv',
      'snmpv3_priv_alg' => 'SHA',
      'snmpv3_auth_alg' => 'MD5',
      'snmp_check_type' => 'status',
      'snmp_address'    => '$address$',
    },
    target    =>  "${commands}/check-snmp-devices-command.conf",
  }

  # Used for Quagga BGPD
  icinga2::object::checkcommand { 'check-quagga-bgpd':
    import  => [
      'plugin-check-command',
    ],
    command => [
      'PluginDir + /3dparty/check_quagga_bgpd',
    ],
    target  =>  "${commands}/check-quagga-command.conf",
  }

  # Used for Quagga route
  icinga2::object::checkcommand { 'check-quagga-route':
    import  => [
      'plugin-check-command',
    ],
    command => [
      'PluginDir + /3dparty/check_quagga_route',
    ],
    target  =>  "${commands}/check-quagga-command.conf",
  }

  # Used for Synology
  icinga2::object::checkcommand { 'check-snmp-synology':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_snmp_synology',
    ],
    arguments => {
      '-h' => '$snmp_address$',
      '-u' => '$snmpv3_user$',
      '-p' => '$snmpv3_auth_key$',
      '-v' => '$snmp_synology_verbose$',
      '-w' => '$snmp_synology_storage_warning$',
      '-c' => '$snmp_synology_storage_critical$',
      '-W' => '$snmp_synology_temp_warning$',
      '-C' => '$snmp_synology_temp_critical$',
      '-i' => {
        set_if => '$snmp_synology_ignore_dsm_updates$',
      },
      '-v' => {
        set_if => '$snmp_synology_verbose_output$',
      },
    },
    vars      => {
      'snmp_address' => '$address$',
    },
    target    => "${commands}/check-snmp-devices-command.conf",
  }

  # Used for Crypto vault
  icinga2::object::checkcommand { 'check-crypto-vault':
    import  => [
      'plugin-check-command',
    ],
    command => [
      'PluginDir + /3dparty/check_crypto_vault',
    ],
    arguments => {
      '-H' => '$crypto_address$',
      '-u' => '$crypto_user$',
      '-p' => '$crypto_password$',
      '-f' => '$crypto_password_file$',
      },
      vars      => {
        'crypto_address' => '$address$',
        'crypto_user' => 'cryptotest',
        'crypto_password' => 'cryptotest',
      },

      target  =>  "${commands}/check-crypto-vault-command.conf",
  }

  # Used for ZFS check
  icinga2::object::checkcommand { 'check-zfs-linux':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_zfs_linux',
    ],
    arguments => {
      '--pool'         => {
        'value'  => '$zfs_pool$',
        order  => '1',
        skip_key => true,
        required => true,
      },
      '--capacity' =>  {
        set_if => '$zfs_pool_usage_warning$',
        order  => '2',
      },
      '--warning'  => {
        'value'  => '$zfs_pool_usage_warning$',
        skip_key => true,
        required => true,
        order    => 3,
      },
      '--critical' => {
        set_if   => '$zfs_pool_usage_warning$',
        'value'  => '$zfs_pool_usage_critical$',
        skip_key => true,
        required => true,
        order    => 4,
      },
    },
    vars      => {
      'zfs_pool'                => 'rpool',
      'zfs_pool_usage_warning'  => '80',
      'zfs_pool_usage_critical' => '90',
    },
    target    => "${commands}/check-zfs-linux-command.conf",
  }

  # Used for checking open files
  icinga2::object::checkcommand { 'check-open-fds':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_open_fds',
    ],
    arguments => {
      '-w' => '$open_files_warning$',
      '-c' => '$open_files_critical$',
      '-W' => '$open_files_percent_warning$',
      '-C' => '$open_files_percent_critical$',
    },
    target    => "${commands}/check-open-fds-command.conf",
  }

  # Mellanox SNMPv3 Check
  icinga2::object::checkcommand { 'check-mellanox':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_snmp_mlnx',
    ],
    arguments => {
      '-H' => '$snmp_address$',
      '-U' => '$snmpv3_user$',
      '-C' => '$snmp_community$',
      '-X' => '$snmpv3_priv_key$',
      '-A' => '$snmpv3_auth_key$',
      '-x' => '$snmpv3_priv_alg$',
      '-a' => '$snmpv3_auth_alg$',
      '-e' => '$snmp_mlnx_checktype$',
      '-P' => '$snmp_version$',
      '-v' => '$snmp_mlnx_debug$',
    },
    vars      => {
      'snmp_version'    => '3',
      'snmpv3_priv_alg' => 'AES',
      'snmpv3_auth_alg'  => 'SHA',
      'snmp_address'  => '$address$',
    },
    target    => "${commands}/check-snmp-devices-command.conf",
  }

  # Raspberry Pi external GPIO Sensor
  icinga2::object::checkcommand { 'check-rasbi-sensor':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_rasbi/check_rasbi_sensor.py',
    ],
    arguments => {
      '-s' => '$rasbi_sensor$',
      '-c' => '$rasbi_critical$',
      '-w' => '$rasbi_warning$',
      '-t' => '$rasbi_sensor_type$',
      '-n' => '$rasbi_sensor_name$',
    },
    target    => "${commands}/check-rasbi-command.conf",
  }

  # Add Consul checkcommand for the services
  icinga2::object::checkcommand { 'check-consul-service':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_consul_service',
    ],
    arguments => {
      '-t' => {
        set_if   => '$consul_token$',
        'value'  => '$consul_token$',
      },
      '-H' => '$consul_address$',
      '-p' => '$consul_port$',
      '-w' => '$consul_warning$',
      '-c' => '$consul_critical$',
      '-s' => {
        set_if => '$consul_tls$',
      },
      '-d' => {
        set_if => '$consul_datacenter$',
      },
      '--service'          => {
        'value'  => '$consul_service$',
        skip_key => true,
      },
    },
    vars      => {
      'consul_address' => '$address$',
      'consul_port'    => '8500',
    },
    target    => "${commands}/check-consul-command.conf",
  }
  # Add Redis Sentinel check
  icinga2::object::checkcommand { 'check-sentinel-health-service':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_sentinel_master_health',
    ],
    arguments => {
      '-H' => '$sentinel_address$',
      '-p' => '$sentinel_port$',
      '-c' => '$sentinel_critical$',
      '-w' => '$sentinel_warning$',
      '-m' => '$sentinel_master$',
    },
    vars      => {
      'sentinel_address' => '$address$',
      'sentinel_port'    => '17700',
      'sentinel_master'    => 'mymaster',
    },
    target    => "${commands}/check-redis-command.conf",
  }

  # SNMP check for Allnet, because check_snmp get string, intead of needed integer
  icinga2::object::checkcommand { 'check-snmp-allnet':
    import    => [
      'snmp',
    ],
    command   => [
      'PluginDir + /3dparty/check_snmp_allnet',
    ],
    arguments => {
      '-H' => '$snmp_address$',
      '-C' => '$snmp_community$',
      '-c' => '$snmp_crit$',
      '-w' => '$snmp_warn$',
      '-o' => '$snmp_oid$',
    },
    vars      => {
      'snmp_address' => '$address$',
      'snmp_community' => '$snmp_community$',
      'snmp_oid' => '$snmp_oid$',
    },
    target    => "${commands}/check-snmp-devices-command.conf",
  }

  # Check rack door 
  icinga2::object::checkcommand { 'check-fcdoor':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_fcdoor.pl',
    ],
    arguments => {
      '-U' => '$fcdoor_url$',
      '-t' => '$fcdoor_timeout$',
    },
    target    => "${commands}/check-http-command.conf",
  }

  # Add Generic command
  icinga2::object::checkcommand { 'check-generic':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_generic',
    ],
    arguments => {
      '-e'              => {
        'value' => "\$generic_execute_cmd$",
      },
      '-u' => {
        'value' => "\$generic_unknown_expr$",
      },
      '-w' => {
        'value' => "\$generic_warning_expr$",
      },
      '-c' => {
        'value' => "\$generic_critical_expr$",
      },
      '-o' => {
        'value' => "\$generic_ok_expr$",
      },
      '-f' => {
        'value' => "\$generic_false_expr$",
      },
      '-y' => '$generic_data_type$',
      '-i' => '$generic_ignore_rc$',
      '-n' => '$generic_name$',
      '-t' => '$generic_timeout$',
      '-d' => '$generic_tmpdir$',
      '-v' => '$generic_verbose$',
    },
    vars      => {
      'generic_tmpdir' => '/tmp/',
    },
    target    => "${commands}/check-generic-command.conf",
  }

  # Check with JSON
  icinga2::object::checkcommand { 'check-json':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_json.py',
    ],
    arguments => {
      '-u'  => '$json_url$',
      '-w'  => '$json_warning$',
      '-c'  => '$json_critical$',
      '-f'  => '$json_function$',
      '-R'  => '$json_regex$',
      '-tf' => '$json_timeformat$',
      '-td' => '$json_timeduration$',
      '-tz' => '$json_timezone$',
      '-m'  => '$json_match$',
      '-k'  => {
        'value' => '$json_key$',
      },
      '-H'  => {
        'value'    => '$json_header$',
        repeat_key => true,
      },
      '-fs' => {
        'value' => '$json_secret$',
      },
    },
    target    => "${commands}/check-http-command.conf",
  }

  # Check network bonding
  icinga2::object::checkcommand { 'check-linux-bonding':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_linux_bonding',
    ],
    arguments => {
      '--verbose'       => '$bonding_verbose$',
      '--slave-down'    => '$bonding_slave_down_state$',
      '--ignore-num-ad' => '$bonding_ignore_num_ad$',
      '--blacklist'     => '$bonding_blacklist$',
      '--no-bonding'    => '$bonding_no_bonding_state$',
    },
    target    => "${commands}/check-linux-bonding.conf",
  }

  # Check with SNMP interfaces
  icinga2::object::checkcommand { 'check-snmp-interfaces':
    import    => [
      'plugin-check-command',
    ],
    command   => [
      'PluginDir + /3dparty/check_interfaces',
    ],
    arguments => {
      '-h' => '$snmp_address$',
      '-u' => '$snmpv3_user$',
      '-K' => '$snmpv3_priv_key$',
      '-k' => '$snmpv3_priv_alg$',
      '-J' => '$snmpv3_auth_key$',
      '-j' => '$snmpv3_auth_alg$',
      '-m' => '$snmp_interface_mode$',
      '-A' => '$snmp_interface_match_alias$',
      '-b' => '$snmp_interface_bandwidth$',
      '-r' => '$snmp_interface_regex_include$',
      '-R' => '$snmp_interface_regex_exclude$',
      '-N' => { 'set_if' => '$snmp_interface_ifnames$' },
      '-d' => { 'set_if' => '$snmp_interface_down_ok$' },
      '-a' => { 'set_if' => '$snmp_interface_getalias$' },
    },
    vars      => {
      'snmp_privprotocol'      => 'AES',
      'snmp_authprotocol'      => 'SHA',
      'snmp_interface_ifnames' => 'true',
      'snmp_address'           => '$address$',
      'snmp_interface_down_ok' => true,
    },
    target    => "${commands}/check-snmp-devices-command.conf",
  }
}
```

* **profile/manifests/icinga2/notifications.pp**

Dieses Manifest trägt eigentlich einen falschen Namen. Es sorgt eigentlich nur für die Umgebung und hinterlegt Passwörter oder dass benötigte Pakete auf dem System sind, um bestimmte Tools für die Benachrichtigung verwenden zu können.

```
# Notifications scripts and passwords
class profile::icinga2::notifications (
  $domain =  hiera('monitoring::domain'),
){

  $templates = '/etc/icinga2/zones.d/global-templates'
  
  file { '/etc/icinga2/scripts':
    ensure  =>  directory,
    mode    =>  '0755',
    owner   =>  'root',
    group   =>  'nagios',
    source  =>  [
      'puppet:///modules/icinga2_files/scripts',
    ],
    recurse =>  true,
  }
}
```

* **profile/manifests/icinga2/templates.pp**

Hier werden nur die Templates hinterlegt, aber als reguläre Dateien:

```
# Mostly all templates for import
class profile::icinga2::templates {

  $global_templates = '/etc/icinga2/zones.d/global-templates'
  $templates = "${global_templates}/templates.d"

  file { "${global_templates}/templates.d":
    ensure => directory,
    owner  => 'nagios',
    group  => 'nagios',
    mode   => '0750',
    purge  => true,
    force  => true,
  }

  -> file { "${templates}/host-templates.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    tag    =>  'icinga2::config::file',
    source =>  [
      'puppet:///modules/icinga2_files/templates/host-templates.conf',
    ],
  }

  -> file { "${templates}/service-templates.conf":
    ensure =>  file,
    owner  =>  nagios,
    group  =>  nagios,
    tag    =>  'icinga2::config::file',
    source =>  [
      'puppet:///modules/icinga2_files/templates/service-templates.conf',
    ],
  }
}
```

#### Plugins

In den Foren wird oft gefragt, ob es nicht möglich wäre die Plugins per Icinga2 zu verteilen. Das geht natürlich nicht und würde den Code nur aufblasen, zumal es in sehr vielen Fällen ohnehin quatsch wäre. Bei vielen Plugins werden sehr oft weitere Pakete benötigt, die dann nach wie vor fehlen würden. Daher überlässt man das besser anderen.\
Dieses Manifest unterscheidet im Grunde drei Typen:

1. Icinga2 Agent oder Master
2. Virtueller Host / Container
3. Physischer Host

In meinem Fall bekommt der Master ein anderes Plugin Verzeichnis, als der Agent, weil zum Beispiel die IBM oder VMware Ordner auf den Clients nicht benötigt werden. Aber auch hier wäre noch sehr viel Luft zum optimieren. Da habe ich mir jetzt nicht soviel Mühe gegeben :-)

* **profile/manifests/icinga2/plugins.pp**

```
# Install the Icinga monitoring plugins
class profile::icinga2::plugins (
){
  # Sudo template
  sudo::conf { 'nagios':
    priority => 10,
    content  =>  template('profile/icinga2/sudoers_nagios.erb'),
  }

  if ($::kernel == 'linux') and ($::role != master)   {

    $linuxdeps = [
      'monitoring-plugins-standard',
      'nagios-plugins-contrib',
      'libmonitoring-plugin-perl',
      'libcrypt-des-perl',
      'nagios-snmp-plugins',
    ]
    package { $linuxdeps:
      install_options => ['--no-install-recommends'],
    }
  }

  # For Ceph
  if 'ceph' in $::role {
    $cephdeps = [
      'nagios-plugins-ceph',
    ]

    # For MongoDB
    if 'mongodb' in $::role {
      package { 'python-pymongo':
        ensure          => installed,
        install_options => ['--no-install-recommends'],
      }
    }

    # For NSLAN -> required for quagga check
    if 'nslan' in $::role {
      package { 'libnet-telnet-perl':
        ensure => installed,
        install_options => ['--no-install-recommends'],
      }
    }

    # Needed: ceph.conf group is www-data on Proxmox ceph mon node.
    user { 'nagios': groups => www-data }

    package { $cephdeps:
      install_options => ['--no-install-recommends'],
    }

    # Key on ceph generated with: ceph auth get-or-create client.nagios mon 'allow r' > client.nagios.keyring
    $ceph_nagios_key = Sensitive(hiera('ceph_nagios_keyring'))
    file {'/etc/icinga2/secrets/client.nagios.keyring':
      ensure  => file,
      mode    => '0640',
      owner   => 'root',
      group   => 'nagios',
      content =>  "[client.nagios]\n\tkey = ${ceph_nagios_key.unwrap}\n",
      require =>  File['/etc/icinga2/secrets'],
    }
  }

  file {'/etc/icinga2/secrets':
    ensure  => directory,
    mode    => '0750',
    owner   => 'root',
    group   => 'nagios',
  }

  if $::role == 'master' {
    $plugin_source = "puppet:///modules/icinga2_files/plugins"
  } else {
    $plugin_source = "puppet:///modules/icinga2_files/plugins_agent"
  }

  file { '/usr/lib/nagios/plugins/3dparty':
    ensure    => directory,
    mode      => '0755',
    owner     => 'root',
    group     => 'root',
    force     => true,
    show_diff => false,
    source    => [
      $plugin_source,
    ],
    recurse   => true,
    require   => Package['monitoring-plugins-standard'],
  }

  # Workarounds for some checks, that are not included, but in 3dparty/ folder
  file { '/usr/lib/nagios/plugins/check_iostat':
    ensure  =>   link,
    target  =>   '/usr/lib/nagios/plugins/3dparty/check_iostat',
    require =>   File['/usr/lib/nagios/plugins/3dparty'],
  }

  file { '/usr/lib/nagios/plugins/check_mem.pl':
    ensure  =>   link,
    target  =>   '/usr/lib/nagios/plugins/3dparty/check_mem',
    require =>   File['/usr/lib/nagios/plugins/3dparty'],
  }

  file { '/usr/lib/nagios/plugins/check_iostats':
    ensure  =>   link,
    target  =>   '/usr/lib/nagios/plugins/3dparty/check_iostats',
    require =>   File['/usr/lib/nagios/plugins/3dparty'],
  }

  file { '/usr/lib/nagios/plugins/check_mysql_health':
    ensure  =>   link,
    target  =>   '/usr/lib/nagios/plugins/3dparty/mysql/check_mysql_health.pl',
    require =>   File['/usr/lib/nagios/plugins/3dparty'],
  }

  file { '/usr/lib/nagios/plugins/check_nginx_status.pl':
    ensure  =>   link,
    target  =>   '/usr/lib/nagios/plugins/3dparty/check_nginx_status.pl',
    require =>   File['/usr/lib/nagios/plugins/3dparty'],
  }

  # Install perl modules for check_ngi
  if ('nginx' in $::puppet_classes) or
  ('apache' in $::puppet_classes) {
    $webdeps = [
      'libwww-perl',
    ]
    package { $webdeps:
      ensure          => installed,
      install_options => ['--no-install-recommends'],
    }
  }

  # Install IPMI tools for physical hosts
  unless ( $facts['is_virtual'] == true ) {
    $hwdeps = [ 'libipc-run-perl','freeipmi-tools','libconfig-json-perl' ]
    package { $hwdeps:
      ensure          => installed,
      install_options => ['--no-install-recommends'],
    }
  }

  case $::role {
    'master': {
      $mondeps = [
        'libnet-snmp-perl',
        'libcrypt-hcesha-perl',
        'libcrypt-des-perl',
        'libdigest-hmac-perl',
        'libcrypt-rijndael-perl',
        'libxml-simple-perl',
        'libconfig-json-perl',
        'libredis-perl',
        'nagios-snmp-plugins',
        'libhttp-date-perl',
        'liburi-perl',
        'libxml-libxml-perl',
        'libtest-lwp-useragent-perl',
        'libtime-duration-perl',
        'libcrypt-ssleay-perl',
        'openjdk-11-jre-headless',
        'liblist-compare-perl',
        'monitoring-plugins-standard',
        'icinga2-bin',
      ]
      package { $mondeps:
        install_options => ['--no-install-recommends'],
      }

      file { '/usr/local/lib/site_perl':
        ensure => directory,
        mode   => '0755',
        owner  => 'root',
        group  => 'root',
      }

      file { '/usr/lib/nagios/plugins/contrib':
        ensure => directory,
        mode   => '0755',
        owner  => 'root',
        group  => 'root',
      }

      file {'/usr/lib/nagios/plugins/check_icmp':
        ensure => file,
        mode   => '4755',
        owner  => 'root',
        group  => 'root',
      }

      file {'/usr/lib/nagios/plugins/check_ping':
        ensure => file,
        mode   => '4755',
        owner  => 'root',
        group  => 'root',
      }
    }
    default: { }
  }
}
```

##### Sudo

Da wir an sehr vielen Stellen sudo benötigen, hinterlegen wir eine passende Konfiguration. Das Puppet saz-sudo Modul sorgt dafür, dass nur syntaktisch korrekte Sudo Dateien hinterlegt werden.

* **modules/profile/templates/icinga2/sudoers_nagios.erb**

```puppet
# This file is managed by Puppet; changes may be overwritten
# Source modules/profile/templates/icinga2/sudoers_nagios.erb
#
Cmnd_Alias   NAGIOS_MSECLI = /usr/local/sbin/msecli_64
Cmnd_Alias   NAGIOS_IPMI = /usr/sbin/ipmi-sel, /usr/sbin/ipmi-sensors
Cmnd_Alias   NAGIOS_ZFS = /sbin/zpool, /sbin/zfs
Cmnd_Alias   NAGIOS_SMART = /usr/sbin/smartctl
Cmnd_Alias   NAGIOS_PWC = /usr/bin/chage
<% if @fqdn =~ /-aicint/ -%>
Cmnd_Alias   NAGIOS_C = NAGIOS_MSECLI, NAGIOS_IPMI, NAGIOS_ZFS, NAGIOS_SMART, NAGIOS_PWC, NAGIOS_CRON
<% else -%>
Cmnd_Alias   NAGIOS_C = NAGIOS_MSECLI, NAGIOS_IPMI, NAGIOS_ZFS, NAGIOS_SMART, NAGIOS_PWC
<% end -%>
Host_Alias   NAGIOS_H = <%= @fqdn %>

nagios NAGIOS_H = (ALL) NOPASSWD: NAGIOS_C
```

#### Agent

Beim Agent wurde am meisten gegrübelt, da die Anforderungen sich stetig änderten. Anfangs konnte der Master alle Agents direkt erreichen, dann kamen unterschiedliche Zonen sowie Satelliten hinzu. Am Ende bekam ein Satellit sogar einen weiteren Satelliten zugeordnet, der einen nochmaligen Umbau nach sich zog.\

![Satellite Aufbau](/doc/uploads/monitoring/monitoring_satellite.svg)

* *mon* = Monitoring Master
* *monproxy* = Monitoring Satellit
* *jumper* = Eigentlich ein Jumphost, aber dieser ist auch für die Zone MGMT zuständig. **Im Manifest ist dies jumper-02**
* **profile/manifests/icinga2/agent.pp**

```puppet
# For all non-master hosts
# source: profile/manifests/icinga2/agent.pp

class profile::icinga2::agent(
  $zones = hiera("profile::icinga2::agent::zones", {}),
  Hash $parent_endpoints,
  $apiuser_name = false,
  $apiuser_password = false,
  $ticketsalt = Sensitive(hiera('icinga::api::ticketsalt')),
  String $parent = master,
  Array $features = ['mainlog'],
  Stdlib::Compat::Ip_address $agent_ip = $::ipaddress,
) {

  # Ignore everything, if we are master
  if ($::role != master)   {

    # Include Icinga class
    class { '::icinga2':
      manage_repo => true,
      confd       => false,
      features    => $features,
      plugins        => ['plugins', 'plugins-contrib', 'manubulon'],
      require     => Exec['apt_update'],
    }

    validate_hash($parent_endpoints)
    # $parent_endpoints is Hash and we want only the first back, which is master or a DC :satellite
    # Feature: api
    if $facts['hostname'] =~ /.*-srv-monproxy.*/ or $facts['hostname'] =~ /.*-jumper-02.*/  {

      class { '::icinga2::feature::api':
        ca_host         => $parent_endpoints.keys[0],
        pki             => 'icinga2',
        ticket_salt     => $ticketsalt.unwrap,
        accept_config   => true,
        accept_commands => true,
        ssl_protocolmin => 'TLSv1.2',
        ssl_cipher_list => 'ALL:!ADH:RC4+RSA:+HIGH:!MEDIUM:!LOW:!SSLv2:!EXPORT',
        zones           => {},
      }
    } else {

      class { '::icinga2::feature::api':
        ca_host         => $parent_endpoints.keys[0],
        pki             => 'icinga2',
        ticket_salt     => $ticketsalt.unwrap,
        accept_config   => true,
        accept_commands => true,
        ssl_protocolmin => 'TLSv1.2',
        ssl_cipher_list => 'ALL:!ADH:RC4+RSA:+HIGH:!MEDIUM:!LOW:!SSLv2:!EXPORT',
        zones           => {
          'ZoneName' => {
            'endpoints' => ['NodeName'],
            'parent'    => "$parent",
          },
        },
      }
    }

    # Create a API user, for submit passive results
    if $apiuser_name {
      ::icinga2::object::apiuser { "$apiuser_name":
        ensure       => 'present',
        password     => $apiuser_password,
        permissions  => ["*"],
        target       => '/etc/icinga2/conf.d/api-users.conf',
      }
    }

    # Default zones
    ::icinga2::object::zone { 'linux-commands':
      global => true,
      order  => '47',
    }
    ::icinga2::object::zone { 'global-templates':
      global => true,
      order  => '48',
    }
    ::icinga2::object::zone { 'director-global':
      global => true,
      order  => '49',
    }

    # Create zones and endpoints out of Hiera
    create_resources(::icinga2::object::zone, $zones )
    create_resources(::icinga2::object::endpoint, $parent_endpoints )

    # Include the Icinga / Nagios plugins
    contain ::profile::icinga2::plugins

    firewall { '501 allow incoming Icinga2 connections':
      chain  => 'INPUT',
      dport  => [5665],
      proto  => 'tcp',
      action => 'accept',
    }
  }
}
```

### Icingaweb2

Icingaweb2 wird ebenfalls auf dem Master installiert, daher haben wir auch dafür ein passendes Manifest.\

{{% alert theme="warning" %}}**Wichtig ist zu wissen, dass die Icingaweb2 Module wie Puppet und Co per Git und nicht per Paket installiert werden.** {{% /alert %}}

{{% alert theme="warning" %}}Ab Icingaweb2 Director Modul v1.7, werden drei weitere Module benötigt, für die noch kein [Puppet Rezept existiert](https://github.com/Icinga/puppet-icingaweb2/issues/247). Dies wird hier noch aufgeführt, bis es eine offizielle Lösung für puppet-icingaweb2 gibt. {{% /alert %}}


* **dev/modules/profile/manifests/icinga2/icingaweb2.pp**

```puppet
# Install Icingaweb and setup all the basics.
class profile::icinga2::icingaweb2 (

  String   $web_db_name = hiera('icingaweb2::mysql_db'),
  String   $web_db_user = hiera('icingaweb2::mysql_user'),
  $web_db_pass          = Sensitive(hiera('icingaweb2::mysql_password')),
  String   $web_db_host = hiera('monitoring::mysql::ipaddress'),
  String  $ido_db_name = hiera('icinga::mysql_db'),
  String  $ido_db_user = hiera('icinga::mysql_user'),
  $ido_db_pass         = Sensitive(hiera('icinga::mysql_password')),
  $icinga2_api_pass   = Sensitive(hiera('icinga::api::director::password')),
  String $icinga2_api_user = hiera('icingaweb2::icinga2::api_user')
) {
    package { ['php-gettext', 'php-curl', ]:
    ensure =>  $ensure,
  }

  class { '::icingaweb2':
    import_schema => true,
    db_type       => 'mysql',
    db_host       => $web_db_host,
    db_username   => $web_db_user,
    db_password   => $web_db_pass.unwrap,
  }

  class {'icingaweb2::module::monitoring':
    ido_host             => $web_db_host,
    ido_db_name          => $ido_db_name,
    ido_db_username      => $ido_db_user,
    ido_db_password      => $ido_db_pass.unwrap,
    commandtransports    => {
      icinga2 => {
        transport => 'api',
        username  => $icinga2_api_user,
        password  => $icinga2_api_pass.unwrap,
      }
    }
  }

  -> augeas { 'php.ini':
    context => '/files/etc/php.ini/PHP',
    changes => ['set date.timezone Europe/Berlin',],
  }

  # For resources
  $myresource = hiera('icingaweb2::config::resource', {})
  create_resources( 'icingaweb2::config::resource', $myresource)

  # For auth config
  $myauthconfig = hiera('icingaweb2::config::authmethod', {})
  create_resources( 'icingaweb2::config::authmethod', $myauthconfig)

  # For group config
  $mygroupconfig = hiera('icingaweb2::config::groupbackend', {})
  create_resources( 'icingaweb2::config::groupbackend', $mygroupconfig)

  # IcingaWeb2 - Roles
  $icingaweb_roles = hiera_hash( icingaweb2::config::role, undef )
  if( $icingaweb_roles ) { create_resources( icingaweb2::config::role, $icingaweb_roles ) }

  # IcingaWeb2 - LiveStatus
  $icingaweb_livestatus = hiera_hash( icingaweb2::config::resource_livestatus, undef )
  if( $icingaweb_livestatus ) { create_resources( icingaweb2::config::resource_livestatus, $icingaweb_livestatus ) }

    # Create User for Director daemon

  user { 'icingadirector':
    ensure     => 'present',
    comment    => 'Background user for Icingaweb2',
    managehome => true,
    system     => true,
    home       => '/var/lib/icingadirector',
    shell      => '/bin/false',
    gid        => 'icingaweb2',
    require    => Package['icingaweb2']
  }->

  systemd::unit_file { 'icinga-director.service':
    source => "/usr/share/icingaweb2/modules/director/contrib/systemd/icinga-director.service",
    enable => true,
    active => true,
  }->

  contain ::icingaweb2::module::director

   # Add PuppetDB module
  $puppetdb_host = hiera('icingaweb2::module::puppetdb::host')
  $my_certname   = $::fqdn
  $ssldir        = $::settings::ssldir
  $web_ssldir    = '/etc/icingaweb2/modules/puppetdb/ssl'
  $ssl_subdir    = "${web_ssldir}/${puppetdb_host}"

  file { $web_ssldir:
    ensure => directory,
  }

  file { $ssl_subdir:
    ensure  => directory,
    source  => $ssldir,
    recurse => true,
  }

  ~> exec { "Generate combined .pem file for $puppetdb_host":
    command     => "cat private_keys/${my_certname}.pem certs/${my_certname}.pem > private_keys/${my_certname}_combined.pem",
    path        => ["/usr/bin", "/bin"],
    cwd         => $ssl_subdir,
    refreshonly => true
    }

  firewall { '100 allow http and https access':
    dport  => [80,443],
    proto  => tcp,
    action => accept,
  }
}
```

### Hiera

Damit haben wir die wichtigsten Manifests zusammen. Gehen wir zum Hiera Teil über.\
Ein erheblicher Anteil kann nun über Hiera definiert werden. Fangen wir beim Master an und gehen dann Monproxy und Agent.

#### Master

Es gibt drei Orte die zum Einsatz kommen:

1. hieradata/common.eyaml
    - Für allgemeine Parameter die für Master und Agents gleichermaßen gültig sind
2. hieradata/role/master.yaml
    - Parameter die nur die Master betreffen 
3. hieradata/node/office-ffm-master-01.4lin.net.eyaml
    - Parameter die nur speziell diese Node betreffen. 

Fangen wir wir mit der `hieradata/common.eyaml` an. Alles mit **secret** muss natürlich durch eigene Kennwörter getauscht werden. Die monitoring Schlüssel sind sozusagen übergeordnet, da sie nicht nur bei Icinga2 zum Einsatz kommen, sondern auch noch für z.B. Grafana oder ähnliches. Ist aber reine Geschmackssache.

```yaml
# Common class
---
# Add the Icinga2 Agent profile to the others
classes:
    # ...
    - profile::icinga2::agent
# ...
# ...
############################ Monitoring settings ########################
monitoring::domain: '4lin.net'
profile::icinga2::agent::zones:
  "master":
    endpoints:
      - 'office-ffm-master-01.4lin.net'

profile::icinga2::agent::parent_endpoints:
  'office-ffm-master-01.4lin.net':
    ensure: 'present'

##########################
### MySQL related settings
###########################
# MariaDB Host or HA IP
'monitoring::mysql::ipaddress': '192.168.1.30'
'monitoring::mysql::port': 3306


##########################
## icinga related settings
##########################

###########  Icinga secret ##########
icinga::mysql_password: DEC::GPG[secret]!
icinga::influxdb::password: DEC::GPG[secret]!
# generatated with openssl rand -hex 42
icinga::api::ticketsalt: DEC::GPG[752490c98cb5d3616c327e3123db39fae889c7f17c6871d984a55aa4fbfb1a3209301737d30fbcd20dc7]!
icinga::api::root::password: DEC::GPG[secret]!
icinga::api::director::password: DEC::GPG[secret]!
icinga::api::dashing::password: DEC::GPG[secret]!
icinga::api::production::password: DEC::GPG[secret]!
icinga::api::ticketsalt: DEC::GPG[long_secret]!

###########  Icingaweb2 secret ##########
icingaweb2_director::mysql_password: DEC::GPG[secret]!
icingaweb2_x509::mysql_password: DEC::GPG[secret]!
icingaweb2::mysql_password: DEC::GPG[secret]!
```

Die `hieradata/role/master.yaml` beinhaltet alles für die Master Nodes:
* Icinga2 selbst
    - Zonen
    - Endpoints
    - Plugins
* Apache2 für Icingaweb2
    - PHP
* Icingaweb2

```yaml
---
# Basics
classes:
  - profile::icinga2::master
  - profile::webserver::apache2
  - profile::webserver::apache2_php
  - profile::webserver::apache2_remoteip
  - profile::icinga2::icingaweb2
  - 'ntp'

ntp::autoupdate: false
ntp::enable: true
ntp::servers:
  - 192.53.103.108 iburst
  - 131.188.3.222 iburst

apt::sources:
  'debian_icinga2':
    comment: 'hieradata/role/mon.yaml'
    location: 'http://packages.icinga.com/debian'
    release: "icinga-stretch"
    repos: 'main'
    include:
      src: false
      deb: true

############# Icinga2 settings #############
# Variable: monitoring::icinga::fqdn
# Description:
# Default value:
'monitoring::icinga::fqdn': "office-ffm-master-01.4lin.net"
'icinga::mysql_db': 'icinga2_ido_db'
'icinga::mysql_user': 'icinga2_ido_db'

'icingaweb2::mysql_db': 'icingaweb2_db'
'icingaweb2::mysql_user': 'icingaweb2_db'
'icingaweb2::icinga2::api_user': 'icingaweb2_director'

'icingaweb2_director::mysql_db': 'icingaweb2_director_db'
'icingaweb2_director::mysql_user': 'icingaweb2_director_db'

icinga2::features:
  - 'notification'
  - 'checker'
  - 'mainlog'
  - 'statusdata'
  - 'command'

# icinga2::manage_database: true
icinga2::restart_cmd: 'service icinga2 reload'
icinga2::plugins:
  - 'nscp'
  - 'plugins'
  - 'plugins-contrib'
  - 'windows-plugins'
  - 'manubulon'


# Endpoints
icinga2::feature::api::endpoints:
  'office-ffm-master-01.4lin.net':
    host: 192.168.1.34

# Zones
icinga2::feature::api::zones:
  master:
    endpoints:
      - 'office-ffm-master-01.4lin.net'


# Manage PKI
icinga2::feature::api::pki: 'icinga2'

############# Icinga2 API User #############

icinga2::object::apiuser:
  'root':
    target: '/etc/icinga2/conf.d/api-user.conf'
    apiuser_name: 'root'
    password: "%{hiera('icinga::api::root::password')}"
    permissions:
      - '*'
  'icingaweb2_director':
    target: '/etc/icinga2/conf.d/api-user.conf'
    apiuser_name: 'icingaweb2_director'
    password: "%{hiera('icinga::api::director::password')}"
    permissions:
      - '*'
  'dashing':
    target: '/etc/icinga2/conf.d/api-user.conf'
    apiuser_name: 'dashing'
    password: "%{hiera('icinga::api::dashing::password')}"
    permissions:
      - '*'
  'production':
    target: '/etc/icinga2/conf.d/api-user.conf'
    apiuser_name: 'production'
    password: "%{hiera('icinga::api::production::password')}"
    permissions:
      - '*'

############# Icingaweb2 #############
icingaweb2::module::puppetdb::host: 'office-ffm-srv-puppet.4lin.net'
icingaweb2::module::incubator::git_revision: 'v0.5.0'
icingaweb2::module::ipl::git_revision: 'v0.3.0'
icingaweb2::module::reactbundle::git_revision: 'v0.7.0'
icingaweb2::module::monitoring::protected_customvars: '*pw*,*pass*,community,*key*,*priv*,*password*'
icingaweb2::module::director::db_name:  "icingaweb2_director_db"
icingaweb2::module::director::db_username:  "icingaweb2_director_db"
icingaweb2::module::director::db_password: "%{hiera('icingaweb2_director::mysql_password')}"
icingaweb2::module::director::db_host:  "%{hiera('monitoring::mysql::ipaddress')}"
icingaweb2::module::director::db_port:  3306
icingaweb2::config::resource:
  'icingaweb2_director_db':
    host:  "%{hiera('monitoring::mysql::ipaddress')}"
    type: 'db'
    db_type:  'mysql'
    db_name:  "icingaweb2_director_db"
    port:  3306
    db_charset:  "utf8"
    db_username:  "icingaweb2_director_db"
    db_password:  "%{hiera('icingaweb2_director::mysql_password')}"

icingaweb2::db: 'mysql'
icingaweb2::db_name: "icingaweb2_db"
icingaweb2::db_user: "%icingaweb2_db"
icingaweb2::db_password: "%{hiera('icingaweb2::mysql_password')}"
icingaweb2::db_host: "%{hiera('monitoring::mysql::ipaddress')}"

icingaweb2::config::authmethod:
  'mysql':
    backend: 'db'
    resource: 'icingaweb2_director_db'
    order: '01'

icingaweb2::config::role:
  'Administrators':
    groups: 'Gruppe_Icinga_admins'
    permissions: '*'
  'Members':
    groups: 'Gruppe_Icinga_users'
    permissions: 'module/doc, module/monitoring, dashboards, monitoring/commands/schedule-check, monitoring/command/acknowledge-problem, monitoring/command/remove-acknowledgement, monitoring/command/remove-acknowledgement, monitoring/command/downtime/*, monitoring/command/comment/*, monitoring/command/comment/add, monitoring/command/comment/delete'

############# Apache2 settings #############

apache::default_mods: false
apache::mpm_module: prefork
apache::mod::remoteip::internal_proxy: []
apache::vhost:
  "%{::fqdn}":
    servername: "office-ffm-master.4lin.net"
    serveradmin: 'webmaster@%{::domain}'
    serveraliases:
      - "%{::fqdn}"
    port: 80
    docroot: '/var/www'
    additional_includes: '/etc/apache2/conf-enabled/dehydrated.conf'
    docroot_owner: 'www-data'
    docroot_group: 'www-data'
    custom_fragment: |
      RewriteEngine On
      # Ugly escape needed, for Hiera # https://tickets.puppetlabs.com/browse/HI-127
      RewriteCond %%{%}{REQUEST_URI} !^/server-status [OR]
      RewriteCond %%{%}{REQUEST_URI} !^/.well-known
      RewriteRule (.*) https://office-ffm-master-01.4lin.net/icingaweb2/$1 [R=301,L]
      RemoteIPHeader X-Forwarded-For
  "%{::fqdn}_ssl":
    servername: "office-ffm-master.4lin.net"
    serveradmin: 'webmaster@%{::domain}'
    serveraliases:
      - "%{::fqdn}"
    ssl: true
    ssl_cert: "/etc/ssl/certs/ssl-cert-snakeoil.pem" # gegen richtige Zertifikate tauschen !
    ssl_key: "/etc/ssl/private/ssl-cert-snakeoil.key" #gegen richtigen Schlüssel tauschen !
    port: 443
    docroot: '/var/www/html'
    # Wir binden die Icingaweb2 Config hier ein und nutzen nicht die Include Datei /etc/apache2/conf-enabled/icingaweb2.conf !
    custom_fragment: |
      RemoteIPHeader X-Forwarded-For
    aliases:
      -
        alias: '/icingaweb2'
        path: '/usr/share/icingaweb2/public'
    directories:
      -
        path: '/usr/share/icingaweb2/public'
        options: 'SymLinksIfOwnerMatch'
        overrides: 'None'
        custom_fragment: |
          SetEnv ICINGAWEB_CONFIGDIR "/etc/icingaweb2"
          EnableSendfile Off
          RewriteEngine on
          RewriteBase /icingaweb2/
          RewriteCond %%{%}{REQUEST_FILENAME} -s [OR]
          RewriteCond %%{%}{REQUEST_FILENAME} -l [OR]
          RewriteCond %%{%}{REQUEST_FILENAME} -d
          RewriteRule ^.*$ - [NC,L]
          RewriteRule ^.*$ index.php [NC,L]
      -
        path: '/server-status'
        provider: 'location'
        require: "ip %{::ip} 127.0.0.1"

#  "dash_ssl":
#    servername: 'dash.%{::domain}'
#    serveradmin: 'webmaster@%{::domain}'
#    ssl: true
#    ssl_cert: "%{hiera('ssl_inatec_chain_path')}"
#    ssl_key: "%{hiera('ssl_inatec_key_path')}"
#    port: 443
#    custom_fragment: |
#      <Proxy *>
#       Require ip 192.168
#      </Proxy>
#      RemoteIPHeader X-Forwarded-For
#    docroot: '/var/www'
#    proxy_preserve_host:  true
#    proxy_pass:
#      path: '/'
#      url:  'http://127.0.0.1:8005/'
```

* **profile/dev/hieradata/node/office-ffm-master-01.4lin.net.eyaml**

```yaml
---
# We are config master
icinga::config_master: true
```

## Erster Puppetlauf

Hat man es bis zu diesem Punkt geschafft, kann man mittels `puppet agent -t --noop` einen ersten Testlauf starten und nach Syntaxfehlern Ausschau halten. Erfahrungsgemäß gibt es jede Menge davon. Des weiteren kann (und wird) es nötig sein, den Lauf mehrfach zu starten, da das ganze Konstrukt zu komplex ist, als dass es auf Anhieb durchläuft.

Ein Erfolgt ist es, wenn der Login auf Icingaweb2 bereits klappt, auch wenn noch keine Hosts sichtbar sind, da die Standard Konfiguration von Puppet entfernt wird. Stattdessen nutzen wir das Director Modul und Puppet um neue Hosts hinzuzufügen.

## Director PuppetDB

Nach dem Puppet durchgelaufen ist, können wir den PuppetDB Host hinzufügen und auf die dort enthaltenen Ressourcen zugreifen.

![PuppetDB Anbindung](/doc/uploads/monitoring/icingaweb2_director_puppetdb.png)

Wurde die erste Synchronisierung durchgeführt, dürften wie in diesem Fall zwei Nodes bereits in der Vorschau auftauchen. Das allein hilft noch nicht, den wir müssen dem Modul noch sagen, wie es damit verfahren soll.\

### Director Vorarbeit

Bevor die ersten Hosts im Icinga2 auftauchen und überwacht werden können, sind noch einige Schritte zuvor notwendig:

1. Erstellen von eigenen Datenfeldern
  - wir benötigen host.vars.os = Linux, damit die Apply Regeln greifen
  - wir benötigen host.vars.client_endpoint
  - werden über den Director später automatisch befüllt
2. Host Templates erstellen
  - Generic Host (oder generic-host) -> Für ICMP oder Hostalive Hostcheck
  - Director Host -> Bindet Generic Host ein, zusätzlich wird aktive Agent Verbindung gesetzt (Endpoint / Zone )
  - Datenfeld "os" hinzufügen 
3. generic-service Template erstellen
  - es muss dieser Name sein, da in vielen Apply Regeln auf diesen Namen zurückgegriffen wird
  - Dient zum Setzen von Standardwerten

#### Datenfelder

Fangen wir mit dem Datenfeld an. Wir erstellen eine Datenliste mit unterschiedlichen Betriebssystemen. Das ist besser als einfach nur ein String, wegen unterschiedlicher Schreibweisen:

![Director Datenliste](/doc/uploads/monitoring/icingaweb2_director_datalist.png)

Die Liste selbst habe ich "Operation Systems" genannt. Als Schlüssel und Bezeichnung habe ich das Gleiche genommen.

Es folgt als nächstes das Datenfeld "os", welches sich auf diese Liste bezieht:

![Director Datenfeld os](/doc/uploads/monitoring/icingaweb2_director_os_datafield.png)

Folgt als letzten das Feld host.vars.client_endpoint. Das sorgt in den Apply Regeln dafür, dass der Check auf wirklich auf dem gewünschten Host ausgeführt wird:

![Director Datenfeld client_endpoint](/doc/uploads/monitoring/monitoring_director_client_endpoint_datafield.png)

Das waren für den Anfang die wichtigsten Felder.

#### Templates

Nun benötigen wir zwei Host Templates und ein Service Template. Das erste Host Template "Generic Host" (oder falls lieber "generic-host") hat lediglich den Hostcheck gesetzt. In meinem Fall verwende ich lieber das Check Kommando "ICMP", statt dem "Hostalive".\

![Director Generic Host Template](/doc/uploads/monitoring/monitoring_director_generic_host_template.png)

Wurde das Template erstellt, muss auch das Feld "os" und "client_endpoint" hinzugefügt werden.

![Director Generic Host Field](/doc/uploads/monitoring/monitoring_director_host_field.png)

Das zweite Host Template "Director Host" sorgt dafür, dass beim Import durch den Director aus dem Puppet heraus, auch die notwendigen Zonen und Endpoints erstellt werden. Dafür muss lediglich unter den "Icinga Agenten- und Zoneneinstellungen Einstellungen" die Parameter dafür aktiviert werden:

![Director Host Template](/doc/uploads/monitoring/monitoring_director_host_template.png)

Das Service Template "generic-service" wird wie das Host Template erstellt, aber es bedarf da aktuell keine weiteren Parameter, für dieses HowTo. Einen Screenshot erspare ich mir an dieser Stelle. 

#### Sync Regeln

Nun kommt der fummligste Abschnitt: das Erstellen der Synchronisationsregeln.

Grob gesagt geht es darum, dem Director zu sagen, welches Puppet Fact (ip / hostname / role / blockdevice ...) zu welchem Feld im Icinga2 Host Object gehören soll.

* Ein Beispiel:

| Puppet Fact | Icinga Host Object |
|-------------|---------------|
| fact.ip | host.address |
| certname | host.display_name |
| fact.fqdn | host.vars.fqdn |
| fact.role | host.vars.role |

Jedes gewünschte Feld im Icinga Host Object muss zuvor auch im Director unter Datenfelder erstellt werden (meistens als String), damit es dann in der Regel zugeordnet werden kann.\
Der nächste wichtige Punkt ist der, dass die neuen Datenfelder im Host Template (z.B Generic Host oder Director Host) hinzugefügt wurden.

Hier werde ich sechs Regeln exemplarisch aufzeigen.

Zum erstellen des Regelsets im Director müsst ihr wie im Screenshot dargestellt auf Icinga-Director gehen, dann Automatisierung, Syncronisationsregel. Dort wird ein neuer Regelsatz erstellt:

![Director Role Set](/doc/uploads/monitoring/monitoring_director_rule_set.png)

Hier habe ich den Regelsatz "Master PuppetDB Sync" genannt und als Objekttyp "Host", Aktualisierungsrichtlinie "Zusammenführen" und Bereinigen mit "Ja" angegeben.

Wurde der Satz erstellt, kommen die Regeln. Wählt den neu erstellten Regelsatz aus und dann direkt auf "Eigenschaften". An dieser Stelle wird nun dem Director gesagt, was wir gerne wo hätten. Als Beispiel habe diese gewählt:

![Director Datenfeld client_endpoint](/doc/uploads/monitoring/monitoring_director_sync_rules.png)

| Puppet Fact | Icinga Host Object |
|-------------|---------------|
| icinga2_zone | zone |
| certname | display_name |
| facts.ipaddress | address |
| certname | object_name |
| facts.kernel | vars.os |
| Director Host | import |
| cername | vars.client_endpoint |

Die Möglichkeit einem Host Object Templates mit auf dem Weg zu geben, ist besonders hervorzuheben. In diesem Beispiel wird dem zu erstellendem Icinga Host Object das Template "Director Host" mitgegeben. In Kombination mit dem Filter kann ich sagen, dass der Host  "office-ffm-srv-puppet" -- welcher den Hostname "srv" hat -- zum Beispiel ein Template "SRV Host" mitbekommt. Von dieser Funktion machen wir reichlich Gebrauch, um gleichartige Hosts zu erschlagen.\
Ein fertiger Host vom Director sieht dann zum Beispiel so aus:

```
zones.d/master/hosts.conf

object Host "office-ffm-srv-puppet.4lin.net" {
    display_name = "office-ffm-srv-puppet.4lin.net"
    address = "192.168.1.33"
    check_command = "icmp"
    vars.client_endpoint = "office-ffm-srv-puppet.4lin.net"
    vars.os = "Linux"
}

zones.d/master/agent_endpoints.conf

object Endpoint "office-ffm-srv-puppet.4lin.net" {
    host = "192.168.1.33"
    log_duration = 0s
}

zones.d/master/agent_zones.conf

object Zone "office-ffm-srv-puppet.4lin.net" {
    parent = "master"
    endpoints = [ "office-ffm-srv-puppet.4lin.net" ]
}
```

Fügen wir noch das Fact "role" hinzu:

![Director add role fact](/doc/uploads/monitoring/monitoring_director_add_role.png)

Das Feld "role" habe ich natürlich vorher schon hinzugefügt. Damit können wir am Ende dann Apply Regeln erstellen, die nach host.vars.role == "foo" schauen :-)

Nehmen wir noch einen Import hinzu ! Dafür habe ich ein weiteres Template erstellt, das "DB Host" heißt und möchte, dass jedes Icinga Host Object dieses Template erhält, welches als `certname=*db*` enthält:

![Director import DB host](/doc/uploads/monitoring/monitoring_director_db_host.png)

Ist das auch geschafft, kann man im ersten Karteireiter den Sync anstoßen und sollte dann folgendes Ergebnis erhalten:

```
zones.d/master/hosts.conf

object Host "office-ffm-db-01.4lin.net" {
    import "Director Host"
    import "DB Host"

    display_name = "office-ffm-db-01.4lin.net"
    address = "192.168.1.30"
    vars.client_endpoint = "office-ffm-db-01.4lin.net"
    vars.os = "Linux"
    vars.role = "db"
}

zones.d/master/agent_endpoints.conf

object Endpoint "office-ffm-db-01.4lin.net" {
    host = "192.168.1.30"
    log_duration = 0s
}

zones.d/master/agent_zones.conf

object Zone "office-ffm-db-01.4lin.net" {
    parent = "master"
    endpoints = [ "office-ffm-db-01.4lin.net" ]
}
```

Und das ist das **geniale**, denn mit dieser Funktion könnte ihr nahezu alles in ein Host Objekt packen lassen. Wir übergeben damit sogar Festplattentypen und Mountpoints, um dann mit entsprechenden Apply Regeln darauf zu reagieren.

## Danksagung

An dieser Stelle gilt mein Dank vor allem der [Netways Truppe](https://www.netways.de/netways/team/) Ich muss zugeben, dass ich ein Teil meines Gehaltes wegen ihrer Arbeit erhalte :-) Da sind besonders zu nennen:

* Lennart Betz - Der Mensch hinter den Puppet Modulen
* Thomas Gelf - Der Icingaweb2 Mensch
* Michael (dnsmichi) Friedrich - Der Typ der dafür sorgt, dass Icinga2 einen Saint Nessus Scan überlebt :-D

Dann wäre da noch [Marianne M.Spiller](https://www.unixe.de), mit [deren Hilfe](https://www.unixe.de/icinga2-director-die-einrichtung/) ich überhaupt erst den Director kennen und nutzen gelernt habe.

Dann kämen natürlich noch die unzähligen Menschen von Puppet selbst und deren Module. Die wichtigsten für mich sind da:

* [Steffen Zieger](https://forge.puppet.com/saz) - hat etliche Puppet Module geschrieben
* [Rob Nelson](https://forge.puppet.com/rnelson0) - ebenfalls ettliche Puppet Module und vor allem Rollen / Profile erklärt unter Puppet erklärt. 

# Links

* Rollen und Profile mit Hiera -https://rnelson0.com/2014/07/14/intro-to-roles-and-profiles-with-puppet-and-hiera/
* Icingaweb Director - https://www.unixe.de/icinga2-director-die-einrichtung/
* Puppet Beispiele für Icinga2 - https://github.com/Icinga/puppet-icinga2/tree/master/examples