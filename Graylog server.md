


> Written with [StackEdit](https://stackedit.io/).
> Ich helfe dir gerne beim Aufbau deines Log-Servers mit Graylog, Zoraxy als Reverse Proxy und Grafana zur Visualisierung – alles orchestriert mit Docker. Ein solider Plan! Die Kombination aus Graylogs zentraler Logverwaltung und Grafanas flexiblen Dashboards ist sehr effektiv .

Da die direkte Integration von Graylog als Datenquelle in Grafana in den neueren Versionen (ab Graylog 6) etwas komplexer geworden ist , konzentrieren wir uns in dieser Anleitung auf eine robuste und zukunftssichere Methode. Wir werden Grafana mit der zugrundeliegenden OpenSearch/Elasticsearch-Datenbank von Graylog verbinden.

Hier ist die detaillierte Schritt-für-Schritt-Anleitung.

### 📋 Vorbereitung

Stelle sicher, dass Docker und Docker Compose auf deinem System installiert sind. Lege ein neues Verzeichnis für das Projekt an, in dem alle Konfigurationsdateien gespeichert werden.

### 🚀 Schritt 1: Die `docker-compose.yml` erstellen

Erstelle im Projektverzeichnis eine Datei namens `docker-compose.yml` mit folgendem Inhalt. Diese Datei definiert alle notwendigen Dienste: MongoDB, OpenSearch (als Datenspeicher), Graylog, Grafana und Zoraxy.

```yaml
version: '3.8'

services:
  # MongoDB - Speichert Graylog Konfigurationen und Metadaten
  mongo:
    image: mongo:6.0
    container_name: graylog-mongo
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db
    networks:
      - graylog_network
      - zoraxy_network  # Optional: Falls du das MongoDB-Interface je proxyen wolltest

  # OpenSearch (oder Elasticsearch) - Speichert die eigentlichen Log-Nachrichten
  opensearch:
    image: opensearchproject/opensearch:2.11.0
    container_name: graylog-opensearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - plugins.security.disabled=true
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g" # Speicher anpassen
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - opensearch_data:/usr/share/opensearch/data
    networks:
      - graylog_network

  # Graylog Server
  graylog:
    image: graylog/graylog:6.0
    container_name: graylog-server
    restart: unless-stopped
    depends_on:
      - mongo
      - opensearch
    environment:
      # WICHTIG: Ändere diese Werte!
      - GRAYLOG_PASSWORD_SECRET=<dein-sicherer-zufallsstring>   # 'pwgen -N 1 -s 96'
      - GRAYLOG_ROOT_PASSWORD_SHA2=<sha2-hash-deines-passworts> # 'echo -n "deinPasswort" | sha256sum'
      - GRAYLOG_HTTP_BIND_ADDRESS=0.0.0.0:9000
      - GRAYLOG_HTTP_EXTERNAL_URI=http://127.0.0.1:9000/ # Wichtig für interne Redirects, später durch Domain ersetzen
      - GRAYLOG_ELASTICSEARCH_HOSTS=http://opensearch:9200
      - GRAYLOG_MONGODB_URI=mongodb://mongo:27017/graylog
    ports:
      # Graylog Web UI und API (werden später über Zoraxy gelenkt)
      - "127.0.0.1:9000:9000"   # NUR lokal gebunden
      # Eingänge (Inputs) für Logs
      - "12201:12201/tcp"       # GELF TCP
      - "12201:12201/udp"       # GELF UDP
      - "1514:1514/tcp"         # Syslog TCP
      - "1514:1514/udp"         # Syslog UDP
    volumes:
      - graylog_journal:/usr/share/graylog/data/journal
    networks:
      - graylog_network
      - zoraxy_network  # Damit Zoraxy auf Graylog zugreifen kann

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"   # NUR lokal gebunden
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - graylog_network  # Um auf OpenSearch zuzugreifen
      - zoraxy_network   # Damit Zoraxy auf Grafana zugreifen kann

  # Zoraxy Reverse Proxy
  zoraxy:
    image: zoraxydocker/zoraxy:latest
    container_name: zoraxy
    restart: unless-stopped
    ports:
      - "80:80"          # Standard HTTP
      - "443:443"        # Standard HTTPS
      - "8000:8000"      # Zoraxy Web UI (sollte ebenfalls hinter einem Proxy oder lokal bleiben)
    volumes:
      - zoraxy_config:/opt/zoraxy/config
      - zoraxy_data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock  # Erlaubt Zoraxy, andere Container zu "sehen" 
    environment:
      FASTGEOIP: "true"
    networks:
      - zoraxy_network

# Volumes für dauerhafte Datenspeicherung
volumes:
  mongo_data:
  opensearch_data:
  graylog_journal:
  grafana_data:
  zoraxy_config:
  zoraxy_data:

# Netzwerke
networks:
  graylog_network:
    driver: bridge
  zoraxy_network:
    driver: bridge
```

**Erklärung der wichtigsten Komponenten:**

*   **Datenbanken:** MongoDB und OpenSearch sind die Fundamente von Graylog .
*   **Graylog:** Empfängt Logs über verschiedene Ports (z.B. GELF 12201, Syslog 1514) . Die Ports für das Webinterface (9000) sind nur lokal gebunden (`127.0.0.1:`), da wir sie später sicher über Zoraxy verfügbar machen.
*   **Grafana:** Auch hier ist das Interface (Port 3000) nur lokal gebunden.
*   **Zoraxy:** Läuft ebenfalls in einem Container und mountet die Docker-Socket, um Dienste automatisch erkennen zu können . Er lauscht auf den Standard-Webports 80 und 443. Die Umgebungsvariable `FASTGEOIP` beschleunigt den Start .
*   **Umgebungsvariablen:** Die Platzhalter für Passwörter in der Graylog-Konfiguration müssen vor dem ersten Start durch echte Werte ersetzt werden .

### 🔑 Schritt 2: Notwendige Passwörter generieren

Bevor du den Stack startest, ersetze die Platzhalter im `docker-compose.yml`:

1.  **`GRAYLOG_PASSWORD_SECRET`**: Ein geheimer, langer String, der für Passwort-Hashes und API-Keys verwendet wird. Generiere ihn z.B. mit `pwgen -N 1 -s 96` (falls installiert) oder einem Online-Generator.
2.  **`GRAYLOG_ROOT_PASSWORD_SHA2`**: Der SHA-256-Hash deines Admin-Passworts. Ersetze `deinPasswort` mit deinem Wunsch-Passwort und führe den Befehl aus: `echo -n "deinPasswort" | sha256sum`. Der Output (ein langer Hex-String) kommt hier rein .

### 🚀 Schritt 3: Den Stack starten

Öffne ein Terminal im Verzeichnis mit deiner `docker-compose.yml` und führe aus:

```bash
docker compose up -d
```

Dies lädt alle Images herunter und startet die Container im Hintergrund. Die erste Ausführung kann einige Minuten dauern .

### ⚙️ Schritt 4: Zoraxy als Reverse Proxy konfigurieren

Jetzt richtest du Zoraxy ein, damit du über eine lesbare Adresse (wie `graylog.meine-domain.de` und `grafana.meine-domain.de`) auf deine Services zugreifen kannst, ohne dir die Ports merken zu müssen .

1.  **Zoraxy-UI aufrufen:** Öffne deinen Browser und gehe zu `http://<IP-deines-Servers>:8000`.
2.  **Initiales Setup:** Lege einen Benutzernamen und ein Passwort für die Zoraxy-Administration fest .
3.  **Proxy-Regel für Graylog erstellen:**
    *   Gehe im Zoraxy-Dashboard zu "HTTP Proxy" oder "Create Proxy Rules".
    *   **Matching Keyword / Domain:** Trage hier den Domain-Namen ein, den du für Graylog verwenden willst, z.B. `graylog.beispiel.de` .
    *   **Target IP / Domain with port:** Hier kommt der Zielcontainer und -port hin. Da Graylog im selben Docker-Netzwerk (`zoraxy_network`) läuft, kannst du seinen Containernamen als Hostname verwenden: `graylog-server:9000` . (Siehe `container_name` in der Compose-Datei).
    *   Klicke auf "Create Endpoint".
4.  **Proxy-Regel für Grafana erstellen:**
    *   Wiederhole den Vorgang für Grafana.
    *   **Domain:** `grafana.beispiel.de`
    *   **Target:** `grafana:3000` (da der Containername `grafana` ist).

**Optional - Automatisches HTTPS (Let's Encrypt):**
Zoraxy kann automatisch SSL-Zertifikate besorgen. Dafür müssen deine Domains (`graylog.beispiel.de`, `grafana.beispiel.de`) aber öffentlich erreichbar sein und auf die IP deines Servers zeigen. In den Zoraxy-Einstellungen (unter "Certificates") kannst du Let's Encrypt aktivieren .

### 📊 Schritt 5: Log-Quellen anbinden (Inputs in Graylog)

Jetzt sagst du Graylog, wie es Logs empfangen soll.

1.  **Graylog-UI aufrufen:** Gehe auf die von dir konfigurierte Domain (z.B. `http://graylog.beispiel.de`). Melde dich mit `admin` und dem von dir gewählten Passwort an.
2.  **Input einrichten:** Navigiere zu **System -> Inputs**.
3.  **Input auswählen:** Wähle z.B. `GELF UDP` aus dem Dropdown-Menü und klicke auf "Launch new input". GELF ist das native Graylog-Format und ideal für Docker-Container .
4.  **Input konfigurieren:** Vergib einen Namen (z.B. "Docker GELF UDP") und belasse den Port auf `12201`. Speichere die Konfiguration. Dein Graylog ist nun bereit, Logs zu empfangen .

**Docker-Container-Logs an Graylog senden:**
Um die Logs *anderer* Docker-Container an Graylog zu senden, musst du den Container mit dem GELF-Logging-Treiber starten. Hier ist ein Beispiel für einen "Ubuntu"-Container, der direkt in deinem Stack (im selben Netzwerk) läuft:

```bash
docker run -d \
  --name beispiel-container \
  --network zoraxy_network \
  --log-driver gelf \
  --log-opt gelf-address=udp://graylog-server:12201 \
  ubuntu bash -c 'while true; do echo "Hallo Graylog!"; sleep 10; done'
```

Diese Konfiguration funktioniert, weil der Container und der Graylog-Server sich im selben Docker-Netzwerk befinden .

### 📈 Schritt 6: Grafana mit den Log-Daten verbinden

Jetzt verknüpfst du Grafana mit der OpenSearch-Datenbank von Graylog, um mächtige Dashboards zu erstellen .

1.  **Grafana-UI aufrufen:** Gehe auf deine Grafana-Domain (z.B. `http://grafana.beispiel.de`). Der Standard-Login ist `admin` / `admin` (du wirst aufgefordert, das Passwort zu ändern).
2.  **Datenquelle hinzufügen:**
    *   Gehe zu **Configuration (Zahnrad) -> Data Sources -> Add data source**.
    *   Wähle **OpenSearch** (oder Elasticsearch, je nachdem, was du verwendet hast).
3.  **Datenquelle konfigurieren:**
    *   **URL:** `http://graylog-opensearch:9200` (der Containername von OpenSearch im internen Netzwerk).
    *   **Access:** Wähle "Server (default)".
    *   **OpenSearch details:** Index name: Gib `graylog_*` ein (Graylogs Indizes folgen meist diesem Muster).
    *   **Version:** Wähle die passende Version (z.B. 7.x+).
    *   Klicke auf "Save & Test". Wenn alles korrekt ist, erscheint eine Erfolgsmeldung.

Nun kannst du in Grafana ein neues Dashboard erstellen und die OpenSearch-Datenquelle verwenden, um Visualisierungen deiner Log-Daten zu bauen.

### 🔍 Wichtige Hinweise & Fehlerbehebung

*   **Daten-Persistenz:** Alle wichtigen Daten werden in Docker-Volumes gespeichert (`mongo_data`, `opensearch_data`, ...). Dadurch gehen deine Logs und Konfigurationen nicht verloren, wenn du die Container neu startest .
*   **Zoraxy und Docker:** Zoraxy läuft zwar in Docker, aber einige erweiterte Funktionen wie mDNS-Discovery funktionieren innerhalb von Containern nicht. Für den Standard-RP-Einsatz ist es aber völlig ausreichend .
*   **Graylog und Grafana:** Die Verbindung über OpenSearch ist der zuverlässigste Weg. Eine direkte Graylog-Datenquelle für Grafana ist aktuell nicht trivial einzurichten .
*   **Ressourcen:** Besonders OpenSearch/Elasticsearch kann speicherhungrig sein. Passe die `OPENSEARCH_JAVA_OPTS` in der `docker-compose.yml` bei Bedarf an deine Gegebenheiten an.
*   **Logs prüfen:** Falls etwas nicht funktioniert, überprüfe die Logs der Container: `docker compose logs graylog-server`.

Mit diesem Setup hast du eine hochflexible und skalierbare Logging-Umgebung, die du über Zoraxy zentral und sicher verwalten kannst. Viel Erfolg beim Ausprobieren! Wenn du auf spezifische Probleme stößt, kannst du gerne detaillierter nachfragen.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYzOTMzOTMxMV19
-->