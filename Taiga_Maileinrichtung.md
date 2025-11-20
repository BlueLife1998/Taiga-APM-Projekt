
## 1. Was die IT liefern muss (Voraussetzung)

Damit Taiga Mails schicken kann, wird **genau eine** der folgenden Optionen benötigt:

1. **SMTP-Konto mit Benutzer/Passwort**, das sich an `smtp-out.hs-el.de` anmelden darf (Port + Verschlüsselung siehe unten),
   **oder**
2. **SMTP-Relay**, das von *diesem Server* (also der VM, auf der Taiga läuft) Mails ohne Auth annimmt.

Ohne eine dieser beiden Varianten bleibt es immer bei `535 authentication failed`, so wie es im aktuellen Log zu sehen ist.

An die IT kann z. B. folgender Text geschickt werden:

> „Wir betreiben eine Anwendung (Taiga) im Docker auf dem Host `apm...` im Hochschulnetz. Die Anwendung kann per SMTP über `smtp-out.hs-el.de:465` eine TLS-Verbindung aufbauen. Aktuell scheitert es an:
>
> ```text
> SMTPAuthenticationError: (535, b'5.7.8 Error: authentication failed: authentication failure')
> ```
>
> Wir benötigen entweder
> a) ein SMTP-Konto mit Benutzer/Passwort, das Plain-SMTP-Auth über Port 465 (SSL) oder 587 (STARTTLS) erlaubt
> **oder**
> b) ein SMTP-Relay, das von der IP des Servers Mails ohne Auth annimmt.“

Das ist die eigentliche Hürde.

---

## 2. Anleitung „Taiga mailfähig machen“ (für später)

Diese Schritte können später 1:1 durchgegangen werden, sobald die IT funktionierende SMTP-Daten bereitgestellt hat.

```bash
# 1. Auf den Server gehen
ssh <user>@<server>
cd ~/taiga-docker

# 2. Mail-Compose-Datei anlegen/anpassen
#    - EMAIL_HOST_USER / EMAIL_HOST_PASSWORD durch die echten Daten ersetzen
#    - Wenn IT Port 587 + STARTTLS will, siehe Alternative weiter unten
cat > docker-compose.email.yml <<'EOF'
services:
  taiga-back:
    environment:
      - EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
      - EMAIL_HOST=smtp-out.hs-el.de
      - EMAIL_PORT=465
      - EMAIL_USE_SSL=True
      - EMAIL_USE_TLS=False
      - EMAIL_HOST_USER=SERVICE.KONTO@hs-el.de
      - EMAIL_HOST_PASSWORD=SEHR_GEHEIM
      - DEFAULT_FROM_EMAIL=taiga@hs-el.de
      - SERVER_EMAIL=taiga@hs-el.de
EOF

# 3. Backend mit der Datei neu starten
docker compose \
  -f docker-compose.yml \
  -f docker-compose.override.yml \
  -f docker-compose.email.yml \
  up -d taiga-back

# 4. Test-Mail schicken
docker compose exec -T taiga-back sh -c 'cd /taiga-back && python manage.py sendtestemail ADMIN@hs-el.de'

# 5. Logs prüfen
docker compose logs --tail=200 taiga-back
```

**Wenn die IT STARTTLS (Port 587) vorschreibt**, dann sieht der Block in Schritt 2 so aus:

```bash
cat > docker-compose.email.yml <<'EOF'
services:
  taiga-back:
    environment:
      - EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
      - EMAIL_HOST=smtp-out.hs-el.de
      - EMAIL_PORT=587
      - EMAIL_USE_SSL=False
      - EMAIL_USE_TLS=True
      - EMAIL_HOST_USER=SERVICE.KONTO@hs-el.de
      - EMAIL_HOST_PASSWORD=SEHR_GEHEIM
      - DEFAULT_FROM_EMAIL=taiga@hs-el.de
      - SERVER_EMAIL=taiga@hs-el.de
EOF
```

Wenn *diese* Testmail durchgeht, funktionieren anschließend auch die Einladungen über die Taiga-Oberfläche.

---

## 3. Ursprungszustand wiederherstellen

Es soll vermieden werden, dass später versehentlich weiterhin mit aktuell verwendeten SMTP-Daten gearbeitet wird. Daher folgende Schritte:

1. **Mail-Compose-Datei löschen**

   ```bash
   cd ~/taiga-docker
   rm -f docker-compose.email.yml
   ```

2. **Backend ohne Mail-Override neu starten**
   (also wieder nur mit den Standarddateien)

   ```bash
   cd ~/taiga-docker
   docker compose \
     -f docker-compose.yml \
     -f docker-compose.override.yml \
     up -d taiga-back
   ```

   Damit läuft der Container wieder nur mit den Variablen aus den ursprünglichen Dateien → kein SMTP-User, kein Passwort.

3. **(Empfohlen) Shell-History bereinigen oder Passwort rotieren**
   Das Passwort (`toryA4NUk`) wurde in die Shell getippt. Dadurch steht es ziemlich sicher in `~/.bash_history`. Zwei Möglichkeiten:

   * Passwort in M365 / beim Account **ändern** → das ist die sauberste Variante.
   * oder History bearbeiten / löschen:

     ```bash
     history -w
     # oder komplett:
     cat /dev/null > ~/.bash_history
     ```

   Besser ist in jedem Fall: **Passwort ändern**, dann ist eine mögliche Leckage unkritisch.

4. **(Optional) prüfen, dass keine Mail-Settings mehr drin sind**

   Nach dem Neustart kann geprüft werden, welche Env-Variablen Taiga sieht:

   ```bash
   docker compose exec -T taiga-back env | grep EMAIL_ || true
   ```

   Wenn nichts oder nur Defaults ausgegeben werden, ist der Zustand wieder „wie vorher“.

---

## 4. Was sich aus dem Log sicher sagen lässt

* ✅ Netzwerk nach draußen: **OK**
* ✅ SSL-Handshake mit `smtp-out.hs-el.de:465`: **OK**
* ✅ Taiga/Django-Mailbackend: **OK** (die richtigen Einstellungen werden übernommen)
* ❌ SMTP-AUTH: **NICHT OK** – der Server lehnt *dieses* Konto ab

Das bedeutet: **Das Setup ist richtig, aber die Gegenstelle akzeptiert die Anmeldung nicht.** Genau deshalb ist oben Punkt 1 (etwas, das die IT freischaltet) notwendig.

---

## 5. Extra-Hinweis für die Person nachher

Wenn die IT irgendwann einen „internen Relay, keine Auth, nur aus dem Hochschulnetz“ bereitstellt (kommt oft vor), wird die Konfiguration noch einfacher:

```bash
cat > docker-compose.email.yml <<'EOF'
services:
  taiga-back:
    environment:
      - EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
      - EMAIL_HOST=MAILRELAY.INTERN.HS-EL.DE
      - EMAIL_PORT=25
      - EMAIL_USE_SSL=False
      - EMAIL_USE_TLS=False
      - DEFAULT_FROM_EMAIL=taiga@hs-el.de
      - SERVER_EMAIL=taiga@hs-el.de
EOF
```

dann wieder:

```bash
docker compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.email.yml up -d taiga-back
```

und Test-Mail.
