# Chngelog | 29. Oktober 2026

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
