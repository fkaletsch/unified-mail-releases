# Unified Mail — public distribution

This repository only ships **release artifacts**. Application source stays private.

| Artifact | Where |
|----------|--------|
| Android APK | [Releases](https://github.com/fkaletsch/unified-mail-releases/releases/latest) (`unifiedmail.apk`) |
| Server images | GHCR: `ghcr.io/fkaletsch/unifiedmail-backend`, `ghcr.io/fkaletsch/unifiedmail-web` |

## Self-host (Docker)

```bash
cp .env.example .env
# Before first start: set POSTGRES_PASSWORD + matching UM_DATABASE_URL, and UM_MASTER_KEY
mkdir -p um_blobs && chmod a+rwX um_blobs
docker compose pull
docker compose up -d
```

See [`.env.example`](.env.example) and `docker-compose.yml`.

**NAS / Synology:** Schritt-für-Schritt inkl. Container Manager und Reverse-Proxy → **[install-nas.md](install-nas.md)**.

## Android

Open the [latest release](https://github.com/fkaletsch/unified-mail-releases/releases/latest), download `unifiedmail.apk`, install (allow unknown sources if needed), then point the app at your server URL.

## License

See [LICENSE](LICENSE). Binaries and images are provided for self-hosting under that license. Source code is not published here.
