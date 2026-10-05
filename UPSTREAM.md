# Upstream — procedencia del código

Este proyecto **no contiene el código fuente de Sure**. Lo usa como imagen
pre-construida, con la procedencia fijada abajo.

| | |
|---|---|
| **Proyecto** | [`we-promise/sure`](https://github.com/we-promise/sure) |
| **Qué es** | Fork comunitario de Maybe Finance (archivado) — gestor de finanzas personales |
| **Licencia** | AGPL-3.0 |
| **Estado upstream** | ✅ activo — releases mensuales (`v0.7.5` 2026-09-30) |
| **Auditoría** | [`docs/auditoria-sure.md`](docs/auditoria-sure.md) → 🟢 sin código malicioso |

---

## Versión fijada

| Campo | Valor |
|---|---|
| **Commit** | `14638a70181d9adadc75e7d47d5b2975ee18bf46` |
| **Descripción** | `fix(transactions): stop Cancel in Convert to trade from converting (#3941)` |
| **Fecha** | 2026-10-02 |
| **Imagen** | `sure:local` (construida desde fuente, no bajada de GHCR) |
| **Digest** | `sha256:0735b52e4561ef8df60df1898e40296422663d7d42e5c6ce53a28e017ce73519` |
| **Tamaño** | 1.43 GB · linux/amd64 |
| **Build arg** | `BUILD_COMMIT_SHA=14638a70181d…` (embebido dentro de la imagen) |

> **Por qué construir desde fuente y no bajar `ghcr.io/we-promise/sure:stable`:**
> la imagen de GHCR **no lleva attestation SLSA ni firma cosign** (el workflow
> usa `provenance: false`). Construirla uno mismo elimina la necesidad de
> confiar en el registro. Ver auditoría, apartado "no verificado".

---

## Clon del código fuente (contexto de build)

El código vive **fuera de este repo** y **no se commitea** (181 MB, y es código
de un tercero):

```bash
git clone https://github.com/we-promise/sure.git /home/germoso/code/sure
git -C /home/germoso/code/sure checkout 14638a70181d9adadc75e7d47d5b2975ee18bf46
```

Se usa **solo** como contexto de `docker build`. Los contenedores NO dependen
de esa carpeta: corren desde la imagen ya construida.

---

## Reconstruir la imagen

```bash
cd /home/germoso/code/sure
docker build \
  --build-arg BUILD_COMMIT_SHA=$(git rev-parse HEAD) \
  -t sure:local .
```

## Actualizar a una versión nueva de upstream

```bash
cd /home/germoso/code/sure
git fetch origin
git log --oneline HEAD..origin/main          # ver qué cambia
git merge origin/main                         # o git reset --hard origin/main

# ⚠️ antes de reconstruir, revisar:
#   - docs/ (guías de hosting) por cambios en variables de entorno
#   - compose.example.yml por nuevos servicios
#   - .env.example por nuevas variables requeridas
#   - CHANGELOG / releases por avisos de migración

docker build --build-arg BUILD_COMMIT_SHA=$(git rev-parse HEAD) -t sure:local .
cd /home/germoso/code/finance-box/deploy
docker compose up -d
```

---

## Parches pendientes de la auditoría

Aplicarlos **en el clon** y reconstruir (o contribuirlos upstream):

| Severidad | Archivo | Fix |
|---|---|---|
| Media-alta | `app/models/provider/simplefin.rb:35,41` | allowlist de host + forzar `https` (SSRF) |
| Media | `app/controllers/import/mappings_controller.rb:37,41` | allowlist en `.constantize` |
| Media | `app/controllers/api/v1/auth_controller.rb:15,75` | guardas `ensure_signup_open` y `local_login_allowed_for?` |
| Media | `config/initializers/content_security_policy.rb` | activar CSP (hoy todo comentado) |
| Media | `config/initializers/cors.rb:13` | restringir `origins "*"` |

---

## Verificación de procedencia

```bash
# La imagen declara de qué commit salió
docker buildx imagetools inspect ghcr.io/we-promise/sure:stable \
  --format '{{json .manifest.annotations}}' | jq
#   org.opencontainers.image.revision = 789883081feda49...

# Ese commit existe en el clon local
git -C /home/germoso/code/sure cat-file -t 789883081feda49fcfa8c7bc923cc77c9fe301ad
#   commit

# Continuidad con el upstream archivado (Maybe)
git -C /home/germoso/code/sure merge-base --is-ancestor \
  77b5469832758d1cbee1a940f3012a1ae1c74cd3 HEAD && echo OK
```
