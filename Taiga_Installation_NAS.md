# Taiga (Docker) auf Hochschulserver mit Rootless-Docker, NFS-Storage und Host-Reverse-Proxy

Diese Anleitung beschreibt die vollständige Einrichtung einer stabilen Taiga-Instanz in Docker auf einem Hochschulserver. Sie berücksichtigt typische Hochschulrandbedingungen (Rootless-Docker, NFS-Ablage, VPN-Erreichbarkeit) und enthält alle erforderlichen Shell-Befehle sowie begründete Architekturentscheidungen.
Zusätzlich ist jetzt dokumentiert, wie man den in der Praxis häufig auftretenden Fehler

> „Hoppla, etwas ist schief gelaufen, deine Änderungen wurden nicht gespeichert“

behebt, **wenn die Änderung in Wirklichkeit gespeichert wurde**, im Backend aber ein 500er wegen RabbitMQ/Celery auftritt.

---

## 0) Zielbild & Eckdaten

**Zielbild**

* Taiga läuft als Container-Stack (Front/Back/Events/Protected/DB).
* Docker läuft **rootless** im Benutzerkontext (geringeres Risiko auf Multi-User-Hosts).
* Projektdateien (Uploads/Media/Static, Backups, Logs) liegen auf einem **NFS-Share**.
* Erreichbarkeit intern über Host-Nginx (HTTP) → sauberer Reverse-Proxy ohne SSH-Tunnels.
* **Neu:** Der Stack enthält zwei RabbitMQ-Instanzen aus den offiziellen Taiga-Compose-Dateien:

  * `taiga-async-rabbitmq` → für Celery/Timeline/Benachrichtigungen
  * `taiga-events-rabbitmq` → für das Events-/Websocket-System
    Das ist wichtig für die Fehlersuche weiter unten.

**Beispielumgebung (bei Bedarf anpassen)**

* OS: Ubuntu 24.04 (VM)
* Hostname/DNS (im Hochschul-VPN): `apm.hs-emden-leer.de`
* NFS-Mount: `/mnt/nas`
* Taiga-Daten auf NFS: `/mnt/nas/taiga`

**Warum so?**

* **Rootless-Docker**: Minimiert Privilegien, reduziert Angriffsfläche auf geteilten Systemen.
* **Host-Nginx** als Reverse-Proxy: vermeidet Container-DNS-Besonderheiten und Ports ins LAN; stabil bei Rootless-Setups.
* **NFS**: zentrale, snapshot-fähige Ablage; einfaches Backup/Restore.
* **Neu:** Explizite Celery-/RabbitMQ-Anbindung dokumentiert, weil Taiga viele Dinge *nach* dem Speichern in Tasks schiebt. Wenn RabbitMQ nicht erreichbar ist, wirkt das Frontend fehlerhaft, obwohl die eigentliche Änderung schon in der DB/NFS liegt.

---

## 1) Rootless-Docker installieren & aktivieren

> Als normaler Benutzer (z. B. `lohdens`) arbeiten; nur dort `systemctl --user …` verwenden.

```bash
# Pakete (falls noch nicht vorhanden)
# sudo apt-get update
# sudo apt-get install -y docker-ce docker-ce-cli docker-ce-rootless-extras

# Rootless initial einrichten
dockerd-rootless-setuptool.sh install

# Linger aktivieren, damit User-Docker auch ohne Login startet
sudo loginctl enable-linger <Nutzername>

# User-Docker starten & aktivieren
systemctl --user enable --now docker
systemctl --user status docker   # mit 'q' wieder verlassen

# Komfort: Umgebungsvariablen setzen
echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///run/user/1001/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

**Warum Rootless-Docker?**
Container-Runtime läuft im Userkontext; selbst bei Container-Breakouts bleibt die Auswirkung auf Benutzerrechte begrenzt. Ideal für Hochschulserver mit wechselnden Admins/Studierenden.

---

## 2) NFS-Berechtigungen korrekt setzen (rootless-tauglich)

```bash
# 2.1 Mountpunkt "durchlaufbar" machen (x-Bit), ohne weltweites Lesen
sudo chmod 0711 /mnt/nas

# 2.2 Taiga-Root auf NFS anlegen
sudo mkdir -p /mnt/nas/taiga
sudo chown root:users /mnt/nas/taiga
sudo chmod 2770 /mnt/nas/taiga

# 2.3 Unterordner für Daten
sudo mkdir -p /mnt/nas/taiga/{media,static,backups,logs}
sudo chgrp users /mnt/nas/taiga/{media,static,backups,logs}
sudo chmod 2775 /mnt/nas/taiga/{media,static,backups,logs}

# 2.4 Schreibprobe als normaler User (ohne sudo!)
echo ok > /mnt/nas/taiga/media/.w && rm /mnt/nas/taiga/media/.w
echo ok > /mnt/nas/taiga/static/.w && rm /mnt/nas/taiga/static/.w
```

**Wichtiger Hinweis zu NFS:**
Bei üblichen **`root_squash`**-Exports erscheinen Root-Vorgänge als anonymer User. Deshalb niemals mit `sudo` in die NFS-Verzeichnisse schreiben; Owner muss der Rootless-Benutzer sein, damit Container sauber lesen/schreiben können.

---

## 3) Taiga-Docker-Projektverzeichnis vorbereiten

```bash
# Arbeitsverzeichnis
cd ~
mkdir taiga && cd taiga

# "volumes" anlegen und auf NFS verlinken
mkdir -p volumes
ln -sfn /mnt/nas/taiga/media  volumes/media
ln -sfn /mnt/nas/taiga/static volumes/static
ln -sfn /mnt/nas/taiga/logs   volumes/logs
```

**Compose-Dateien ablegen**
Die offiziellen `docker-compose.yml` und `docker-compose-inits.yml` von Taiga im Projektordner ablegen.
(Versionen aus der offiziellen Dokumentation verwenden; in dieser Anleitung wird ein Override ergänzt.)

### 3.1 Compose-Override (Ports, Healthcheck, Mount-Fix, Broker)

```yaml
# Datei: docker-compose.override.yml
services:
  taiga-db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -q -h /var/run/postgresql || pg_isready -q -h 127.0.0.1 || exit 1"]
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 10s

  # Services nur lokal veröffentlichen (Reverse-Proxy auf dem Host)
  taiga-front:
    ports:
      - "127.0.0.1:8080:80"

  taiga-back:
    volumes:
      # Mount-Fix: beide möglichen Zielpfade im Image mit NFS verbinden
      - ./volumes/media:/srv/taiga-back/media
      - ./volumes/media:/taiga-back/media
      - ./volumes/static:/srv/taiga-back/static
      - ./volumes/static:/taiga-back/static
    ports:
      - "127.0.0.1:8000:8000"
    # NEU: Celery-Broker explizit auf den von Taiga mitgebrachten RabbitMQ setzen
    # (Name des Services in den offiziellen Compose-Dateien: taiga-async-rabbitmq)
    environment:
      - CELERY_BROKER_URL=amqp://taiga:taiga@taiga-async-rabbitmq:5672/taiga

  taiga-protected:
    volumes:
      - ./volumes/media:/srv/taiga-back/media:ro
      - ./volumes/media:/taiga-back/media:ro
    ports:
      - "127.0.0.1:8003:8003"

  taiga-events:
    ports:
      - "127.0.0.1:8888:8888"
```

**Warum Mount-Fix?**
Zwischen Taiga-Image-Versionen variierten Pfade für Uploads/Static (`/srv/taiga-back/...` **oder** `/taiga-back/...`). Durch **beide Bind-Mounts** wird verhindert, dass Uploads im Container verbleiben und auf NFS fehlen.

**Warum der Broker-Eintrag?**
Das Taiga-Backend verschickt nach erfolgreichen Schreibvorgängen (z. B. Datei hochladen, Wiki speichern) oft noch Celery-Tasks an RabbitMQ (`taiga-async-rabbitmq`). Wenn das Backend nicht weiß, wo der Broker ist, oder der Broker andere Zugangsdaten hat, ist der Upload zwar in der DB/NFS, aber der Request endet mit 500 → das Frontend zeigt „Hoppla…“.

---

## 4) Taiga-Umgebungsvariablen (Domain/URLs) korrekt setzen

In den **Env-Sektionen** der offiziellen Compose-Dateien (Front/Back/Events) **Domain & URLs** auf Host anpassen (ohne IP-Bindung). Für reinen VPN-HTTP-Betrieb:

```yaml
# Beispielauszug: taiga-back (environment:)
    environment:
      - TAIGA_SITES_DOMAIN=apm.hs-emden-leer.de
      - TAIGA_SITES_SCHEME=http
      # wenn hier weitere envs stehen, bleibt der CELERY_BROKER_URL-Eintrag aus dem Override bestehen

# Beispielauszug: taiga-events (environment:)
    environment:
      - TAIGA_SITES_DOMAIN=apm.hs-emden-leer.de
      - TAIGA_SITES_SCHEME=http

# Beispielauszug: taiga-front (environment:)
    environment:
      - TAIGA_URLS__FRONT=http://apm.hs-emden-leer.de
      - TAIGA_URLS__API=http://apm.hs-emden-leer.de/api/v1/   # WICHTIG: mit abschließendem Slash
      - TAIGA_URLS__EVENTS=ws://apm.hs-emden-leer.de/events
      - TAIGA_URL=http://apm.hs-emden-leer.de
      - TAIGA_WEBSOCKETS_URL=ws://apm.hs-emden-leer.de
```

**Warum der Slash in `TAIGA_URLS__API`?**
Ohne abschließenden Slash erzwingt Django/DRF einen Redirect (301) → Browser/Proxy-Buffering/Timeouts führen zu „Endlos-Spinnern“ oder „Speichern fehlgeschlagen“, obwohl Daten gespeichert wurden.

---

## 5) Stack starten & Admin-Nutzer anlegen

```bash
# 5.1 Starten
docker compose up -d --remove-orphans
docker compose ps

# 5.2 Superuser einmalig anlegen
docker compose -f docker-compose.yml -f docker-compose-inits.yml \
  run --rm taiga-manage createsuperuser
```

**Schnelltests (direkt auf der VM)**

```bash
curl -sS -o /dev/null -w 'front  %{http_code}\n'  http://127.0.0.1:8080/
curl -sS -o /dev/null -w 'api    %{http_code}\n'  http://127.0.0.1:8000/api/v1/
curl -sS -o /dev/null -w 'prot   %{http_code}\n'  http://127.0.0.1:8003/
curl -m 1 -sS -o /dev/null -w 'events %{http_code}\n' http://127.0.0.1:8888/events
```

**Upload-Verifikation**

```bash
# Nach einem Test-Upload in Taiga
ls -l /mnt/nas/taiga/media | tail
```

---

## 6) Reverse-Proxy auf dem Host (Nginx, HTTP)

```bash
# Nginx installieren
sudo apt-get update
sudo apt-get install -y nginx

# Default-Site (insb. IPv6-Bind) deaktivieren
sudo rm -f /etc/nginx/sites-enabled/default

# Site-Config anlegen
sudo tee /etc/nginx/sites-available/taiga.conf >/dev/null <<'NGINX'
server {
  listen 80;
  server_name apm.hs-emden-leer.de;

  client_max_body_size 100M;

  # Frontend (SPA)
  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_read_timeout 300s;
  }

  # Backend API
  location /api/ {
    proxy_pass http://127.0.0.1:8000/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_read_timeout 300s;
  }

  # Events (SSE/WebSocket)
  location /events {
    proxy_pass http://127.0.0.1:8888/events;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 300s;
  }

  # Geschützte Medien
  location /media/ {
    proxy_pass http://127.0.0.1:8003/media/;
    proxy_set_header Host $host;
    proxy_read_timeout 300s;
  }

  # Statische Dateien (Fallback)
  location /static/ {
    proxy_pass http://127.0.0.1:8080/static/;
    proxy_set_header Host $host;
    proxy_read_timeout 300s;
  }
}
NGINX

# Aktivieren & reloaden
sudo ln -sf /etc/nginx/sites-available/taiga.conf /etc/nginx/sites-enabled/taiga.conf
sudo nginx -t && sudo systemctl reload nginx
```

**Host-seitige Funktionsprüfung**

```bash
curl -sS -o /dev/null -w 'host     %{http_code}\n' -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/
curl -sS -o /dev/null -w 'host-api %{http_code}\n' -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/api/v1/
```

**Warum Host-Nginx statt „Gateway“-Container?**
Bindet klar an `127.0.0.1`, vermeidet Container-DNS/Overlay-Netze, spielt gut mit Rootless-Ports und System-Firewall. Bei IPv6-Problemen wird die Default-Site entfernt, um Konflikte auf `::80` zu vermeiden.

---

## 7) Typische Stolpersteine & schnelle Checks

**A) API liefert `000/502`**

```bash
docker compose port taiga-back 8000
ss -ltn | grep ':8000' || echo 'nichts auf 8000'
docker compose logs --tail=50 taiga-back
```

> Falls Rootless-Ports kollidieren: Back/Events auf alternative Ports legen, z. B. `127.0.0.1:18000:8000` und `127.0.0.1:18888:8888` (Compose-Override + Nginx-Ziele anpassen), dann `docker compose up -d` und `sudo systemctl reload nginx`.

**B) NFS „chown: Operation not permitted“**

* Auf NFS mit `root_squash` normal. Entscheidend ist die **Schreibprobe als normaler Benutzer** (siehe §2.4).
* **Nie** mit `sudo` in `/mnt/nas/taiga/...` arbeiten.

**C) Postgres „unhealthy“**

* Healthcheck im Override sorgt dafür, dass `taiga-back` erst startet, wenn DB bereit ist.
* Keine Bind-Mounts für die DB verwenden (Standard-Named-Volume der Compose-Dateien belassen).

**D) Endlos-Spinner / „Speichern fehlgeschlagen“**

* Meist **fehlender Slash** in `TAIGA_URLS__API`. In `conf.json` muss `.../api/v1/` erscheinen.
  Prüfung:

  ```bash
  curl -sS -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/conf.json | jq .api,.eventsUrl
  ```

**E) Redirect-Prüfung**

```bash
curl -sSI -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/api/v1/projects      | head -n1  # darf 301 sein
curl -sSI -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/api/v1/projects/     | head -n1  # 200/401/403, kein 301/302
```

**F) Live-Logs beim UI-Klick**

```bash
docker compose logs -f taiga-back taiga-events
# Erwartung bei Projekterstellung: POST /api/v1/projects/ -> 201
```

### **G) 500 beim Hochladen / Wiki-Speichern trotz erfolgreicher Speicherung (RabbitMQ/Celery)**

**Symptom**

* Im Browser: „Hoppla, etwas ist schief gelaufen, deine Änderungen wurden nicht gespeichert“
* In Wirklichkeit: Anhang / Wiki-Eintrag / Änderung ist da.
* Im `taiga-back`-Log:

  ```text
  amqp.exceptions.AccessRefused: (0, 0): (403) ACCESS_REFUSED - Login was refused using authentication mechanism PLAIN.
  ```
* Nginx und `conf.json` sind korrekt.

**Ursache**
Taiga speichert zuerst in der DB und versucht **danach**, eine Celery-Task ins Message-Queue-System zu legen (Timeline, Notifications usw.). Der Taiga-Container heißt z.B. `taiga-back`, der RabbitMQ dazu heißt in der Standard-Compose aber `taiga-async-rabbitmq`. Wenn der Broker andere Zugangsdaten hat (oder beim ersten Start schon Daten angelegt hat), greifen spätere `RABBITMQ_DEFAULT_USER/PASS/VHOST` nicht mehr → Taiga bekommt beim Task-Publish 403 → der HTTP-Request endet mit 500 → UI meckert.

**So behebst du es:**

1. **RabbitMQ stoppen**

   ```bash
   cd ~/taiga-docker
   docker compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.rabbit.yml \
     stop taiga-async-rabbitmq
   ```

2. **Container wirklich entfernen** (sonst bleibt das Volume „in use“)

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.rabbit.yml \
     rm -f taiga-async-rabbitmq
   ```

3. **altes RabbitMQ-Volume löschen** (Name aus `docker inspect ...` nehmen, hier Beispiel):

   ```bash
   docker volume rm taiga-docker_taiga-async-rabbitmq-data
   ```

   > Hintergrund: Beim allerersten Start hat RabbitMQ seine DB im Volume angelegt. Ab dann ignoriert es spätere `RABBITMQ_DEFAULT_*`-Variablen. Durch das Löschen des Volumes erzwingst du einen „frischen“ Start, bei dem die Env-Variablen wieder gelten.

4. **RabbitMQ + Back mit den gewünschten Env-Variablen neu starten**
   (das ist die Datei, die wir zuvor angelegt haben – hier nochmal zur Vollständigkeit)

   ```bash
   # ~/taiga-docker/docker-compose.rabbit.yml
   cat > docker-compose.rabbit.yml <<'EOF'
   services:
     taiga-async-rabbitmq:
       environment:
         RABBITMQ_DEFAULT_USER: taiga
         RABBITMQ_DEFAULT_PASS: taiga
         RABBITMQ_DEFAULT_VHOST: taiga

     taiga-back:
       environment:
         CELERY_BROKER_URL: amqp://taiga:taiga@taiga-async-rabbitmq:5672/taiga
   EOF
   ```

   Dann:

   ```bash
   docker compose \
     -f docker-compose.yml \
     -f docker-compose.override.yml \
     -f docker-compose.rabbit.yml \
     up -d --force-recreate taiga-async-rabbitmq taiga-back
   ```

5. **prüfen, ob das Backend die Broker-URL hat**

   ```bash
   docker compose \
     -f docker-compose.yml \
     -f docker-compose.override.yml \
     -f docker-compose.rabbit.yml \
     exec taiga-back sh -c 'env | grep -i broker || env | grep -i celery'
   ```

   Erwartung:

   ```text
   CELERY_BROKER_URL=amqp://taiga:taiga@taiga-async-rabbitmq:5672/taiga
   ```

6. **Upload erneut testen** und parallel Logs anschauen:

   ```bash
   docker compose \
     -f docker-compose.yml \
     -f docker-compose.override.yml \
     -f docker-compose.rabbit.yml \
     logs -f taiga-back
   ```

   → Wenn der 403 weg ist, verschwindet auch die „Hoppla…“-Meldung.

**Merksatz:**
Wenn Taiga „Hoppla…“ sagt, obwohl die Datei da ist, liegt es fast immer **nach** dem eigentlichen Speichervorgang – sehr häufig an RabbitMQ/Celery. Dann ist nicht Nginx kaputt, sondern der Broker.

---

## 8) Backups

```bash
# DB-Backup (in NFS-Backups ablegen)
docker compose exec -T taiga-db \
  pg_dump -U taiga taiga | gzip > /mnt/nas/taiga/backups/taiga-$(date +%F_%H%M).sql.gz

# Media/Static liegen bereits auf NFS (separat per Snapshot/Backup sichern)
```

---

## 9) Optional: HTTPS (Let’s Encrypt über Host-Nginx)

```bash
# Firewall ggf. öffnen (optional)
# sudo ufw allow 'Nginx Full'

# Certbot
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d apm.hs-emden-leer.de
# Automatische Erneuerung wird als Cron/Timer eingerichtet
```

Die Certbot-Routine ergänzt die `server`-Blöcke mit `listen 443 ssl;` und passenden Zertifikatspfaden.
**Hinweis:** Für WebSockets/Events sind keine weiteren Anpassungen nötig; Nginx setzt die Upgrade-Header bereits.

---

## 10) Kurz-Checkliste (Neuinstallation in wenigen Minuten)

```bash
# 1) Rootless-Docker (einmalig)
dockerd-rootless-setuptool.sh install && sudo loginctl enable-linger <Nutzername>
systemctl --user enable --now docker

# 2) NFS vorbereiten
sudo chmod 0711 /mnt/nas
sudo mkdir -p /mnt/nas/taiga/{media,static,backups,logs}
sudo chgrp users /mnt/nas/taiga /mnt/nas/taiga/{media,static,backups,logs}
sudo chmod 2770 /mnt/nas/taiga && sudo chmod 2775 /mnt/nas/taiga/{media,static,backups,logs}

# 3) Projektgerüst
mkdir -p ~/taiga/volumes && cd ~/taiga
ln -sfn /mnt/nas/taiga/media volumes/media
ln -sfn /mnt/nas/taiga/static volumes/static
ln -sfn /mnt/nas/taiga/logs   volumes/logs
# docker-compose.yml & docker-compose-inits.yml ablegen
# docker-compose.override.yml (aus §3.1) anlegen

# 4) Env-Variablen (Domain/URLs) wie in §4 setzen
#    + CELERY_BROKER_URL wie in §3.1 setzen

# 5) Start & Admin
docker compose up -d --remove-orphans
docker compose -f docker-compose.yml -f docker-compose-inits.yml run --rm taiga-manage createsuperuser

# 6) Host-Nginx
sudo apt-get install -y nginx
sudo rm -f /etc/nginx/sites-enabled/default
# taiga.conf aus §6 anlegen
sudo nginx -t && sudo systemctl reload nginx
```

---

## Anhang A: Gateway-Nginx im Compose (Alternative zu Host-Nginx)

*(unverändert, wie in deiner ursprünglichen Anleitung – geeignet, wenn man keinen Host-Nginx nutzen will oder alles in Docker kapseln möchte)*

```nginx
# Datei im Gateway-Container (z.B. mounted): /etc/nginx/conf.d/taiga.conf
upstream taiga_back   { server taiga-back:8000;   keepalive 32; }
upstream taiga_front  { server taiga-front:80;    keepalive 8;  }
upstream taiga_events { server taiga-events:8888; keepalive 8;  }

server {
  listen 9000;
  server_name _;

  client_max_body_size 100M;

  location = /conf.json {
    proxy_pass http://taiga_front/conf.json;
    proxy_set_header Host $host;
    proxy_http_version 1.1; proxy_set_header Connection "";
    proxy_read_timeout 300s; proxy_buffering off;
  }

  location / {
    proxy_pass http://taiga_front/;
    proxy_set_header Host $host;
    proxy_http_version 1.1; proxy_set_header Connection "";
    proxy_read_timeout 300s; proxy_buffering off;
  }

  location /api/ {
    proxy_pass http://taiga_back/api/;                     # immer mit Slash!
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1; proxy_set_header Connection "";
    proxy_request_buffering off;
    proxy_buffering off;
    proxy_read_timeout 600s;
  }

  location /events {
    proxy_pass http://taiga_events/events;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 600s;
    proxy_buffering off;
  }

  location /media/ {
    proxy_pass http://taiga-protected:8003/media/;
    proxy_set_header Host $host;
    proxy_http_version 1.1; proxy_set_header Connection "";
    proxy_read_timeout 300s; proxy_buffering off;
  }

  location /static/ {
    proxy_pass http://taiga_front/static/;
    proxy_set_header Host $host;
    proxy_http_version 1.1; proxy_set_header Connection "";
    proxy_read_timeout 300s; proxy_buffering off;
  }
}
```

---

## Anhang B: Diagnose-Kommandos (kompakt)

```bash
# Compose-Status & Logs
docker compose ps
docker compose logs --tail=200 taiga-back

# DB-Migrationszähler (Erwartungswert je nach Version)
docker compose exec -T taiga-db psql -U taiga -d taiga -c "select count(*) from django_migrations;"

# Ports (lokal veröffentlicht)
curl -sS -w 'front %{http_code}\n'  http://127.0.0.1:8080/ -o /dev/null
curl -sS -w 'api   %{http_code}\n'  -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/api/v1/ -o /dev/null
curl -sS -w 'evts  %{http_code}\n'  -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/events -o /dev/null
```

**Neu dazu merken:**
Wenn im `taiga-back`-Log **nur** der RabbitMQ-403 steht, aber vorne alles gut aussieht, sofort §7.G ausführen (Container entfernen → Volume löschen → RabbitMQ mit Defaults neu starten → Back neu starten). Dann verschwindet auch das Taiga-Frontend-Popup.

---

## Anhang C: Troubleshooting: 502 Bad Gateway (nginx/Ubuntu)

Tritt beim Aufruf von Taiga ein **502 Bad Gateway** auf, liegt das in dieser Umgebung fast immer daran, dass entweder der Host-Nginx keinen Container erreicht oder der rootless-Docker-Stack (Taiga) nicht sauber läuft.

1. **Host-Nginx prüfen und neu laden**
   Erst sicherstellen, dass die Reverse-Proxy-Konfiguration gültig ist:

   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

   Wenn hier Fehler kommen, zuerst die Nginx-Config korrigieren.

2. **Rootless-Docker neu starten**
   Wenn der User-Docker-Daemon hängt, sieht Nginx die Container nicht:

   ```bash
   systemctl --user restart docker
   export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
   ```

3. **Taiga-Stack neu starten**
   Danach den kompletten Stack einmal geordnet neu hochfahren:

   ```bash
   cd ~/taiga-docker
   docker compose down --remove-orphans
   docker compose up -d
   docker compose ps
   ```

4. **Dienste lokal testen**
   Prüfen, ob die Container auf den erwarteten Host-Ports antworten:

   ```bash
   curl -sS -o /dev/null -w 'front  %{http_code}\n' http://127.0.0.1:8080/
   curl -sS -o /dev/null -w 'api    %{http_code}\n' -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/api/v1/
   curl -sS -o /dev/null -w 'events %{http_code}\n' -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/events
   ```

   * Antworten die Container hier **200/401/403**, kommt der 502 meist vom Host-Nginx (z. B. falscher Upstream).
   * Antworten sie **nicht**, Logs prüfen:

     ```bash
     cd ~/taiga-docker
     docker compose logs --tail=100 taiga-back
     docker compose logs --tail=100 taiga-db
     ```

Mit diesen vier Schritten lässt sich ein Großteil der 502-Fälle nach Updates, NFS-Hängern oder Docker-Neustarts beheben.
