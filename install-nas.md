# Unified Mail auf einer NAS (z. B. Synology)

Anleitung für Self-Host mit den **öffentlichen** Images aus diesem Repo.  
Es wird **nicht** aus dem Quellcode gebaut — nur `docker compose pull`.

Voraussetzungen:

- Docker / **Container Manager** (Synology DSM 7+)
- Genug Speicher für Mail-Cache (Richtwert: ≥ 50 GB frei, siehe `UM_CACHE_QUOTA_GB`)
- Optional, aber empfohlen: Reverse-Proxy mit TLS (Synology Application Portal, Nginx Proxy Manager, Caddy, Traefik, …)
- Outbound-HTTPS zu den Mail-Providern

Unified Mail bringt **keinen** Reverse-Proxy mit. Öffentlich sollte nur der Proxy auf Port **8088** (Web-UI) zeigen.

---

## 1. Dateien auf die NAS legen

Auf dem PC dieses Repo klonen oder als ZIP laden und nach z. B. `/volume1/docker/unified-mail` kopieren.  
Im Projektordner sollten mindestens liegen:

```text
docker-compose.yml
.env.example
```

SSH-Beispiel (Synology, Pfad anpassen):

```bash
# auf der NAS
mkdir -p /volume1/docker/unified-mail
cd /volume1/docker/unified-mail
# Dateien hierher legen (SCP, File Station, git clone dieses Repos, …)
```

---

## 2. Konfiguration

```bash
cd /volume1/docker/unified-mail
cp .env.example .env
mkdir -p um_blobs
chmod a+rwX um_blobs
```

In `.env` **vor dem ersten Start** mindestens setzen:

| Variable | Beispiel / Hinweis |
|----------|--------------------|
| `POSTGRES_PASSWORD` | starkes Passwort — **nicht** den Platzhalter lassen |
| `UM_DATABASE_URL` | muss dasselbe Passwort (und User/DB) enthalten, z. B. `postgres://um:GEHEIM@db:5432/unifiedmail?sslmode=disable` |
| `UM_MASTER_KEY` | `openssl rand -hex 16` (auf PC oder NAS) — **sichern**, Verlust = Provider neu einbinden |
| `UM_PUBLIC_URL` | öffentliche URL, z. B. `https://mail.example.com` (ohne Slash am Ende) |
| `UM_COOKIE_SECURE` | hinter HTTPS: `true` |
| `UM_WEBAUTHN_RPID` | Hostname, z. B. `mail.example.com` |
| `UM_WEBAUTHN_ORIGIN` | volle Origin, z. B. `https://mail.example.com` |

**Postgres-Passwort:** Compose liest `POSTGRES_*` aus `.env`. Das gilt nur beim **ersten** Anlegen des Volumes `um_pg`. Später nur in `.env` zu ändern reicht nicht — dann müsstest du das Volume löschen (Datenverlust) oder das Passwort in Postgres manuell anpassen. Deshalb: vor `docker compose up` setzen und `UM_DATABASE_URL` synchron halten.

Lokal nur zum Testen: `UM_PUBLIC_URL=http://NAS-IP:8088` und `UM_COOKIE_SECURE=false` reichen oft.

`.env` und `um_blobs/` **nicht** öffentlich teilen; `.env` gehört ins Backup (wegen `UM_MASTER_KEY` und DB-Passwort).

---

## 3. Starten

### Variante A — SSH (empfohlen)

```bash
cd /volume1/docker/unified-mail
docker compose pull
docker compose up -d
docker compose ps
```

UI: `http://<NAS-IP>:8088` bzw. deine Proxy-URL.

### Variante B — Synology Container Manager

1. **Container Manager** → **Projekt** → **Erstellen**
2. Projektpfad = Ordner mit `docker-compose.yml` (z. B. `/docker/unified-mail`)
3. Quelle: vorhandene `docker-compose.yml` verwenden
4. Vor dem Start: `.env` und `um_blobs` wie in §2 anlegen (per SSH oder File Station + Texteditor)
5. Projekt erstellen / starten — Images werden von GHCR gezogen (`ghcr.io/fkaletsch/unifiedmail-…`)

Falls der Pull von `ghcr.io` scheitert: Packages müssen **öffentlich** sein, oder Registry-Login in Container Manager hinterlegen.

---

## 4. Reverse-Proxy (Synology)

Ziel: HTTPS von außen → HTTP `localhost:8088` (Service `web`).

1. **Systemsteuerung** → **Anmeldeportal** / **Application Portal** → **Umgekehrter Proxy** (DSM-Versionen heißen das leicht anders)
2. Neue Regel, z. B.:
   - Quelle: `HTTPS`, Hostname `mail.example.com`, Port `443`
   - Ziel: `HTTP`, `localhost`, Port `8088`
3. Zertifikat (Let's Encrypt) dem Hostnamen zuweisen
4. Wichtig: Proxy soll `X-Forwarded-Proto: https` und den korrekten `Host` setzen (bei Synology meist Standard bei Reverse-Proxy)

Danach in `.env` die öffentlichen Werte aus §2 eintragen und neu starten:

```bash
docker compose up -d
```

OAuth-Redirects (Google/Microsoft) müssen exakt zur `UM_PUBLIC_URL` passen, z. B.  
`https://mail.example.com/api/v1/mail-accounts/oauth/<provider>/callback`.

---

## 5. Erster Start

1. Browser → deine URL → Registrierung (**erster User = Admin**, nur bei leerer Datenbank)
2. Weitere Nutzer per Invite (Admin/Settings)
3. Postfächer im Wizard verbinden (oft **App-Passwort** statt Web-Login)

Android-App: [Releases](https://github.com/fkaletsch/unified-mail-releases/releases/latest) → `unifiedmail.apk`, Server-URL = deine `UM_PUBLIC_URL`.

---

## 6. Updates (manuell)

Kein automatisches Watchtower als Default — bessere Kontrolle auf der NAS.

```bash
cd /volume1/docker/unified-mail
docker compose pull
docker compose up -d
```

Vor größeren Updates: Postgres-Volume und vor allem **`.env` / `UM_MASTER_KEY`** sichern.

Container Manager: Projekt → **Aktualisieren** / Images neu pullen, sofern die UI das anbietet — SSH ist oft zuverlässiger.

---

## 7. Kurz-Troubleshooting

| Problem | Check |
|---------|--------|
| Seite lädt nicht | `docker compose ps`, Port 8088 frei? Firewall? |
| Login hinter HTTPS weg | `UM_COOKIE_SECURE=true`, Proxy `X-Forwarded-Proto` |
| Keine Anhänge / Sync-Fehler | `um_blobs` existiert und ist beschreibbar (`chmod a+rwX`)? |
| Image-Pull von GHCR fehlgeschlagen | Package öffentlich? Netz/DNS der NAS? |
| OAuth falsch | `UM_PUBLIC_URL` = exakte öffentliche Origin |

Services im Stack: `db` (Postgres), `redis`, `api`, `worker`, `web`.  
`api`/`worker` teilen das Backend-Image; `web` liefert UI und proxyt `/api` intern.
