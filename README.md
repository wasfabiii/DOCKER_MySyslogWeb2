# MySyslogWeb2

Dieser Projektordner enthält eine vollständig containerisierte Umgebung, um Syslog-Nachrichten zu sammeln, zu speichern und über eine Weboberfläche (Kibana) auszuwerten. Das bereitgestellte `docker-compose.yml`-Setup bündelt alle notwendigen Dienste (Elasticsearch, Logstash und Kibana), sodass keine zusätzlichen externen Abhängigkeiten notwendig sind.

## Architekturüberblick

| Dienst         | Aufgabe                                                                                     | Ports (Host) |
| -------------- | -------------------------------------------------------------------------------------------- | ------------ |
| Elasticsearch  | Persistente Speicherung der aufbereiteten Syslog-Ereignisse                                  | 9200         |
| Logstash       | Entgegennahme von Syslog über TCP/UDP 5514, Parsing und Weiterleitung an Elasticsearch       | 5514 (TCP/UDP) |
| Kibana         | Weboberfläche zur Visualisierung, Analyse und Filterung der gespeicherten Logdaten           | 5601         |

Die Logstash-Pipeline (`logstash/pipeline/syslog.conf`) nutzt Grok-Pattern, um klassische Syslog-Zeilen in strukturierte Felder aufzubrechen. Anschließend werden die Ereignisse im Index-Muster `syslog-YYYY.MM.dd` in Elasticsearch persistiert. Kibana greift auf denselben Elasticsearch-Knoten zu und stellt Dashboards, Discover-Ansichten sowie Such- und Filterfunktionen bereit.

## Voraussetzungen (Ubuntu 22.04 LTS)

1. **Docker Engine** installieren
   ```bash
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gnupg lsb-release
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   sudo apt-get install -y docker-ce docker-ce-cli containerd.io
   ```

2. **Docker Compose Plugin** installieren (falls nicht bereits enthalten)
   ```bash
   sudo apt-get install -y docker-compose-plugin
   ```

3. Den aktuellen Benutzer der `docker`-Gruppe hinzufügen (optional, ermöglicht Docker ohne `sudo`)
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```

## Deployment

1. Repository klonen oder Inhalt lokal bereitstellen.
   ```bash
   git clone <REPO-URL>
   cd DOCKER_MySyslogWeb2
   ```

2. Container-Stack im Hintergrund starten.
   ```bash
   docker compose up -d
   ```

3. Den Status der Dienste prüfen.
   ```bash
   docker compose ps
   docker compose logs -f logstash
   ```

   Warten Sie, bis `elasticsearch` den Healthcheck besteht und `logstash` meldet, dass die Pipeline initialisiert wurde.

4. Kibana über den Browser aufrufen: [http://localhost:5601](http://localhost:5601)

   > **Hinweis:** Beim ersten Aufruf muss ggf. das Index-Pattern `syslog-*` erstellt werden, um Daten in der *Discover*-Ansicht einzusehen.

## Tests & Verifikation

Führen Sie nach dem Deployment folgende Schritte aus, um den End-to-End-Datenfluss zu prüfen:

1. **Syntax-Check der Compose-Datei**
   ```bash
   docker compose config
   ```

2. **Beispiel-Syslog-Nachricht einspeisen** (ersetzt `<hostname>` nach Bedarf):
   ```bash
   logger --server 127.0.0.1 --port 5514 --tag demo "MySyslogWeb2 Testereignis"
   ```

3. **Verfügbarkeit von Elasticsearch prüfen**
   ```bash
   curl -fsSL http://localhost:9200/_cluster/health | jq
   ```

4. **Gespeicherte Nachricht über Elasticsearch abfragen**
   ```bash
   curl -fsSL 'http://localhost:9200/syslog-*/_search?q=demo&pretty'
   ```

   In der Ausgabe sollte das zuvor erzeugte Ereignis erscheinen.

5. **Kibana prüfen**
   - In Kibana unter *Stack Management → Kibana → Index Patterns* das Muster `syslog-*` anlegen.
   - Unter *Discover* nach dem Tag `demo` suchen und das Event verifizieren.

## Betrieb & Wartung

- **Skalierung:** Für höhere Datenvolumina können zusätzliche Logstash-Instanzen durch horizontales Skalieren des Dienstes bereitgestellt werden. Für elastische Cluster-Szenarien muss `elasticsearch` entsprechend erweitert werden.
- **Persistenz:** Der Elasticsearch-Datenordner wird per Docker-Volume `es-data` auf dem Host gespeichert und über Container-Neustarts hinweg beibehalten.
- **Konfiguration anpassen:**
  - Logstash-Pipeline anpassen: `logstash/pipeline/syslog.conf`
  - Logstash-Grundkonfiguration: `logstash/config/logstash.yml`
  - Zusätzliche Pipelines können als weitere `.conf`-Dateien abgelegt werden.
- **Shutdown:**
  ```bash
  docker compose down
  ```
  Optional mit Datenlöschung:
  ```bash
  docker compose down -v
  ```

## Fehlersuche

- Prüfen Sie die Logstash-Logs: `docker compose logs logstash`
- Elasticsearch-Status: `curl http://localhost:9200/_cluster/health?pretty`
- Netzwerkzugriff auf Port 5514 sicherstellen (Firewall ggf. anpassen):
  ```bash
  sudo ufw allow 5514/tcp
  sudo ufw allow 5514/udp
  sudo ufw allow 5601/tcp
  sudo ufw allow 9200/tcp
  ```

Mit diesem Setup erhalten Sie eine sofort einsatzbereite Logging-Plattform, die Syslog-Ereignisse entgegennimmt, speichert und über Kibana visuell analysierbar macht.
