# Mein Homelab: Praxis-Grundlagen vor der FISI-Umschulung

Dieses Repository dokumentiert mein privates Homelab. Ich habe diese Infrastruktur aufgebaut, um vor dem Start meiner Umschulung zum FiSi im Februar 2027 echte Praxiserfahrung mit Linux, Containern und Netzwerk-Routing zu sammeln. 

Hier halte ich fest, was ich mir bisher selbst beigebracht habe und wie mein System aufgebaut ist.

---

## 1. Die Hardware & Ressourcen-Zuweisung
Da mein alter Laptop ungenutzt war, habe ich ihn reaktiviert, um ihn als Testumgebung für meine Umschulung zu nutzen:
* **Modell:** Acer Aspire V5
* **CPU:** 4 Kerne
* **Arbeitsspeicher:** 4 GB RAM
* **Hypervisor:** Proxmox VE
* **Gesamtspeicher:** 700 GB

### Ressourcen-Partitionierung:
Aufgrund der strikten 4 GB RAM-Grenze des Laptops wurde der Speicher präzise aufgeteilt, um dem Hypervisor genug Luft zu lassen:
* **Proxmox Host:** 1,0 GB RAM reserviert für das Hauptsystem.
* **LXC 1 (`pihole-dns`):** 512 MB RAM, 1 Core (Core-Infrastruktur).
* **LXC 2 (`srv-docker-apps`):** 2,5 GB RAM, 3 Cores, 500 GB Speicher (Docker-Umgebung).

---

## 2. Die Netzwerk- & Kapselungs-Architektur (Dienstetrennung)

Um die Stabilität meines Heimnetzwerks zu garantieren, ist das Lab konsequent nach dem Prinzip der **Dienstetrennung (Kapselung)** aufgebaut. Die Infrastruktur ist in zwei separate Proxmox LXC-Container (jeweils unpriviligiert für maximale Sicherheit) aufgeteilt:

### Bereich A: Core Infrastructure (`pihole-dns`)
* **Dienst:** Pi-hole (DNS & DHCP)
* **Kritikalität:** **Kritischer Single Point of Failure (SPOF)** für den gesamten Haushalt.
* **Sicherheitskonzept:** Da das Pi-hole sowohl die DNS-Auflösung als auch die IP-Adressvergabe (DHCP) für alle Geräte im Haus übernimmt, darf dieser Dienst niemals ausfallen. Er läuft als absolut isolierte Instanz, völlig unberührt von experimentellen Anwendungen.

### Bereich B: Application Infrastructure (`srv-docker-apps`)
* **Dienst:** Docker-Engine & Docker-Compose
* **Konzept:** Hier befindet sich meine "Spielwiese" für Web-Anwendungen. Sollte ein Docker-Container aufgrund von Fehlkonfigurationen oder Ressourcenmangel abstürzen, bleibt der LXC-Container für die Core-Infrastruktur (`pihole-dns`) davon komplett unberührt. Das Internet im Haus läuft stabil weiter.

**Die installierten Docker-Container:**
* **Nginx Proxy Manager:** Der einzige Container, der die Web-Ports (80/443) belegt. Er nimmt Anfragen an und leitet sie intern per Domainname weiter.
* **Uptime Kuma:** Mein Monitoring-Tool, das die Verfügbarkeit der Dienste überwacht.
* **Nextcloud & MariaDB:** Meine private Cloud-Umgebung, als Microservices (Applikation und Datenbank getrennt) aufgesetzt.

---

## 3. Design-Entscheidungen & Erkenntnisse (Lessons Learned)


## 🛠️ Mein Linux & Docker Spickzettel

Da ich im Terminal viele neue Befehle lerne, führe ich ein eigenständiges, fortlaufendes Logbuch. Um diese Readme übersichtlich zu halten, habe ich alle Befehle ausgelagert:

👉 **[Hier geht es zu meinem Linux & Docker CLI Spickzettel (CLI-CHEAT-SHEET.md)](CLI-CHEAT-SHEET.md)**
