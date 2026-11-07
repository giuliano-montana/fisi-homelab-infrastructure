# Changelog | 07. November 2026

**Thema: Automatisierung einer Backup-Pipeline (Anwendungs- und Infrastrukturkonsistent)**

## 1. Ausgangssituation & Zielsetzung 
Nachdem der Docker-Stack mit Nextcloud und MariaDB stabil lief, fehlte eine automatisierte, ausfallsichere Disaster-Recovery-Strategie. Ziel war es, ein tägliches Backup des Docker-LXC-Containers (`VM 101`) auf dem Proxmox-Host einzurichten, ohne für Ausfallzeiten der Dienste zu sorgen. Besondere Priorität hatte die absolute Datenintegrität der relationalen MariaDB-Datenbank, um Tabellenkorruption beim Wiederherstellen zu verhindern.

**Zusatz-Ziel (System-Härtung):** Da der LXC-Container standardmäßig nur über den allmächtigen `root`-Account verfügte, sollte im selben Zuge die Netzwerksicherheit erhöht werden. Um den Best-Practice-Standard der IT-Sicherheit (*Principle of Least Privilege*) zu erfüllen, sollte ein passwortgeschützter Standard-Benutzer für die tägliche Administration über SSH eingerichtet werden, um direkte Root-Logins über das Netzwerk vollständig zu unterbinden.

## 2. Aufgetretene Probleme & Technische Analyse
* **Problem 1: Crash-Inkonsistenz bei reinen Hypervisor-Snapshots**
    * **Symptom:** Ein reiner Proxmox-Snapshot friert nur die Festplattensektoren ein.
    * **Analyse:** Datenbanken (MarieDB) halten aktive Transaktionen und Indizes im flüchtigen RAM. Wird die Festplatte "mid-light" eingefrorem. fehlen diese Daten im Snapshot. Das Backup wäre im Ernstfall korrupt ("Crash-Inkonsistent").
* **Problem 2: Virtualisierungs-Ebenen verwechselt (Fehlermeldung: Permission Denied)**
    * **Symptom:** Der SSH-Loginversuch mit `ssh giuliano@192.168.2.3` vom Desktop schlug fehl.
    * **Analyse:** Der Benutzer wurde fälschlicherweise auf dem übergeordneten Proxmox-Host (PVE) erstellt. Der isolierte LXC-Container hatte diesen Benutzernamen nicht in seiner lokalen Systemdatenbank (`/etc/passwd`). Da der Container die Identität nicht kannte, verweigerte er den Zugriff. Zudem fehlte dem Container die administrative Zusatz-Software `sudo`, um Befehle zu eskalieren.
* **Problem 3: CLI-Sonderzeichen-Konflikt (Fehlermeldung: Access Denied 1045)**
    * **Symptom:** Der manuelle Testbefehl von `mysqldump`schlug fehl schlug fehl obwogl das Passwort richtig eingegeben wurde.
    * **Analyse:** Das Datenbank Passwort enthielt die Satzzeichen `.,`(Komma/Punkt). Der Linux-Kommandozeileninterpreter (Bash) interpretierte das Komma als Trennzeichen (Delimiter) und schnitt das Passwort beim Übergeben an den Docker-Container ab.

## 3. Durchgeführte Maßnahmen und Lösungen
1. **Benutzer-Härtung (Least Privilege):** Per `pct enter` den LXC-Container betreten und einen dedizierten User (`giuliano`) angelegt. Sudo nachinstalliert (`apt install sudo`) und den User der Sudo-Gruppe zugewiesen, um den unsicheren SSH-Root-Zugriff netzwerkweit komplett zu blockieren.
2. **Korrektur der Hostnamen-Auflösung:** Den internen LXC-Hostnamen permanent auf `srv-docker-apps` (vorher `monitoring`, da zu dem Zeitpunkt nur uptime-kuma installiert war) geändert, die `/etc/hosts` angepasst und damit den Sudo-Warnfehler behoben.
3. **Erstellung des anwendungskonsitenten DB-Dumps:** Lösung des Sonderzeichen-Konflikts durch das Kapseln der Ausführung in einer Docker-internen Shell (`sh -c`). Das Passwort wird dabei verschlüsselt als Umgebungsvariable (`$MYSQL_ROOT_PASSWORD`) direkt aus dem sicheren RAM-Speicher des Containers ausgelesen.
4. **Skript-Automatisierung & Cron-Scheduling:** Erstellung eines Master-Bash-Scripts (`/usr/local/bin/backup-pipeline.sh`) auf dem Proxmox-Host. Das Skript triggert erst den DB-Dump im Container und führt danach den ZSTD-komprimierten Host-Snapshot (`vzdump 101`) mit einer automatischen Pruning-Retention (`keep-last=3`) aus. Eingebunden in die System-Crontab für das nächtliche Backup-Fenster um 03:00 Uhr.

## 4. Ergebnis & Fazit
Die zweistufige Backup-Pipeline läuft vollständig jede Nacht um 03:00 Uhr. Der Speicherverbrauch im 100GB `local` -Verzeichnis ist durch das rollierende 3-Tage Fenster fest gedeckelt (ca. 14GB Gesammt-Footprint bei 4.58 GB pro ZSTD-Archiv). Durch die Kombination aus virtuellem `mysqldump`und physischem `vzdunp`ist die Wiederherstellung im Ernstfall zu 100% Konsistent und vor der Datenkorruption geschützt. 

# Changelog | 03. November 2026

**Thema: Netzwerk-Isolation von Uptime Kuma & Fehlerbehebung der Nextcloud-Überwachung**

## 1. Ausgangssituation & Zielsetzung
Nachdem Nextcloud erfolgreich im lokalen Netzwerk bereitgestellt wurde, sollte ein Überwachungstool (**Uptime Kuma**) eingerichtet werden. Ziel war es, den Status der Nextcloud kontinuierlich zu überwachen. Uptime Kuma lief zuvor isoliert im sogenannten `host`-Netzwerkmodus und sollte aus Sicherheits- und Stabilitätsgründen in das gemeinsame Docker-Netzwerk integriert werden, um direkt mit den anderen Containern zu kommunizieren.

## 2. Aufgetretene Probleme & Technische Analyse

* **Problem 1: Container-Namensauflösung fehlgeschlagen (Fehlermeldung: `ENOTFOUND`)**
    * **Symptom:** Nach dem Wechsel von Uptime Kuma in das Docker-Netzwerk konnte das Tool die lokale Domain `cloud.lan` nicht mehr finden.
    * **Analyse:** Uptime Kuma hatte innerhalb des isolierten Docker-Netzwerks keinen Zugriff auf den lokalen DNS-Server (Pi-hole). Dadurch konnte die Domain `cloud.lan` nicht in eine IP-Adresse übersetzt werden. Ein Versuch, den Nginx Proxy Manager direkt über dessen Containernamen anzusprechen, scheiterte, da zusätzliche HTTP-Header (Host-Header) zu Konflikten bei der internen Adresssuche des Tools führten.

* **Problem 2: Sicherheitsblockade durch Nextcloud (Fehlermeldung: `Status Code 400 - Bad Request`)**
    * **Symptom:** Als Uptime Kuma die Nextcloud direkt über den internen Containernamen (`http://nextcloud-app:80`) ansprach, verweigerte Nextcloud den Zugriff.
    * **Analyse:** Nextcloud besitzt eine integrierte Sicherheitsfunktion. Sie akzeptiert nur Anfragen von Domains oder Geräten, die explizit in ihrer Konfigurationsdatei als vertrauenswürdig hinterlegt sind. Da der interne Containername `nextcloud-app` dort fehlte, stufte Nextcloud die Anfrage als potenziell gefährlich ein und blockierte sie.

## 3. Durchgeführte Maßnahmen & Lösung (Fix)

1. **Netzwerk-Bereinigung & Daten-Absicherung:** Vor dem Umzug wurde geprüft, ob Uptime Kuma seine Daten dauerhaft auf der Festplatte des Servers speichert (über einen sogenannten Volume-Mount unter `/opt/uptime-kuma`). Der alte Container wurde sicher gelöscht und im selben Docker-Netzwerk wie Nextcloud neu gestartet, ohne dass Daten verloren gingen.
2. **Direkte Container-Verbindung eingerichtet:** In Uptime Kuma wurde die externe Domain durch den direkten internen Containernamen (`http://nextcloud-app:80`) ersetzt. Das Feld für zusätzliche HTTP-Header wurde komplett geleert, um DNS-Fehler im Container zu verhindern.
3. **Erweiterung der vertrauenswürdigen Domains (Trusted Domains):** Über das Terminal wurde die Konfigurationsdatei der Nextcloud (`config.php`) im Docker-Volume geöffnet. Der interne Containername `nextcloud-app` wurde manuell in die Liste der `trusted_domains` eingetragen.

## 4. Ergebnis & Fazit
Die Überwachung läuft nun perfekt. Uptime Kuma erreicht Nextcloud jetzt auf dem direktesten und sichersten Weg komplett intern über das Docker-Netzwerk. Nextcloud beantwortet die Anfragen erfolgreich mit dem Statuscode `200 OK` (Grün). Durch diese Architektur wird der externe Netzwerkverkehr geschont, und der Monitor bleibt selbst dann aktiv, wenn die lokale DNS-Auflösung (Pi-hole) einmal ausfallen sollte.


# Changelog | 29. Oktober 2026

## Thema: Nextcloud-Deployment & Reverse-Proxy-Integration

### 1. Ausgangssituation & Zielsetzung
Nach der gestrigen Basis-Installation von Nextcloud im Docker-Container sollte der Dienst heute produktiv genommen werden. Ziel war es, Nextcloud über die lokale Custom-Domain `http://cloud.lan` erreichbar zu machen. Als zentraler Einstiegspunkt (Inbound-Routing) dient der bereits vorhandene *Nginx Proxy Manager (NPM)*, die DNS-Auflösung übernimmt ein *Pi-hole*.

---

### 2. Aufgetretene Probleme & Technische Analyse

* **Problem 1: Endloses Laden (Gateway Timeout / 504)**
  * *Symptom:* Die Domain `cloud.lan` war nicht erreichbar.
  * *Analyse:* Ein Aufruf von `docker network inspect` zeigte, dass der NPM und der Nextcloud-Stack in isolierten Docker-Standard-Netzwerken liefen. Der Proxy konnte den Containernamen `nextcloud-app` physisch nicht auflösen. Zudem lag ein Fehlrouting auf Layer 3 vor: Im Pi-hole war für `cloud.lan` fälschlicherweise eine veraltete Host-IP hinterlegt, sodass Anfragen den Proxy gar nicht erst erreichten.

* **Problem 2: Fehlende Formatierung ("File not found" / Raw Text)**
  * *Symptom:* Nach Behebung des Netzwerk-Routings lud die Seite nur als unformatierter Text ohne CSS-Styles und Bilder.
  * *Analyse:* Das verwendete Image `nextcloud:fpm-alpine` liefert ausschließlich den nackten PHP-FPM-Code (Port 9000). Da FPM keine statischen Assets (CSS/JS) nativ ausliefern kann, kam es zu Darstellungsfehlern, weil der Proxy diese Dateien nicht isoliert abgreifen konnte.

* **Problem 3: Schreibrechte-Konflikt (Permission Denied)**
  * *Symptom:* Nextcloud blockierte den Start mit der Meldung *"Cannot write into 'config' directory!"*.
  * *Analyse:* Bedingt durch das UID/GID-Mapping im unprivilegierten Proxmox-LXC-Container stimmten die Besitzrechte der persistenten Named Volumes nach dem Image-Wechsel nicht mehr mit dem internen Webserver-User überein.

---

### 3. Durchgeführte Maßnahmen & Lösung (Fix)

1. **DNS-Korrektur:** Die IP-Adresse für `cloud.lan` im Pi-hole korrigiert und konsistent auf die IP des Reverse Proxys (`192.168.2.X`) umgebogen.
2. **Netzwerk-Kapselung:** Ein globales Docker-Netzwerk (`proxy-network`) per CLI erstellt und die Container `nginx-proxy-manager-app-1` und `nextcloud-app` nachträglich hineingehängt.
3. **Optimierung der Image-Wahl:** Umstellung in der `docker-compose.yml` vom komplexen FPM-Image auf das Standard-Web-Image (`nextcloud:latest`). Dieses bringt einen integrierten Apache-Webserver mit, der statische Assets nativ über Port 80 verarbeitet.
4. **Bereinigung der Dateirechte:** Vollständiger Reset der korrupten Volume-Rechte mittels `docker compose down -v` und anschließender Neuinitialisierung über `docker compose up -d`. Dadurch wurden die Named Volumes mit den korrekten Berechtigungen neu aufgebaut.

---

### 4. Ergebnis & Fazit
Die Nextcloud ist nun erfolgreich, stabil und vollformatig unter `http://cloud.lan` im lokalen Netzwerk erreichbar. Durch den Wechsel auf das Apache-Image konnte der fehleranfällige Custom-Nginx-Code im Advanced-Tab des NPM komplett entfernt werden. Das System läuft stabil auf der unprivilegierten LXC-Infrastruktur und schont die restriktiven Hardware-Ressourcen (4 GB RAM) des Hosts optimal.
