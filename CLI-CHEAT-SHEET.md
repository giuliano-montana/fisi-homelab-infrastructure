# Mein persönlicher Linux & Docker CLI Spickzettel

Dieses Dokument dient als meine persönliche Wissensdatenbank während meiner Vorbereitung auf die FISI-Umschulung. Hier halte ich alle Terminal-Befehle fest, die ich in der Praxis gelernt habe, um sie dauerhaft zu verinnerlichen.

---

## 1. Linux Verzeichnis- & Datei-Management (Die Basis)

Diese Befehle nutze ich, um mich im Terminal durch das Dateisystem meines Proxmox-LXC-Containers zu bewegen.

*   `mkdir -p <pfad>`
    *   **Bedeutung:** *Make Directory* (Ordner erstellen).
    *   **Erklärung:** Erstellt einen neuen, leeren Ordner. Der Schalter `-p` ist der Sicherheitsmodus: Er sorgt dafür, dass auch alle übergeordneten Ordner im Pfad automatisch mit erstellt werden, falls sie noch nicht existieren.
    *   **Beispiel:** `mkdir -p ~/homelab/nextcloud`
*   `pwd`
    *   **Bedeutung:** *Print Working Directory* (Arbeitsverzeichnis anzeigen).
    *   **Erklärung:** Spuckt den exakten Pfad aus, in dem mein Terminal sich gerade befindet. Lebenswichtig, um vor einem Docker-Start zu prüfen, in welchem Projektordner man steht.
*   `ls -la`
    *   **Bedeutung:** *List Extended* (Detaillierte Liste).
    *   **Erklärung:** Listet alle Dateien und Ordner im aktuellen Verzeichnis auf – inklusive der versteckten Systemdateien (die mit einem Punkt beginnen). Zeigt mir sofort, ob eine `docker-compose.yml` im Ordner existiert.   
*   `cd <pfad>`
    *   **Bedeutung:** *Change Directory* (Ordner wechseln).
    *   **Erklärung:** Bewegt mich virtuell in den angegebenen Ordner hinein. Das Zeichen `~` (Tilde) springt dabei immer direkt in mein Benutzer-Heimatverzeichnis.
    *   **Beispiel:** `cd ~/homelab/nextcloud`
*   `cp <quell_datei> <ziel_datei>`
    *   **Bedeutung:** *Copy* (Datei kopieren).
    *   **Erklärung:** Erstellt eine exakte Kopie einer Datei. Perfekt, um vor Änderungen ein Backup anzulegen.
    *   **Beispiel:** `cp docker-compose.yml docker-compose.yml.bak`
* `find <pfad> -name "<dateiname>" 2>/dev/null`
    * **Bedeutung:** Dateisystem nach bestimmten Dateien durchsuchen.
    * **Erklärung:** Durchsucht ab dem angegebenen Pfad (z. B. `/` für die ganze Festplatte) alle Ordner. `2>/dev/null` blendet Fehlermeldungen für Ordner aus, auf die man keine Zugriffsrechte hat.
    * **Beispiel:** `find / -name "config.php" 2>/dev/null`


---

## 2. Der Terminal-Texteditor (Nano)

Da ich auf Servern keine grafische Oberfläche (GUI) habe, editiere ich Konfigurationsdateien direkt im Terminal.

*   `nano <dateiname>`
    *   **Erklärung:** Öffnet oder erstellt eine Textdatei im eingebauten Linux-Texteditor *Nano*.
*   **Wichtige Tastatur-Shortcuts in Nano:**
    *   `Strg + K` = Löscht die aktuelle Zeile komplett (praktisch, um Dateien schnell zu leeren).
    *   `Strg + O` -> danach `Enter` drücken = Speichert die Datei ab.
    *   `Strg + X` = Beendet den Nano-Editor und kehrt zur normalen Kommandozeile zurück.

---

## 3. Docker Diagnose- & Verwaltungs-Befehle

Mit diesen Befehlen kontrolliere ich den Zustand der isolierten App-Boxen (Container) auf meinem System.

*   `docker ps`
    *   **Erklärung:** Zeigt eine Tabelle aller aktuell **laufenden** Docker-Container auf dem Server an. Die Spalte *NAMES* verrät mir die exakten Bezeichnungen der aktiven Microservices.
*   `docker compose up -d`
    *   **Erklärung:** Liest die `docker-compose.yml` im aktuellen Ordner aus, lädt fehlende Software-Images aus dem Docker Hub herunter und startet das gesamte Container-Team im Hintergrund. Das `-d` (*detached mode*) sorgt dafür, dass mein Terminal danach sofort wieder für neue Befehle frei wird.
*   `docker compose down`
    *   **Erklärung:** Stoppt und löscht alle Container, die in der lokalen `docker-compose.yml` definiert sind, **rückstandslos**. Der Server ist danach wieder komplett sauber, während die persistenten Daten in den gesharten Volumes sicher auf der Festplatte liegen bleiben.

* `docker inspect -f '{{ json .Mounts }}' <container_name>`
    * **Bedeutung:** Docker-Volumes und Speicherpfade auslesen.
    * **Erklärung:** Zeigt an, wo ein Container seine Daten auf dem echten Server (Host) dauerhaft speichert. Wichtig vor dem Löschen von Containern, um Datenverlust zu vermeiden.
    * **Beispiel:** `docker inspect -f '{{ json .Mounts }}' nextcloud-app`

* `docker exec -it <container_name> <befehl>`
    * **Bedeutung:** Befehle direkt *innerhalb* eines laufenden Containers ausführen.
    * **Erklärung:** Erlaubt es, in die isolierte Umgebung des Containers hineinzuspringen (z. B. mit `bash` oder `sh`), um dort direkt Diagnosen durchzuführen.
    * **Beispiel:** `docker exec -it nextcloud-app bash`

* `docker network create <netzwerk_name>`
  * **Bedeutung:** Docker-Netzwerk erstellen.
  * **Erklärung:** Erstellt ein eigenständiges, virtuelles Switch-Netzwerk, um Container aus unterschiedlichen Docker-Compose-Dateien miteinander zu verbinden.

* `docker network connect <netzwerk_name> <container_name>`
  * **Bedeutung:** Container im laufenden Betrieb an ein Netzwerk anbinden.
  * **Erklärung:** Verknüpft eine neue virtuelle Netzwerkkarte mit dem Container. Unverzichtbar, um bestehende Container nachträglich für den Reverse Proxy erreichbar zu machen.

* `docker network inspect <netzwerk_name>`
  * **Bedeutung:** Netzwerk-Details analysieren.
  * **Erklärung:** Spuckt die genauen IPs und verbundenen Container eines Netzwerks im JSON-Format aus. Perfekt für das Debugging auf Layer 3, um Konnektivität zu prüfen.

* `docker logs --tail <zeilen_anzahl> <container_name>`
  * **Bedeutung:** Letzte Log-Einträge anzeigen.
  * **Erklärung:** Begrenzt die Fehlerausgabe auf die neuesten Zeilen. Mit dem Zusatz `-f` (`--follow`) wird der Log-Stream live im Terminal verfolgt.
 
* `docker inspect <container_name> | grep -i -E "mysql_|mariadb_"`
   * **Bedeutung:** Umgebungsvariablen auslesen.
   * **Erklärung:** Filtert gezielt die geheimen System-Variablen (wie User und Passwörter) aus dem RAM_Blueprint des laufenden Containers, ohne dass man die docker-compose.yml suchen muss
 
* `docker exec <container_name> sh -c 'mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" <db_name>' > <ziel_pfad.sql>`
   * **Bedeutung:** Answendungskonsistenter Datenbank-Export.
   * **Erklärung:** Führt den Dump geschützt vor Sonderzeichen-Konflikten aus. Die Single-Quotes `'` verbieten der äußeren Linux-Shell das Mitlesen, während das `-p"$MYSQL_ROOT_PASSWORD` das Passwort direkt im sicheren Container-Speicher auflöst.

---

## 4. Wichtige Architektur-Erkenntnisse (Docker Networking & Storage)

*   **Das Ordner-Prinzip bei Compose:** Der Befehl `docker compose up -d` sucht *immer* nur im aktuellen Ordner nach der Konfigurationsdatei. Man kann problemlos mehrere `docker-compose.yml`-Dateien auf dem Server haben, solange sie in unterschiedlichen Verzeichnissen liegen. Sie behindern sich gegenseitig nicht.
*   **Named Volumes vs. Bind Mounts:** 
    *   *Named Volumes* (z. B. `- nextcloud-data:/var/www/html`) überlassen Docker die komplette Rechteverwaltung auf der Festplatte. Das ist bei unpriviligierten Proxmox-LXCs zwingend notwendig, um *Permission Denied*-Fehler (Rechtekonflikte durch das unpriviligierte UID-Mapping) bei Datenbanken und Web-Apps automatisch zu verhindern. Zudem müssen sie am Ende der Datei im globalen `volumes:`-Block deklariert (angemeldet) werden.
    *   *Bind Mounts* (z. B. `- ./data:/data`) verweisen auf einen festen, sichtbaren Pfad. Gut für einfache Konfigurationsordner, die man selbst als Administrator häufig händisch mit Nano bearbeiten möchte.
*   **Unattended Isolation (Passwörter im Code):** Docker-Container starten vollautomatisch ohne menschliche Eingabe. Daher müssen Umgebungsvariablen (`environment:`) wie Passwörter und Datenbanknamen im Vorfeld fest im Code definiert werden, damit die isolierten App-Boxen beim ersten Start automatisch die Tabellen anlegen und sich miteinander vernetzen können.
*   **Docker-internes DNS & Namensauflösung:** Innerhalb eines benutzerdefinierten Docker-Netzwerks (Bridge) läuft ein eigener DNS-Dienst. Container können sich dadurch direkt über ihren vergebenen `container_name` (z. B. `http://nextcloud-app:80`) gegenseitig anfunken. Man muss und sollte keine fehleranfälligen IP-Adressen der Container oder des Host-Systems fest verbauen.
*   **Das "Trusted Domains"-Prinzip (Sicherheitsfunktion):** Viele moderne Web-Apps (wie Nextcloud) besitzen einen integrierten Schutz vor unerlaubten Zugriffen. Sie akzeptieren Anfragen nur dann mit einem erfolgreichen Statuscode `200 OK`, wenn die aufgerufene Adresse (egal ob Domain wie `cloud.lan` oder interner Containername wie `nextcloud-app`) explizit in der Konfigurationsdatei (`config.php`) als vertrauenswürdig hinterlegt ist. Fehlt der Eintrag, blockiert die App mit einem Fehler `400 Bad Request`.
*   **Der I/O Delay Engpass (Storage Bottleneck):** Eine hohe CPU-Last von z.B. 9% bei gkleichzeitig hohem Load-Average (z.B. > 1.0) deutet auf einen I/O Wait hin. Die CPU-Kerne sind so schnell, dass sie sich langweilen und permanent darauf warten müssen, dass die mechanische Festplatte (HDD) die Datenblöcke liest oder schreibt
*   **Crash-Consistent vs. Application-Consistent Backups:** Reine Virtualisierungs-Snapshots spiegeln nur die Festplatte wider (Crash-Consistent). Da relationale Datenbanken Daten im RAM cachen, führt das Einspielen eines solchen Backups oft zu beschädigten Tabellen. Erst ein vorgeschalteter Anwendungs-Dump (`mysqldump`) sichert die Daten im RAM konsistent auf die Platte (Application-Consistent).
*   **Die Trennung von 'local' und 'local-lvm':** Proxmox trennt das System sauber auf: `local-lvm` ist roher, unformatierter Block-Speicher (Thin Pool), der extrem schnell für die virtuellen Laufwerke der LXCs genutzt wird, aber keine einzelnen Backup-Dateien speichern kann. `local` ist ein normales Verzeichnis-Dateisystem (ext4/XFS), das flexibel `.tar.zst` Archivdateien aufnimmt.

---

## 5. Proxmox Hypervisor- & Automatisierungs-Management

Diese Befehle steuern die Virtualisierungsebene auf dem Proxmox-Host (PVE) und die System-Automatisierung.

* `pct enter <vmid>`
   * **Erklärung:** Durchbricht die Virtualisierungs-Isolationswand vom Proxmox-Host aus und springt direkt ohne Passwort in die Root-Shell des angegebenen LXC-Containers.
   * **Beispiel:** `pct enter 101`

* `vzdump <vmid> --storage local --mode snapshot --compress zstd --prune-backups keep-last=3`
   * **Bedeutung:** Vollständiges Infrastruktur-backup eines Containers.
   * **Erklärung:** Erstellt einen Live-Snapshot des gesamten LXC-Containers. `--mode snapshot` sorgt für Null Ausfallzeit. ZSTD komprimiert die Daten massiv. `--prune-backups keep-last=3` verhindert dass Volllaufen der Festplatte, indem immer nur die neusten 3 Backups behalten werden.

* `pvesm status`
   * **Bedeutung:** Change Mode (Ausführungsrechte vergeben).
   * **Erklärung:** Listet alle aktiven Speicher-Partitionen auf dem Server (wie `local` und `local-lvm`) inkluxive deren Speicherplatzbelegung auf.

* `chmod +x <skriptpfad>`
   * **Bedeutung:** Change Mode (Ausführungsrechte vergeben).
   * **Erklärung:** Wandelt eine reine Textdatei in ein ausführbares Programm um, das vom Linux-Kernel als automatisiertes Skript gestartet werden darf.
   * **Beispiel:** `chmod +x /usr/local/bin/backup-pipeline.sh`
 
* `crontab -e`
   * **Bedeutung:** Chronological Table Editor.
   * **Erklärung:** Öffnet den systemweiten Kalender-Planer für automatisierte Hintergrundaufgaben. Jede Zeile definiert ein festes Zeitifenster für ein Skript.

* `/usr/local/bin/backup-pipeline.sh > /var/log/backup_pipeline.log 2>&1`
   * **Bedeutung:** Log-Redirection (Ausgabe umleiten) 
   * **Erklärung:** Der Operator `>` leitet alle Textausgaben und Fehlermedlungen (`2>&1`) des Skript in eine saubere Logdatei um, statt sie ins Nirgendwo zu senden. Perfekt für das morgendliche Backup-Auditing.
 
* `Die Cron-Timing-Matrix (0 3 * * *)`
   * Aufbau von links nach rechts: `Minute (0-59) | Stunde (0-23) | Tag des Monats (1-31) | Monat (1-12) | Wochentag (0-6)`
   * Das Zeichen `*` ist der Wildcard-Operator (Platzhalter für "jeden/alle").
   * `0 3 * * *` bedeutet übersetzt: Führe den Befehl in der 0. Minute der 3. Stunde aus, an jedem Tag des Monats, in jedem Monat, an jedem Wochentag (also täglich um Punkt 03:00 Uhr nachts).
