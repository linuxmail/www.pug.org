---
title: Mitmachen
---

# Mitmachen ?

## Webseite

Wenn du gerne ein wenig von dir erzählen möchtest, wie du zu Computern, Linux und Co. gekommen bist, oder sogar eine Anleitung für andere schreiben möchtest, kannst du dies sehr gerne tun.\

### Wie ?

#### Unser Konstrukt

Die Webseite der Pug.org wird automatisch mit der Hilfe von [Hugo](https://hugo.io) und Markdown Dateien generiert. Die Dateien selbst werden über Git und [https://www.github.com](https://github.com/linuxmail/www.pug.org/) verwaltet, wobei Github als Vermittler agiert.

* Eine etwas vereinfachte Grafik:

{{<mermaid align="center">}}
    graph LR;
    A[Repo klonen]-->|git clone| H
    A --> B
    B[Neue Datei / Editieren] -->|testen| C(hugo server)
    C--> D{Sieht gut aus?}
    D-->|Ja| E[push to git]
    E-->|Nein| B
    E[git commit] -->|git push| H
    H(www.github.com)
    G(www.pug.org)
    G -->|git pull| H
{{< /mermaid >}}

Mit einigen Sätzen erläutert, sieht das vollständige Konstrukt so aus (Variante 2):\

1. Das Git Repository auf den eigenen Rechner geklont
2. Eine neue Datei im Markdown Format anlegen, oder eine vorhandene Datei editieren
3. Die Änderung testen, mittels `hugo server` und der URL http://localhost:1313/doc
4. Die Änderung an Git übergeben und hochladen
5. Ein Cron Script auf www.pug.org holt die Änderungen
6. Ein Script wandelt die Dateien via Hugo in statisches HTML und speichert sie im Webserver Verzeichnis

Da wir Github als Vermittler verwenden, gibt es zwei Möglichkeiten an der Webseite mitzuarbeiten:

### Variante 1 - direkt editieren über Github 

Das ist die einfachste und schnellste Variante. Einzige Voraussetzung ist das Vorhandensein eines [Github Accounts](https://github.com/join?source=header-home). Diesem Account können wir dann eine Schreibberechtigung erteilen.\
Auf jeder Seite finden sich ganz unten ein winziges Symbol, welches aussieht wie eine Verzweigung. Ein Klick darauf führt direkt zu Github im Editiermodus.
Darüber lassen sich auch neue Seiten etc. anlegen.\
Einziger Nachteil: die Änderung wird auf der Webseite erst nach ein paar Minuten sichtbar.\
Diese Methode eignet sich gut um Kleinigkeiten zu ändern.

### Variante 2 - Git

#### Voraussetzungen

Wir benötigen das [Git Kommando](https://git-scm.com/book/de/v1/Los-geht%E2%80%99s-Git-installieren) und den öffentlichen Teil des eigenen SSH Schlüssels. Es ist außerdem noch von Vorteil, Hugo [selbst zu haben](https://gohugo.io/getting-started/installing/) auf dem Rechner zu haben, um Änderungen vorab zu begutachten, **bevor** diese hochgeladen werden. 

#### Im Detail

Zwei Punkte sind vorab wichtig zu wissen:

1. Das Theme [Dockdoc](https://docdock.netlify.com/) ist ein eigenständiges Git Repo und wird mittels --recursive ebenfalls von Github bezogen
2. Arbeiten im Hauptzeit sollte vermieden werden, stattdessen kann man zum Beispiel in einem eigenen Zweig (z.B. develop) arbeiten und später mit dem Hauptzweig vereinigen (merge)

Auf der Kommandozeile sieht das so aus:

* Repo klonen; zum develop Zweig wechseln und Hugo starten

```bash
$ git clone --recursive git@github.com:linuxmail/www.pug.org.git
$ cd www.pug.org
$ git checkout develop
$ hugo server
```

* URL aufrufen: http://localhost:1313/doc
* Dateien im Ordner content/ erzeugen oder vorhandene Dateien editieren
* Im Browser überprüfen, ob das Ergebnis OK ist
* Die Änderungen nun an Git übergeben ; mit dem Master Zweig vereinen und auf Github hochladen.

```bash
$ git status
$ git commit -m "Meine Änderungen"
$ git checkout master  ; git merge --no-ff develop ; git push ; git checkout develop
```

Ein paar Minuten später ist das Ergebnis auf https://www.pug.org zu sehen.

### Die Syntax ###

Wir bereits erwähnt, verwenden wir [Dockdoc](https://docdock.netlify.com/create-page/) als Grundlage.\
Für den Kopf (die Metadaten der jeweiligen Seite), verwenden wir allerdings nicht TOML (+++), sondern die YAML (---) Syntax.\
Unter dem angegeben Link finden sich alle notwendigen Seiten, was das Aussehen betrifft.

Für die MarkDown Syntax selbst, findet sich [hier](https://sourceforge.net/p/hugo-generator/wiki/markdown_syntax/) alles wichtige.\
Am einfachsten ist es, in die bereits bestehenden Seiten zu schauen, wie etwas formatiert wurde.


Bei Fragen: einfach eine E-Mail an die Mailingliste :-)