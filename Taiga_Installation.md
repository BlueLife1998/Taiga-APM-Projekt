# Taiga (Docker) auf Hochschulserver mit Rootless-Docker, lokalen Docker-Volumes und Host-Reverse-Proxy

Diese Anleitung beschreibt die vollständige Einrichtung einer stabilen Taiga-Instanz auf einem Hochschulserver:

* Docker läuft **rootless** im Benutzerkontext.
* Taiga wird über das offizielle `taiga-docker`-Repo betrieben (aktuelles Setup mit `.env`).
* Daten liegen in **Docker-Volumes** auf dem Server (NFS optional).
* Erreichbarkeit über **Host-Nginx** unter `http://apm.hs-emden-leer.de`.
* Enthält Troubleshooting zu:

  * „DB unhealthy“,
  * fehlenden Migrationen,
  * Login-Problemen („Oops, username/password incorrect“),
  * Gateway/Port 9000.

---

## 0) Zielbild & Eckdaten

**Zielbild**

* Taiga läuft als Docker-Stack (`taiga-back`, `taiga-front`, `taiga-events`, `taiga-protected`, `taiga-async`, RabbitMQs, `taiga-gateway`).
* Docker läuft **rootless** im Userkontext (z. B. `lohdens`).
* Daten (DB, Media, Static) liegen in **Docker-Volumes** (Standard aus `taiga-docker`).
* Erreichbarkeit von außen:
  → Host-Nginx auf `apm.hs-emden-leer.de` → `http://127.0.0.1:9000` (Gateway).

**Beispielumgebung**

* OS: Ubuntu 24.04 (VM)
* User: `lohdens`
* Hostname/DNS (im Hochschul-VPN): `apm.hs-emden-leer.de`
* Taiga-Installationsverzeichnis: `~/taiga-docker`

**Warum so?**

* **Rootless Docker** minimiert Privilegien (Multi-User-Host, Studierende etc.).
* **Offizielles `taiga-docker`** vermeidet Frickelei mit eigenen Compose-Dateien; Updates folgen der Doku. ([docs.taiga.io][1])
* **Docker-Volumes** sind unaufwändig und robust; NFS lässt sich separat für Backups/Snapshots nutzen.
* **Host-Nginx** ermöglicht volle Kontrolle über Domain, TLS, Logging etc.

---

## 1) Rootless-Docker installieren & aktivieren

> Alle Schritte als normaler Benutzer (z. B. `lohdens`), nicht als root.

### 1.1 Docker-Pakete installieren (falls noch nicht da)

```bash
# evtl. als root
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli docker-ce-rootless-extras
```

### 1.2 Rootless Docker initial einrichten

```bash
# Als User
dockerd-rootless-setuptool.sh install
```

### 1.3 Linger aktivieren (damit User-Docker auch ohne Login läuft)

```bash
sudo loginctl enable-linger $USER
```

### 1.4 User-Docker starten und aktivieren

```bash
systemctl --user enable --now docker
systemctl --user status docker   # mit 'q' wieder raus
```

### 1.5 Umgebungsvariablen setzen

```bash
echo 'export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

Optional testen:

```bash
docker info | head
```

Wenn der Befehl ohne Fehlermeldung läuft, ist rootless Docker einsatzbereit.

---

## 2) (Optional) NFS-Basis vorbereiten

Wenn wie in einer ersten Variante ein NFS-Share für Backups/Logs/etc. genutzt werden soll, kann weiterhin z. B. `/mnt/nas` eingerichtet werden.
**Wichtig:** In diesem Setup werden Taiga-Container standardmäßig **nicht** direkt auf NFS gemountet, sondern nutzen ihre Docker-Volumes. NFS ist hier optional für Snapshots/Backups.

Minimal:

```bash
# Mountpunkt "durchlaufbar" machen
sudo mkdir -p /mnt/nas
sudo chmod 0711 /mnt/nas
```

Der Rest (Shares etc.) kann nach Bedarf eingerichtet werden, für Taiga selbst ist dies in dieser Anleitung jedoch nicht erforderlich.

---

## 3) Taiga-Docker-Projekt einrichten (offizielles Setup)

Es wird der aktuellen „Install Taiga in Production / Docker“-Doku gefolgt, angepasst für die hier verwendete Domain. ([docs.taiga.io][1])

### 3.1 Repository klonen und Branch `stable` auschecken

```bash
cd ~
git clone https://github.com/taigaio/taiga-docker.git
cd taiga-docker
git checkout stable
```

### 3.2 `.env` anlegen/prüfen und konfigurieren

Falls noch keine `.env` vorhanden ist:

```bash
# Nur falls nötig:
cp .env.sample .env  2>/dev/null || true
```

Dann:

```bash
nano .env
```

Wichtige Einstellungen:

**DB:**

```env
POSTGRES_USER=taiga
POSTGRES_PASSWORD=taiga
```

(Produktiv sollten hier *eigene* Credentials gewählt werden, für ein erstes Setup ist obige Einstellung aber ausreichend.)

**URLs / Domain:**

```env
TAIGA_SCHEME=http
TAIGA_DOMAIN=apm.hs-emden-leer.de
SUBPATH=""
WEBSOCKETS_SCHEME=ws
```

**Secret-Key (unbedingt anpassen):**

```env
SECRET_KEY="hier-einen-langen-zufaelligen-string-eintragen"
```

z. B. generieren:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(50))"
```

**Mail (Start-Setup):**

```env
EMAIL_BACKEND=console
EMAIL_DEFAULT_FROM=changeme@example.com
EMAIL_USE_TLS=True
EMAIL_USE_SSL=False
```

**Telemetry nach Geschmack:**

```env
ENABLE_TELEMETRY=True   # oder False
```

Speichern & Editor schließen.

---

## 4) Taiga-Stack starten & initial einrichten

### 4.1 Erster Start via `launch-taiga.sh`

```bash
cd ~/taiga-docker
./launch-taiga.sh
```

* Das Skript ruft `docker compose -f docker-compose.yml up -d` auf.
* Beim allerersten Start kann `taiga-db` kurz „unhealthy“ sein, während Postgres initialisiert.

Status prüfen:

```bash
docker compose ps
```

Wichtig ist:

* `taiga-db` → `Up (healthy)` (kann ein paar Sekunden dauern).
* Andere Services dürfen anfangs auch mal `Created` sein – das wird im nächsten Schritt bereinigt.

---

### 4.2 Migrations einmalig ausführen

Ohne Migrations existieren keine Tabellen → `relation "users_user" does not exist` bei `createsuperuser`.

Deshalb:

```bash
cd ~/taiga-docker
./taiga-manage.sh migrate
```

Warten, bis der Vorgang abgeschlossen ist. Es sollte eine lange Liste „Applying … OK“ erscheinen und am Ende **kein** Fehler.

(Optional, aber sauber):

```bash
./taiga-manage.sh loaddata initial_user
./taiga-manage.sh loaddata initial_project_templates
./taiga-manage.sh loaddata initial_domains
```

---

### 4.3 Superuser anlegen

Erst jetzt:

```bash
./taiga-manage.sh createsuperuser
```

* Username z. B. `lukasohdens`
* E-Mail
* Passwort (merken!)

Falls später das Passwort neu gesetzt werden muss:

```bash
./taiga-manage.sh changepassword lukasohdens
```

---

### 4.4 Stack vollständig hochfahren (inkl. Gateway)

Damit wirklich **alle** Services laufen (insbesondere `taiga-gateway` auf Port 9000):

```bash
cd ~/taiga-docker
docker compose up -d
```

Dann:

```bash
docker compose ps
```

Erwartung:

* `taiga-db` → Up (healthy)
* `taiga-back` → Up
* `taiga-front` → Up
* `taiga-events` → Up
* `taiga-protected` → Up
* `taiga-async` → Up
* `taiga-async-rabbitmq` → Up
* `taiga-events-rabbitmq` → Up
* `taiga-gateway` → Up

Port-Mapping prüfen:

```bash
docker compose port taiga-gateway 80
# typischerweise: 0.0.0.0:9000
```

**Lokaler Test:**

```bash
curl -I http://127.0.0.1:9000/
```

Wenn ein HTTP-Status wie 200/302/etc. zurückkommt, läuft Taiga intern.

---

## 5) Reverse-Proxy auf dem Host (Nginx, HTTP)

### 5.1 Nginx installieren

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

Default-Site abschalten:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

### 5.2 Nginx-Site für `apm.hs-emden-leer.de`

Config anlegen:

```bash
sudo tee /etc/nginx/sites-available/taiga.conf >/dev/null <<'NGINX'
server {
    listen 80;
    server_name apm.hs-emden-leer.de;

    client_max_body_size 100M;

    # Frontend + API + Media werden über das Taiga-Gateway auf :9000 bedient
    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
        proxy_pass http://127.0.0.1:9000/;
    }

    # Events (WebSockets)
    location /events {
        proxy_pass http://127.0.0.1:9000/events;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
NGINX
```

Site aktivieren:

```bash
sudo ln -sf /etc/nginx/sites-available/taiga.conf /etc/nginx/sites-enabled/taiga.conf
sudo nginx -t
sudo systemctl reload nginx
```

### 5.3 Hostseitiger Test

```bash
curl -I -H 'Host: apm.hs-emden-leer.de' http://127.0.0.1/
```

Wenn ein sauberer HTTP-Status (kein 502/504) geliefert wird, kann im Browser:

```text
http://apm.hs-emden-leer.de
```

geöffnet, mit dem angelegten Superuser-Account eingeloggt und mit der Arbeit begonnen werden.

---

## 6) Backups (minimal)

### 6.1 DB-Backup

```bash
cd ~/taiga-docker
docker compose exec -T taiga-db \
  pg_dump -U taiga taiga | gzip > ~/taiga-backup-$(date +%F_%H%M).sql.gz
```

Der Pfad kann nach Bedarf z. B. auf NFS (`/mnt/nas/...`) gelegt werden.

### 6.2 Media/Static

Taiga speichert Dateien in einem Docker-Volume, das im Compose definiert ist.
Für ein sauberes Backup kann z. B. mit `docker run --rm -v <volume>:/data ...` gearbeitet oder es können regelmäßige Snapshots des gesamten Hosts/der VM genutzt werden.

---

## 7) HTTPS

Zunächst muss über das Harica Portal ein SSL-Zertifikat beantragt werden (Hinweis: Dies kann nur von wissenschaftlichen Mitarbeitern oder Dozenten beantragt werden!).
Hierzu die Anleitung des HRZ beachten:
[https://hrz-support.hs-emden-leer.de/de/hc/1347112677/44/serverzertifikate?category_id=30](https://hrz-support.hs-emden-leer.de/de/hc/1347112677/44/serverzertifikate?category_id=30)

Anschließend den HRZ Support informieren, dass ein Zertifikat beantragt wurde.

1. Certbot installieren:

   ```bash
   sudo apt-get install -y certbot python3-certbot-nginx
   sudo certbot --nginx -d apm.hs-emden-leer.de
   ```

2. In `.env` anschließend umstellen:

   ```env
   TAIGA_SCHEME=https
   WEBSOCKETS_SCHEME=wss
   # TAIGA_DOMAIN bleibt: apm.hs-emden-leer.de
   ```

3. Taiga-Stack einmal neu starten:

   ```bash
   cd ~/taiga-docker
   docker compose up -d
   ```

---

## 8) Kurz-Checkliste (Neuinstallation auf frischem Server)

```bash
# 0) Rootless Docker
sudo apt-get install -y docker-ce docker-ce-cli docker-ce-rootless-extras
dockerd-rootless-setuptool.sh install
sudo loginctl enable-linger $USER
systemctl --user enable --now docker
echo 'export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock' >> ~/.bashrc
source ~/.bashrc

# 1) Taiga-Docker
cd ~
git clone https://github.com/taigaio/taiga-docker.git
cd taiga-docker
git checkout stable
cp .env.sample .env   # falls nötig
nano .env             # Domain, Secret, Mail, etc. setzen

# 2) Stack starten & initialisieren
./launch-taiga.sh
./taiga-manage.sh migrate
./taiga-manage.sh createsuperuser
docker compose up -d   # Stack vollständig inkl. Gateway hochfahren

# 3) Host-Nginx
sudo apt-get install -y nginx
sudo rm -f /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-available/taiga.conf   # Config wie oben
sudo ln -sf /etc/nginx/sites-available/taiga.conf /etc/nginx/sites-enabled/taiga.conf
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9) Troubleshooting (aktuelle Architektur)

### A) `taiga-db` anfangs „unhealthy“, später aber ok

Wenn beim ersten `./launch-taiga.sh`:

* `taiga-db` kurz als `unhealthy` markiert wird und
* `taiga-back` / `taiga-gateway` nur `Created` sind,

dann:

```bash
cd ~/taiga-docker
docker compose ps   # warten, bis taiga-db "Up (healthy)" ist
docker compose up -d
```

Damit wird der ganze Stack sauber hochgezogen.

---

### B) `createsuperuser` bricht ab mit `relation "users_user" does not exist`

**Symptom:**

* Bei `./taiga-manage.sh createsuperuser` erscheint ein Hinweis auf „261 unapplied migrations“ und:

  ```text
  django.db.utils.ProgrammingError: relation "users_user" does not exist
  ```

**Lösung:**

```bash
cd ~/taiga-docker
./taiga-manage.sh migrate
./taiga-manage.sh createsuperuser
```

Reihenfolge: immer **erst** „migrate“, **dann** Superuser anlegen.

---

### C) Login: „Oops, … username/email or password are incorrect“, obwohl die Zugangsdaten sicher erscheinen

**Symptom:**

* Taiga lädt, die Login-Maske erscheint.
* Es werden vermeintlich richtige Daten eingegeben, aber es erscheint nur der „Oops… username/email or password are incorrect“-Screen.

**Mögliche Ursachen:**

1. Der User existiert schlicht nicht (z. B. Tippfehler bei `createsuperuser`).
2. Das Passwort ist anders, als angenommen.

**Schnellfix (Passwort neu setzen):**

```bash
cd ~/taiga-docker
./taiga-manage.sh changepassword <BENUTZERNAME>
```

Beispiel:

```bash
./taiga-manage.sh changepassword lukasohdens
```

Neues Passwort zweimal eingeben, danach erneut im Browser anmelden.

Um sicherzugehen, dass der User existiert:

```bash
./taiga-manage.sh shell
```

Dann in der Python-Shell:

```python
from django.contrib.auth import get_user_model
User = get_user_model()
print(User.objects.filter(username="lukasohdens").values("id", "username", "email", "is_active", "is_superuser"))
```

---

### D) `curl http://127.0.0.1:9000` → „Failed to connect“

**Symptom:**

```bash
curl -I http://127.0.0.1:9000/
# curl: (7) Failed to connect ...
```

**Ursache:** `taiga-gateway` läuft nicht oder hat kein Port-Mapping.

**Check & Fix:**

```bash
cd ~/taiga-docker
docker compose ps
```

Wenn `taiga-gateway` nicht `Up` ist:

```bash
docker compose up -d
```

Port-Mapping kontrollieren:

```bash
docker compose port taiga-gateway 80
# Erwartung: 0.0.0.0:9000 oder 127.0.0.1:9000
```

Dann erneut:

```bash
curl -I http://127.0.0.1:9000/
```

---

### E) Browser zeigt 502 Bad Gateway auf `apm.hs-emden-leer.de`

**Schnell-Checkliste:**

1. **Host-Nginx-Config prüfen:**

   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

2. **Taiga-Gateway lokal testen:**

   ```bash
   curl -I http://127.0.0.1:9000/
   ```

   * Wenn dieser Aufruf bereits scheitert → `docker compose ps`, `docker compose logs taiga-gateway`.
   * Wenn dieser Aufruf funktioniert, aber `apm.hs-emden-leer.de` 502 liefert → Fehler in der Nginx-Proxy-Config (z. B. falsche IP/Port).

3. **Rootless-Docker neu starten (falls der Daemon hängt):**

   ```bash
   systemctl --user restart docker
   cd ~/taiga-docker
   docker compose up -d
   ```
