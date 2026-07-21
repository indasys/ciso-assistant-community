# Deployment auf der indasys Dokploy-Instanz

Diese Anleitung beschreibt das Deployment von CISO Assistant auf der zentralen
Dokploy-Instanz (`SVW-HQ1-DEPLOY01`) unter einer `*.indasys.de`-Domain.

Grundlage ist [`docker-compose.dokploy.yml`](../docker-compose.dokploy.yml) im
Repo-Root. Diese Datei ist die produktionstaugliche Variante der mitgelieferten
`docker-compose.yml` (die ist für lokale Tests via `docker-compose.sh` gedacht und
nutzt SQLite + eigenen Caddy-Proxy).

## Warum eine eigene Compose-Datei

Die Standard-`docker-compose.yml` funktioniert lokal, aber nicht direkt in Dokploy:

- **SQLite auf Bind-Mount:** Das Backend läuft als User `1001` mit `read_only`-Rootfs.
  Das Setup-Skript `docker-compose.sh` chownt vorher `./db` auf `1001:1001`. Dokploy
  führt dieses Skript nicht aus, legt den Ordner als `root` an, und das Backend hängt
  dann in der Warteschleife `database not ready; waiting` in `backend/startup.sh`
  (es kann die SQLite-Datei nicht schreiben). **Das ist die Ursache für ein hängendes
  Deployment.**
- **Eigener Caddy:** Der Caddy-Container terminiert TLS selbst auf Port 8443. Neben
  Dokploys Traefik ist das doppelt gemoppelt.

Die Dokploy-Variante löst das durch: PostgreSQL statt SQLite, kein Caddy (Traefik
übernimmt), persistentes Volume für Attachments und einen Init-Container, der die
Volume-Ownership fixt.

## Architektur

| Service            | Zweck                                  | Extern erreichbar          |
|--------------------|----------------------------------------|----------------------------|
| `frontend`         | SvelteKit UI (Port 3000)               | ja, über Traefik (`/`)     |
| `backend`          | Django API + Gunicorn (Port 8000)      | ja, über Traefik (`/api`)  |
| `huey`             | Async Worker (Mail, Background Jobs)   | nein                       |
| `postgres`         | Datenbank                              | nein                       |
| `qdrant`           | Vektor-DB für die AI-Features          | nein                       |
| `init-attachments` | Einmal-Job, chownt das Attachment-Volume | nein (beendet sich)      |

Persistente Daten liegen auf drei Named Volumes: `postgres_data` (Datenbank),
`attachments_data` (hochgeladene Evidences) und `qdrant_data` (Vektor-Index).

## Schritt 1: Projekt und Compose-Service in Dokploy anlegen

1. **Create Project** → Name z.B. `ciso-assistant`.
2. Service hinzufügen: **Compose**.
3. Source: **GitHub**, Repo `indasys/ciso-assistant-community`, Branch `main`.
4. **Compose Path** auf `docker-compose.dokploy.yml` setzen (nicht die Default-Datei!).

## Schritt 2: Environment Variables

Im Tab **Environment** eintragen. Alle folgenden Variablen sind **Pflicht**:

```env
# Domain (ohne Protokoll, nur der Host)
CISO_HOSTNAME=ciso-assistant.indasys.de

# Django Secret Key - einmalig erzeugen und fest hinterlegen, danach nie ändern
# (Neuer Key invalidiert alle bestehenden Sessions und Tokens)
DJANGO_SECRET_KEY=<mit `openssl rand -hex 32` erzeugen>

# PostgreSQL
POSTGRES_NAME=ciso_assistant
POSTGRES_USER=ciso
POSTGRES_PASSWORD=<starkes Passwort>
```

> Wichtig: `DJANGO_SECRET_KEY` **muss** gesetzt sein. Fehlt er, versucht das Backend
> ihn nach `db/django_secret_key` zu schreiben, was auf dem read-only Rootfs
> fehlschlägt.

Optional (SMTP für Einladungs- und Benachrichtigungs-Mails, sonst leer lassen):

```env
DEFAULT_FROM_EMAIL=ciso@indasys.de
EMAIL_HOST=<smtp-host>
EMAIL_PORT=587
EMAIL_HOST_USER=<user>
EMAIL_HOST_PASSWORD=<passwort>
EMAIL_USE_TLS=True
```

Diese SMTP-Variablen dann zusätzlich in der Compose-Datei bei `backend` **und**
`huey` durchreichen, falls benötigt.

## Schritt 3: Domain und Traefik

Da es kein Caddy mehr gibt, muss Traefik zwei Routen auf denselben Host legen. In
Dokploy im Tab **Domains** des Compose-Services **zwei Einträge** anlegen:

| Host               | Path   | Service    | Container Port | HTTPS |
|--------------------|--------|------------|----------------|-------|
| `ciso-assistant.indasys.de` | `/api` | `backend`  | 8000           | ja    |
| `ciso-assistant.indasys.de` | `/`    | `frontend` | 3000           | ja    |

Traefik priorisiert automatisch den längeren Pfad, `/api` gewinnt also gegen `/`.
Damit gehen API-Calls des Browsers an `backend`, alles andere ans `frontend`.

Traefik-Hinweise (siehe auch Vault-Ressource Dokploy):

- HTTPS ist Pflicht, Let's Encrypt über Dokploy aktivieren.
- Timeouts großzügig setzen (`readTimeout`/`idleTimeout` ~300s), da Library-Import
  und AI-Verarbeitung länger dauern können.
- Keine POST-Body-Limits, es werden Evidence-Dateien hochgeladen.
- Kein `stripPrefix` auf `/api`, der Pfad muss unverändert ans Backend.

## Schritt 4: Deploy

Deploy auslösen und Logs prüfen. Erwartete Startreihenfolge:

1. `postgres` wird `healthy`.
2. `init-attachments` läuft durch und beendet sich (`exited (0)`).
3. `backend` startet, migriert die DB, lädt die Libraries (dauert ein paar Minuten,
   `start_period` ist 150s), wird dann `healthy`.
4. `huey` und `frontend` starten, sobald das Backend gesund ist.

Wenn im Backend-Log dauerhaft `database not ready; waiting` steht, erreicht das
Backend Postgres nicht → `DB_HOST`, `POSTGRES_*` und den Postgres-Healthcheck prüfen.

## Schritt 5: Ersten Admin anlegen

Da `createsuperuser` interaktiv nicht praktikabel ist, den ersten Admin einmalig per
Environment-Variablen erzeugen. Temporär in Dokploy setzen:

```env
DJANGO_SUPERUSER_EMAIL=marco.renz@indasys.de
DJANGO_SUPERUSER_PASSWORD=<einmal-passwort>
```

`backend/startup.sh` legt den Superuser beim nächsten Start automatisch an (via
`createsuperuser --noinput`). Danach beide Variablen **wieder entfernen** und neu
deployen. Alternativ eine Shell im Backend-Container öffnen und
`python manage.py createsuperuser` manuell ausführen.

## Updates

Neue Version über den Deploy-Button in Dokploy ziehen. Die Images haben
`pull_policy: always`, holen also das aktuelle `:latest`. Migrationen laufen beim
Start automatisch. Die Volumes bleiben erhalten.

## Backup

Regelmäßig sichern:

- `postgres_data` (Datenbank) – z.B. per `pg_dump` aus dem Postgres-Container.
- `attachments_data` (Evidence-Dateien) – Volume-Backup.

`qdrant_data` ist ein reiner Index und wird bei Bedarf neu aufgebaut, ist also nicht
backup-kritisch.
