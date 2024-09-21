---
title: Bloggen mit Hugo & Cloudflare Pages
date: 2024-09-21
description: | 
    Nachdem ich meinen Blog letztens zu Hugo umgzogen und per Cloudflare Pages veröffentlicht habe, zeige ich in diesem Beitrag, wie.  
categories:
    - services
tags:
    - blog
    - hugo
    - cloudflare
    - git
    - github
    - domain
---

## Intro

Wenn man an Blogs im Internet denkt, kommt vielen erst einmal Wordpress in den Sinn. Auch ich habe es eine Zeit lang selbst gehostet und genutzt, denn es war komfortabel, von überall erreichbar und dank Plugins leicht erweiterbar. Wordpress ist jedoch relativ schwer, viele Funktionen benötige ich nicht und da es mit das bekannteste CMS ist, sind auch die Angriffsmöglichkeiten entsprechend verbreitet. Deshalb habe ich mich dazu entschieden, auf [Hugo](https://gohugo.io/) umzusteigen. 

Hugo ist ein Open-Source-Framework zur Generierung statischer Seiten. Die Inhalte werden in Markdown geschrieben und Hugo erstellt daraus eine statische HTML-Website, die mit jedem beliebigen Webserver gehostet werden kann. Da die Seiten vorgeneriert werden, ist der Zugriff schneller. Dank Git ist alles versioniert, zudem kann die Website einfach über [Github Pages](https://pages.github.com/) oder [Cloudflare Pages](https://pages.cloudflare.com/) bereitgestellt werden, wenn man Github nutzt und den Inhalt nicht selbst hosten möchte. 

Ich habe mich für Cloudflare Pages im kostenlosen Tarif entschieden. Gründe meinerseits waren mitunter:
- Ich nutze Cloudflare schon für die DNS-Verwaltung meiner Domain
- Der Inhalt wird über das Cloudflare-CDN bereitgestellt
- Unlimitierte statische Requests und Bandbreite
    - Github Pages haben im kostenlosen Tarif ein Limit von 100 GB / Monat. Auch wenn das erstmal erreicht werden muss, ist es eben eine Grenze, die man im Hinterkopf behalten müsste
- Integration mit Github und Gitlab. Bei jedem `git push` auf den  konfigurierten Branch wird der Build-Prozess angestoßen und die fertige Seite veröffentlicht

Auch wenn man die Inhalte nicht wie bei Wordpress über die Weboberfläche erstellt und bearbeitet, ist Hugo nicht sonderlich komplex. Man sollte jedoch ein Grundverständnis von git und keine Angst vor der Konsole haben.

## Den Blog lokal erstellen

Auf dem System sollte Git und Hugo installiert sein. Unter Linux sollte Git in der Regel vorinstalliert sein und Hugo über den Paketmanager bezogen werden können, unter Windows empfiehlt es sich, das **Windows Subsystem for Linux (WSL)** einzusetzen. Unter Windows nutze ich z.B. Ubuntu 24.04 LTS per WSL, mit meinen Linux-Systemen nutze ich Hugo direkt.

> **Hinweis:** 
> 
> Mit WSL sind die Festplatten von Windows unter `/mnt` verfügbar. Öffnet man WSL aus dem Explorer oder über cmd, befindet man sich weiterhin im entsprechenden Ordner, z.B. `/mnt/c/Users/<Username>`. Erstellt man den Ordner für den Blog in einem von Windows gemounteten Ordner, funktioniert das Live-Preview von Hugo nicht, da Dateiänderungen nicht verfolgt werden. Stattdessen sollte ein Ordner direkt im Dateisystem von WSL angelegt werden, beispielsweise unter `~/blog`.

Nachdem man zum übergeordneten Ordner navigiert ist, führt man folgenden Befehl aus. Hierbei ist `blog` beispielsweise der Name des Ordners, der angelegt wird, und kann beliebig gewählt werden:

```
hugo new site blog
```

Damit erstellt hugo die entsprechende Ordnerstruktur und erstellt verschiedene Dateien. Nachdem man in den angelegten Ordner gewechselt hat, initialisiert man dort noch das leere Git-Repository:

```
git init
```

Jetzt fehlt nur noch ein Theme. Hugo bietet eine [Übersicht über verschiedene Themes mit Vorschaubildern](https://themes.gohugo.io/) an, über die Tags lässt sich die gewünschte Funktionalität filtern. Ich habe mich selbst erst einmal für [Stack](https://themes.gohugo.io/themes/hugo-theme-stack/) entschieden, bei Nichtgefallen lässt sich das Theme jedoch einfach austauschen, ohne den Inhalt der Website anpassen zu müssen.

Der Quellcode hinter den Themes, die auf der Übersichtsseite angeboten werden, ist in Repos auf Github zu finden. Diese lassen sich als [Git Submodul](https://git-scm.com/book/en/v2/Git-Tools-Submodules) hinzufügen. In meinem Fall mit Stack:

```
git submodule add https://github.com/CaiJimmy/hugo-theme-stack.git themes/stack
```

Abschließend muss in der Konfiguration nur noch festgelegt werden, welches Theme verwendet wird. Dafür muss in der Datei `hugo.toml` folgende Zeile ergänzt werden: 

```
theme = 'stack'
```

> **Hinweis:** 
> 
> Diese Konfigurationsdatei ist relativ einfach gehalten. Damit kann man Hugo und das Theme weiter konfigurieren. Statt einer einzelnen Datei kann man auch eine Ordnerstruktur verwenden. Weitere Informationen findet man in der [Dokumentation von Hugo](https://gohugo.io/getting-started/configuration/).
>
> Für Stack findet man im Quick Start-Repo eine [Vorlage für die Konfigurationsdateien](https://github.com/CaiJimmy/hugo-theme-stack-starter/tree/master/config/_default).

Anschließend kann man den Entwicklungsserver von Hugo starten und sich die Website anzeigen lassen, auch wenn bisher kein Inhalt vorhanden ist:

```
hugo server
```

## Das Github-Repository

Als nächstes wird ein neues Repository bei Github erstellt. Der Name kann beliebig gewählt werden, das Repo muss jedoch nicht initialisiert werden.

Ein Vorteil von Cloudflare Pages ist, dass das Repo auch Privat sein kann - mit Github Pages im kostenlosen Tarif muss die Sichtbarkeit *Öffentlich* sein. 

Lokal indexiert man alle angelegten Dateien für den ersten Commit, erzeugt diesen, fügt das zuvor angelegte Repository als Remote hinzu und lädt den Commit hoch. `<Ziel>` ist hierbei entweder `https://github.com/UserName/RepoName.git` (HTTPS) oder `git@github.com:UserName/RepoName.git` (SSH):

```
git add *
git commit -m "Initial Setup"
git branch -M main
git remote add origin <Ziel>
git push -u origin main
```

## Cloudflare Pages

Als nächtes soll der Blog statisch generiert und über Cloudflare Pages bereitgestellt werden. Hierfür klickt man im Cloudflare-Dashboard auf "Workers und Pages > Überblick" und anschließend auf den Button "Erstellen". Wechselt man zum Reiter "Pages", lässt sich ein vorhandenes Git-Repository von Github oder Gitlab importieren. Man kann die Dateien zwar auch direkt hochladen, müsste dies aber bei jeder Änderung manuell machen.

Nach dem Login wählt man den Account sowie das Repository aus, das getrackt werden soll. Danach wählt man einen Projektnamen (dieser Name wird zur Generierung eines dauerhaften Hostnamens auf pages.dev genutzt), den Branch und konfiguriert die Build-Einstellung. Als Preset kann man hierbei **Hugo** auswählen, was auch den Build-Befehl (`hugo`) und das Ausgabeverzeichnis (`public`) ausfüllt. Den Build-Befehl passe ich wie folgt an:

```
hugo --minify -b $BASE_URL
```

Unter "Umgebungsvariablen (erweitert) konfiguriere ich nun noch die Umgebungsvariable `BASE_URL` mit Wert `https://www.akumatic.eu`. Erstelle ich später noch eine Preview-Umgebung, kann ich für diese dann den Wert `$CF_PAGES_URL` einsetzen. Möchte man keine eigene Domain nutzen, kann man hier auch direkt `$CF_PAGES_URL` einsetzen.

Zudem nutzt Cloudflare Pages eine ältere Version von Hugo (in meinem Fall v.0118.2), die mit der aktuellen Version von *Stack* einen Fehler beim Rendern wirft (`<.Site.Lastmod.IsZero>: can't evaluate field Lastmod in type page.Site`). Für dieses Problem existiert ein [Issue auf Github](https://github.com/CaiJimmy/hugo-theme-stack/issues/974), als Lösung muss man die Version von Hugo explizit auf eine Version >= 0.123.0 setzen. Hierfür habe ich noch die Variable `HUGO_VERSION` mit Wert `0.123.8` angelegt.

Fährt man jetzt fort, baut Cloudflare die Seite und veröffentlicht diese. Den Blog kann direkt mit der angezeigten Subdomain von `.pages.dev` aufrufen.

### Eigene Domain nutzen

Eventuell hat man jedoch schon eine eigene Domain gemietet und möchte diese selbst oder eine Subdomain davon nutzen. In meinem Fall möchte ich die Subdomain `www` für die Website selbst nutzen und vom Domain-Apex zu `www` weiterleiten. 

#### Subdomain `www`

Im Dashboard unter "Workers und Pages" wählt man die eben erstellte Page aus und wechselt zum Reiter "*"Benutzerdefinierte Domänen". Nach Klick auf "Benutzerdefinierte Domänene einrichten" kann man die gewünschte Domain oder Subdomain eingeben, die für die Website benutzt werden soll - in meinem Fall `www.akumatic.eu`. Anschließend wird noch gezeigt, welcher DNS-Eintrag erzeugt wird.

Nach der Bestätigung und einer kurzen Wartezeit ist der Eintrag gesetzt und das Zertifikat generiert - die Website kann nun über die Domain aufgerufen werden.

#### Umleitung vom Apex

Analog zu `www` könnte ich auch die Apex-Domain konfigurieren, dadurch wäre die Website direkt durch beide Domains erreichbar. Um den Auftritt konsistent zu halten, richte ich jedoch nur `www` ein und leite Anfragen von `akumatic.eu` zu `www.akumatic.eu` um. 

Hierbei reicht ein einfacher CNAME-Record nicht aus, da das SSL-Zertifikat nur für `www.` ausgestellt wurde. Deshalb wird eine **Umleitungsregel** genutzt, wofür der DNS-Eintrag für die Apex-Domain eingerichtet werden muss. In meinem Fall sieht der Eintrag so aus: 

| Eigenschaft | Inhalt | 
| -- | -- |
| Typ | CNAME |
| Name | `akumatic.eu` |
| Ziel | `www.akumatic.eu` | 
| Proxy-Status | Mit Proxy |

> **Hinweis:** 
> 
> Normalerweise kann man CNAME nicht für den Apex-Eintrag nutzen, Cloudflare ermöglicht dies durch [CNAME Flattening](https://developers.cloudflare.com/dns/cname-flattening/)

Anschließend kann die Umleitungsregel konfiguriert werden. Im Dashboard für die Domain geht man zu "Regeln > Umleitungsregeln" und erstellt eine neue Regel. Der Name kann dabei beliebig gewählt werden. Folgende Einstellungen werden getroffen, damit man bei einer Umleitung auch den vollständigen Pfad beibehält:

**Wenn...**

- [ ] Platzhaltermuster
- [ ] Alle eingehenden Anforderungen
- [x] Benutzerdefinierter Filterausdruck

| Eigenschaft | Inhalt | 
| -- | -- |
| Feld | Hostname |
| Operator | gleich |
| Wert | `akumatic.eu` | 

**Dann...**
URL-Umleitung

| Eigenschaft | Inhalt | 
| -- | -- |
| Typ | Dynamisch | 
| Ausdruck |  `concat("https://www.akumatic.eu", http.request.uri.path` |
| Statuscode | 301 | 
| Abfragezeichenfolge beibehalten | :ballot_box_with_check: | 

## Kommentare mit Giscus

*Stack* bietet mehrere Optionen, um Kommentare zu integrieren, darunter auch Giscus. Giscus hat den Vorteil, dass Kommentare über die Diskussions-Funktion von Github realisiert werden und kein Secret in der Konfiguration hinterlegt werden muss. 

Hierfür muss zum einen die [Diskussions-Funktion für das Repo aktiviert werden](https://docs.github.com/de/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/enabling-or-disabling-github-discussions-for-a-repository), zum anderen muss [Giscus installiert werden](https://github.com/apps/giscus). Dabei kann gewählt werden, ob es für alle oder nur ausgewählte Repos installiert werden soll. 

Zudem habe ich in den Github-Diskussionen die vorhandenen Kategorien aufgeräumt und eine neue Kategorie explizit für die Kommentare vom Typ "Ankündigung" angelegt - dadurch können dort nur ich und die Giscus-App neue Diskussionen eröffnen.

Über die [Seite von Giscus](https://giscus.app/) kann unter "Konfiguration" geprüft werden, ob alle Voraussetzungen erfüllt sind. Anschließend können auf der Seite die Optionen so gewählt werden, wie man sie gerne hätte, und sieht unter "giscus aktivieren" einen fertigen Code-Block, um Giscus einzubinden. Für uns sind jedoch nur die Werte interessant, die wir in der Konfiguration einpflegen. Im Template findet man den entsprechenden Abschnitt unter [config/_default/params.toml](https://github.com/CaiJimmy/hugo-theme-stack-starter/blob/master/config/_default/params.toml):

```
## Comments
[comments]
enabled = true
provider = "giscus"

[comments.giscus]
repo = ""
repoID = ""
category = ""
categoryID = ""
mapping = ""
lightTheme = ""
darkTheme = ""
reactionsEnabled = 1
emitMetadata = 0
```

Nach dem Ausfüllen der Werte und dem Erstellen eines Posts kann die Funktionalität direkt getestet werden.
