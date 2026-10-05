# Levantar Sure desde cero

Guía para montar esta instancia en una máquina nueva (o restaurarla).

**Requisitos:** Docker + Docker Compose, `git`.

---

## 1. Obtener el código fuente (contexto de build)

```bash
git clone https://github.com/we-promise/sure.git /home/germoso/code/sure
git -C /home/germoso/code/sure checkout 14638a70181d9adadc75e7d47d5b2975ee18bf46
```

Ver [`UPSTREAM.md`](../UPSTREAM.md) para la procedencia y por qué se construye
desde fuente en vez de bajar la imagen de GHCR.

## 2. Construir la imagen

```bash
cd /home/germoso/code/sure
docker build --build-arg BUILD_COMMIT_SHA=$(git rev-parse HEAD) -t sure:local .
# ~5 min: instala 277 gems y precompila assets
```

## 3. Configurar secretos

```bash
cd deploy
cp .env.example .env
chmod 600 .env
# editar .env y rellenar con:
#   openssl rand -hex 64   -> SECRET_KEY_BASE
#   openssl rand -hex 32   -> ACTIVE_RECORD_ENCRYPTION_*  y MCP_API_TOKEN
#   openssl rand -hex 24   -> POSTGRES_PASSWORD
```

> ⚠️ **`.env` jamás se sube a git** (`.gitignore` lo cubre en cualquier
> profundidad). Ver hallazgo B1 de la auditoría: el upstream trae un
> `SECRET_KEY_BASE` público por defecto.

## 4. Levantar

```bash
cd deploy
docker compose up -d
# 4 servicios: web, worker (Sidekiq), db (Postgres 16), redis
```

Esperar el healthcheck (~40 s):

```bash
docker compose ps
curl -fsS http://192.168.18.41:3000/up && echo OK
```

## 5. Primer usuario

Abrir <http://192.168.18.41:3000/registration/new>.
**La primera cuenta queda como `super_admin`.**

Después se puede cerrar el registro (`ONBOARDING_STATE=closed` en `.env`), pero
ojo: **el endpoint `POST /api/v1/auth/signup` ignora esa configuración**
(hallazgo B7 de la auditoría).

## 6. Activar MCP (opcional)

```bash
# en deploy/.env
MCP_API_TOKEN=<openssl rand -hex 32>
MCP_USER_EMAIL=<email del super_admin>      # debe coincidir con un usuario existente

docker compose up -d --force-recreate web worker
```

Probar:

```bash
TOKEN=$(grep '^MCP_API_TOKEN=' .env | cut -d= -f2)
curl -X POST http://192.168.18.41:3000/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize"}'
# → HTTP 200, serverInfo {name:"sure"}
```

---

## Troubleshooting

### Los fetch de Yahoo Finance fallan (IPv6)

**Síntoma:** `find_or_fetch_rate` devuelve `nil`, intermitente (3/5, 4/5 OK).

**Causa:** el resolver devuelve AAAA primero y el contenedor **no tiene ruta
IPv6 de salida**; `gai.conf` vacío ⇒ glibc prefiere IPv6.

**Fix ya aplicado en `compose.yml`:**
- `sysctls: net.ipv6.conf.*.disable_ipv6=1`
- mount `./gai.conf:/etc/gai.conf:ro` con `precedence ::ffff:0:0/96 100`

**Verificación:** `Addrinfo.getaddrinfo` debe devolver IPv4 primero → 8/8 OK.

> El upstream resuelve esto forzando `dns: 8.8.8.8 / 1.1.1.1`, lo que manda
> toda tu resolución DNS a Google/Cloudflare. Se eligió `gai.conf` para
> conservar el DNS del host.

### "El token MCP da 401"

`MCP_USER_EMAIL` no coincide con ningún usuario, **o** no está en `x-app-env`
de `compose.yml` (compose no pasa variables que no estén declaradas ahí).

```bash
docker compose config | grep MCP_     # debe mostrar ambos valores
```

### Redis en loop con `setpriv: setresuid failed`

Falta `SETUID`/`SETGID`: el entrypoint necesita bajar de root a usuario `redis`.

```yaml
  redis:
    cap_drop: [ ALL ]
    cap_add: [ SETUID, SETGID ]   # imprescindible
```

### `/rails/tmp` no escribible → `Errno::EACCES`

Los tmpfs declarados con la sintaxis corta `tmpfs: - /rails/tmp` quedan en
**modo 775 (root)**. Usar la forma larga con `mode: 0o1777`.

### La base de datos "se reinició vacía" tras mover la carpeta

Compose usa **el nombre de la carpeta** como nombre de proyecto. Si cambia,
crea volúmenes nuevos. Por eso `compose.yml` fija:

```yaml
name: sure-deploy
```

**No quitarlo** sin antes verificar `docker volume ls`.

---

## Comandos diarios

```bash
cd deploy
docker compose ps
docker compose logs -f web
docker compose down              # parar (conserva datos)
docker compose up -d             # arrancar
docker compose restart web

# Backup de la base
docker compose exec -T db pg_dump -U sure sure_production \
  | gzip > "../backups/sure-$(date +%F).sql.gz"
```

---

## Medidas de seguridad aplicadas

| Control | Valor |
|---|---|
| `privileged` | `false` |
| `security_opt` | `no-new-privileges:true` |
| `cap_drop` | `ALL` (db y redis conservan lo mínimo) |
| `read_only` rootfs | `true` (tmpfs `0o1777` + volumen `/rails/storage`) |
| `mem_limit` | web 1g · worker 768m · db 512m · redis 256m |
| `pids_limit` | 300 / 200 / 100 / 50 |
| `user` | `1000:1000` (no-root) |
| Puerto | **solo** `192.168.18.41:3000` |
| `docker.sock` | no montado |
| Telemetría | `POSTHOG_FEEDBACK_ENABLED=false`, todo lo demás vacío |
