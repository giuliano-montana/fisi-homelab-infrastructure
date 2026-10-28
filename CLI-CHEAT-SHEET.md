# 🛠️ Mein persönlicher Linux & Docker CLI Spickzettel

Dieses Dokument dient als meine persönliche Wissensdatenbank während meiner Vorbereitung auf die FISI-Umschulung. Hier halte ich alle Terminal-Befehle fest, die ich in der Praxis gelernt habe, um sie dauerhaft zu verinnerlichen.

---

## 1. Linux Verzeichnis- & Datei-Management (Die Basis)

Diese Befehle nutze ich, um mich im Terminal durch das Dateisystem meines Proxmox-LXC-Containers zu bewegen.

*   `mkdir -p <pfad>`
    *   **Bedeutung:** *Make Directory* (Ordner erstellen).
    *   **Erklärung:** Erstellt einen neuen, leeren Ordner. Der Schalter `-p` ist der Sicherheitsmodus: Er sorgt dafür, dass auch alle übergeordneten Ordner im Pfad automatisch mit erstellt werden, falls sie noch nicht existieren.
    *   **Beispiel:** `mkdir -p ~/homelab/nextcloud`
*   `cd <pfad>`
    *   **Bedeutung:** *Change Directory* (Ordner wechseln).
    *   **Erklärung:** Bewegt mich virtuell in den angegebenen Ordner hinein. Das Zeichen `~` (Tilde) springt dabei immer direkt in mein Benutzer-Heimatverzeichnis.
    *   **Beispiel:** `cd ~/homelab/nextcloud`
*   `cp <quell_datei> <ziel_datei>`
    *   **Bedeutung:** *Copy* (Datei kopieren).
    *   **Erklärung:** Erstellt eine exakte Kopie einer Datei. Perfekt, um vor Änderungen ein Backup anzulegen.
    *   **Beispiel:** `cp docker-compose.yml docker-compose.yml.bak`

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

