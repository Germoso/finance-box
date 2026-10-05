# Auditoría de seguridad — Sure (`we-promise/sure`)

| Campo | Valor |
|---|---|
| **Fecha** | 2026-10-04 |
| **Repo auditado** | `/home/germoso/code/sure` (clon de `github.com/we-promise/sure`) |
| **HEAD** | `14638a70181d9adadc75e7d47d5b2975ee18bf46` (2026-10-02) |
| **Alcance** | 6.974 archivos · 390.278 líneas Ruby · 44.668 ERB · 16.824 JS/TS · 27.923 Dart · 3.602 commits · 126 MiB de historia |
| **Método** | 6 agentes de auditoría en paralelo + barrido mecánico propio. **Solo lectura**: no se modificó, commiteó ni ejecutó código del repo. |
| **Veredicto** | 🟢 **SIN CÓDIGO MALICIOSO** — con **17 hallazgos de deuda/configuración** a atender antes de producción |

---

## 1. Resumen ejecutivo

**No se encontró ningún código malicioso (severidad A) en ninguna de las 6 áreas auditadas.**

Ni una sola vez apareció: reverse shell, bindeo de puertos, `curl | bash` hacia dominios desconocidos, sockets crudos (TCP/UDP/DNS), montaje de `docker.sock`, tareas `cron`/`at` implantadas, payloads Base64/ofuscados, ejecución dinámica con input de usuario, exfiltración a dominios no atribuibles, `postinstall` scripts, fuentes `git:` en manifiestos, WebAssembly, ni git hooks activos.

**Todo lo encontrado es deuda de seguridad y configuración insegura por defecto — no intención hostil.** La postura general del repo está *por encima* de la media: GitHub Actions pineadas por SHA, contenedor no-root, Helm endurecido, SQL saneado, rate limiting y saneador de telemetría.

| Severidad | Cantidad |
|---|---:|
| 🔴 (A) Malicioso | **0** |
| 🟠 (B) Requiere acción antes de producción | **17** |
| 🟡 (C) Normal pero notable | ~20 |
| ⚪ (D) Falso positivo descartado | ~40 |

---

## 2. Hallazgos que exigen acción

### 🔴 PRIORIDAD 1 — Obligatorio

#### B1 · `SECRET_KEY_BASE` con valor público hardcodeado
- **`compose.example.yml:52`**, **`compose.example.ai.yml:73`**
```yaml
SECRET_KEY_BASE: ${SECRET_KEY_BASE:-a7523c3d0ae56415046ad8abae168d71074a79534a7062258f8d1d51ac2f76d3c3bc86d86b6b0b307df30d9a6a90a2066a3fa9e67c5e6f374dbd7dd4e0778e13}
```
- **Impacto doble, verificado:**
  1. **Forjar cookies de sesión** → suplantación de cualquier usuario en la instancia.
  2. **Derivar claves de Active Record Encryption** → `config/initializers/active_record_encryption.rb:27-37` las calcula desde `SECRET_KEY_BASE` si el operador no configura las suyas. Con el default **y** un dump de la BD se descifran tokens de banco, secretos MFA y PII.
- El repo lo admite: `.env.example:222` y `config/initializers/encryption_warning.rb:13-18`.
- **Acción:** generar clave propia **antes** del primer arranque. Verificar con `docker compose exec web bin/rails runner 'puts Rails.application.secret_key_base[0,8]'` → si empieza por `a7523c3d`, cambiar ya.

#### B2 · Telemetría PostHog ACTIVA por defecto hacia un tercero
- **`config/initializers/posthog.rb:8`** → `ENV.fetch("POSTHOG_FEEDBACK_ENABLED", "true")` = **ON**
- **`config/initializers/posthog.rb:12-15`** → `api_key: "phc_D4stYbjnouz4J3H434HjYNQ655XG4vqL99HJH7cA4zMk"`, `host: "https://us.i.posthog.com"`
- **Qué sale:** 7 eventos de la encuesta del gráfico sankey + `sure_version` + rating. **Sin montos, sin cuentas, sin fechas** (saneador real en `app/javascript/utils/sankey_preview_analytics.mjs:83-100`).
- **Pero:** `sankey_preview_analytics.mjs:105` → `$geoip_disable = false` → **envía la IP para geolocalización**, lo que contradice el comentario de la línea 81 ("must not receive SDK page URLs, referrers, person data").
- **Acción:** `POSTHOG_FEEDBACK_ENABLED=false`.

#### B3 · `bin/codex-env` altera auth de Postgres y oculta cambios de git
- **`bin/codex-env:23-25`** → `sed` sobre `pg_hba.conf` pasando `peer` → **`trust`** (sin contraseña)
- **`bin/codex-env:49`** → comenta `ruby file: ".ruby-version"` del `Gemfile`
- **`bin/codex-env:51`** → `git update-index --assume-unchanged Gemfile Gemfile.lock` → **`git status` deja de mostrar los cambios**
- **No se ejecuta en ningún sitio crítico** (solo referenciado en `docs/llm-guides/harness-adapters.md:118` y `instruction-preservation-map.md:39`, que lo declara *"legacy… it changes system auth"*).
- **Acción:** borrarlo. Si ya se corrió: `git update-index --no-assume-unchanged Gemfile Gemfile.lock`.

---

### 🟠 PRIORIDAD 2 — Antes de conectar datos reales

| # | Hallazgo | file:line | Impacto | Acción |
|---|---|---|---|---|
| B4 | **Webhook de Plaid apunta al servidor del vendor** — `webhook: "https://app.sure.am/webhooks/plaid_eu"` | `lib/tasks/data_migration.rake:14` | Si se ejecuta, Plaid envía tus transacciones a `app.sure.am`. Tarea manual, **no invocada automáticamente** (`git grep eu_plaid_webhooks` → solo su definición). | **No ejecutar** `rake data_migration:eu_plaid_webhooks` |
| B5 | **App móvil por defecto apunta a `demo.sure.am`** | `mobile/lib/services/api_config.dart:11` y `:102-103` | Login/cuentas van al servidor del proyecto hasta cambiar la URL | Cambiar `_defaultBaseUrl` antes de distribuir el APK |
| B6 | **SSRF en SimpleFin** — `claim_url = Base64.decode64(setup_token)` → `self.class.post(claim_url)` sin allowlist de host | `app/models/provider/simplefin.rb:35,41` | Un admin puede hacer que el servidor haga POST a `169.254.169.254` o a la red interna | Allowlist `*.simplefin.org` + forzar `https` |
| B7 | **Reflexión insegura sin allowlist** — `mapping_params[:mappable_type]&.constantize` y `[:type]&.constantize` | `app/controllers/import/mappings_controller.rb:37,41` (params abiertos en `:17`) | Autoloading arbitrario de constantes. **Sin RCE**, query acotada a `family: Current.family`. El resto del repo **sí** usa allowlists (`settings/providers_controller.rb:103`, `snaptrade_items_controller.rb:637-647`) | Añadir allowlist `%w[Account Category Tag]` |
| B8 | **Signup de la API ignora `onboarding_state == "closed"`** | `app/controllers/api/v1/auth_controller.rb:15` + `config/routes.rb:739` vs `registrations_controller.rb:6,118-122` | Si cierras el registro en self-host, `POST /api/v1/auth/signup` sigue creando usuarios | Replicar `ensure_signup_open` como `before_action` |
| B9 | **Login de la API ignora SSO-only** | `api/v1/auth_controller.rb:75-84` vs `app/models/auth_config.rb:45-56` | Con `AUTH_LOCAL_LOGIN_ENABLED=false`, el móvil sigue aceptando contraseñas (MFA sí se verifica, hay rate limit) | Añadir `AuthConfig.local_login_allowed_for?` |
| B10 | **CORS `origins "*"`** en `/api/*`, `/oauth/*`, `/sessions/*` | `config/initializers/cors.rb:13` | Sin `credentials: true` → las cookies **no** viajan. Pero amplía superficie de lectura/CSRF | Restringir a `APP_URL` |
| B11 | **Sin Content-Security-Policy** — bloque entero comentado, mientras las vistas emiten `nonce=""` vacío | `config/initializers/content_security_policy.rb:6-26` | Anula la mitigación de XSS en una app financiera | Activar en `report_only` → luego `enforce` |
| B12 | **API key de demo predecible e irrevocable** | `app/models/api_key.rb:13` (`demo_monitoring_key_a1b2c3d4…`), `:25` `before_destroy :prevent_demo_monitoring_key_destroy!` | Solo si se corre `rake demo_data:default` en producción | No cargar demo data en prod |

---

### 🟡 PRIORIDAD 3 — Endurecimiento

| # | Hallazgo | file:line |
|---|---|---|
| B13 | **9 alertas de Brakeman silenciadas** (incluye redirect sin proteger, mass-assignment, *Dangerous Eval*, *Dynamic Render Path*) | `config/brakeman.ignore:6-206` |
| B14 | **Secreto interpolado en `run:`** en vez de `env:` (no explotable con base64, pero es anti-patrón) | `.github/workflows/flutter-build.yml:63` |
| B15 | **3 workflows sin bloque `permissions:`** | `chart-ci.yml`, `mobile-ci.yml`, `ci.yml` |
| B16 | **Imágenes sin digest** — `redis:latest`, `ollama:latest`, `openclaw:latest`, `open-webui` sin tag | `compose.example.yml:176`, `compose.example.ai.yml:208,233,261` |
| B17 | **Open WebUI sin autenticación** y `WEBUI_SECRET_KEY=t0p-s3cr3t` (solo profile `local-ai`, no arranca por defecto) | `compose.example.ai.yml:244,247` |

### 🟠 Hallazgo HISTÓRICO (ya eliminado)

#### B-H · Workflow de terceros con `contents: write` + cron + tag flotante
- **Commit `b05105722b77f969ef023a4ab4619c1e518fd1df`** (2026-07-15, "Add Gittensor stats") creó `.github/workflows/gittensor-impact.yml` con `cron` diario, `permissions: contents: write` y `uses: matthewevans/gittensor-impact-action@v1` (**tag mutable, no SHA**), más enlaces a `gittensor.io`.
- **Eliminado de `main`**: `7bf371906` (2026-08-11) y `c72236f23` (2026-08-12). **No está en HEAD.**
- **No se encontró payload malicioso** en el diff (solo generación de assets).
- ⚠️ **Sigue vivo en la historia y en la rama `origin/gittensor-impact-assets`.**

---

## 3. Verificaciones negativas (cosas que NO están)

Cada una confirmada por búsqueda explícita:

| Verificación | Resultado |
|---|---|
| Reverse shell / bind shell (`TCPSocket`, `UDPSocket`, `/dev/tcp`, `nc -e`, `pty.spawn`) | **0** |
| `curl \| bash` / `wget \| sh` en `bin/`, `config/`, `.github/`, Dockerfile | **0** |
| `Open3`, `IO.popen`, `Process.spawn`, `exec()` en código de app | **1** (`system("pdftoppm",...)` en array → sin shell, `provider/openai/pdf_processor.rb:232`) |
| `Marshal.load` / `YAML.load` inseguro | **0** (`YAML.load_file` con `permitted_classes` en `eval/dataset.rb:18`) |
| Blobs Base64 ≥60/80 chars en código de app | **0** |
| `chmod 777`, `sudo`, escritura en `~/.ssh` / `~/.aws` / `PATH` | **0** |
| IPs numéricas directas en código | **0** (solo User-Agents) |
| Dominios desconocidos / `webhook.site`, `pastebin`, `ngrok`, `telegram`, `discord webhooks` | **0** (la única mención de ngrok es un comentario, `application_helper.rb:241`) |
| Sockets crudos / DNS-over-HTTPS | **0** |
| WebAssembly (`.wasm`) | **0 archivos** |
| Scripts `postinstall`/`preinstall`/`prepare` en `package.json` | **0** |
| Fuentes `git:` / `path:` / URL externa en los 4 manifiestos | **0** (solo `rubygems.org`, `npmjs`, `pub.dev`, `crates.io`) |
| `.npmrc`, `.gemrc`, `.cargo/config`, mirrors alterados | **0** |
| Git hooks activos | **0** (solo los 15 `*.sample`) |
| Submódulos ocultos (`.gitmodules`, mode 160000) | **0** |
| `pull_request_target` / `workflow_run` en workflows | **0** |
| Actions sin pinear por SHA de 40 hex | **0** |
| Objetos git corruptos o dangling (`git fsck --full`) | **0** |
| Claves privadas / tokens reales en 3.602 commits | **0** (pickaxe `BEGIN … PRIVATE KEY`, `ghp_`, `AKIA` → 1 hit, es un *placeholder* de UI) |
| Binarios > 1.5 MB en toda la historia | **0** (máx: 1.02 MB, un PNG) |
| Migraciones con SQL peligroso, `GRANT`, `lo_import`, URLs con datos | **0** |
| Credenciales en VCR cassettes / CSVs | **0** (tokens enmascarados: `Bearer <OPENAI_ACCESS_TOKEN>`) |
| `serviceWorker` cacheando datos financieros | **0** (solo `/offline.html`) |
| Stash / refs replace / ramas con nombres maliciosos | **0** |

---

## 4. Fortalezas verificadas

| Área | Evidencia |
|---|---|
| **Trazabilidad imagen → código** | `org.opencontainers.image.revision: 789883081feda49fcfa8c7bc923cc77c9fe301ad` → commit **existe en tu clone** |
| **Continuidad con Maybe** | `git merge-base --is-ancestor 77b5469832758d1cbee1a940f3012a1ae1c74cd3 HEAD` → **CIERTO** (ese es el último commit de `maybe-finance/maybe`) |
| **Clon fiel a GitHub** | HEAD = `origin/main` = API de GitHub, mismo `tree`, 0 commits locales, reflog con 1 sola entrada (el clone) |
| **GitHub Actions** | 100% pineadas por SHA · `persist-credentials: false` en todos los checkout · CI corre **antes** de publicar (`publish.yml` → `needs: [ci]`) |
| **Contenedor** | `USER 1000:1000` (no-root) · multi-stage (sin toolchain en el final) · entrypoint solo hace `db:prepare` |
| **Compose** | Sin `privileged`, sin `network_mode: host`, sin `pid: host`, sin `docker.sock`, sin `/` montado · red bridge aislada |
| **Helm** | `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, secretos via `secretKeyRef` |
| **SQL** | Las 16 queries crudas usan `sanitize_sql_array` o bind params `$1..$n` |
| **Auth** | `rack_attack` rate limiting en cada endpoint de credenciales · `SuperAdminConstraint` *fail-closed* para Sidekiq · `filter_parameter_logging` filtra `passw, email, token, pin, code, bank_password` |
| **Dockerfile limpio** | 0 `curl\|bash`, 0 `chmod 777`, 0 `sudo`, imagen base oficial `ruby:3.4.9-slim` |
| **Higiene de firmas** | 0 firmas inválidas (`BADSIG`) en 3.602 commits; 3/3 muestreados → `verified: true, reason: valid` vía API |

---

## 5. Estado de la construcción (Nivel C — desde fuente)

| Campo | Valor |
|---|---|
| Repo | `/home/germoso/code/sure` (181 MB) |
| Imagen | **`sure:local`** — `sha256:0735b52e4561ef8df60df1898e40296422663d7d42e5c6ce53a28e017ce73519` |
| Config | `sha256:7f082f02083cedf71709106c84e6733c94fe55f081e4f59bc6426225baaaedaf` |
| Build arg | `BUILD_COMMIT_SHA=14638a70181d9adadc75e7d47d5b2975ee18bf46` (embebido en la imagen) |
| Dependencias | **277 gems** instalados desde `rubygems.org` (fuente única) |
| Estado | ✅ Construida · **⛔ NO EJECUTADA** (por decisión del usuario) |

---

## 6. Checklist de acción antes de arrancar

```bash
# 1. OBLIGATORIO — clave propia (generar una)
export SECRET_KEY_BASE=$(bin/rails secret)

# 2. Apagar telemetría
export POSTHOG_FEEDBACK_ENABLED=false

# 3. No habilitar IA si no la vas a usar (evita salida a OpenAI)
#    → dejar OPENAI_ACCESS_TOKEN vacío

# 4. NO ejecutar nunca:
#    rake data_migration:eu_plaid_webhooks
#    rake demo_data:default          (en producción)

# 5. Borrar el script que altera auth de Postgres
rm bin/codex-env
```

Pendientes de parche propio (o reportarlos upstream):
1. Allowlist de host en `provider/simplefin.rb:41`
2. Allowlist en `Import::MappingsController:37,41`
3. Guardas de `ensure_signup_open` y `local_login_allowed_for?` en `Api::V1::AuthController`
4. Activar CSP (`content_security_policy.rb`)
5. Restringir CORS a tu origen

---

## 7. Lo que NO se pudo verificar

| # | Límite | Por qué |
|---|---|---|
| 1 | **Provenance criptográfica de la imagen GHCR** | `publish.yml:147` usa `provenance: false` → **sin attestation SLSA ni SBOM ni firma cosign**. La trazabilidad existe (annotation con SHA) pero no está firmada. |
| 2 | **Contenido de las dependencias** — 277 gems, 533 crates, 115 pkgs Dart, 43 JS vendorizados | Solo se verificó *de dónde vienen y cómo se fijan*, no el código de adentro. Los 43 JS **no tienen hashes** en el repo. |
| 3 | **Firma criptográfica de ~3.600 commits** | Localmente `Can't check signature: No public key`. Solo 3 verificados vía API de GitHub. |
| 4 | **Comportamiento en runtime** | Regla de solo lectura: no se ejecutó la app ni se capturó tráfico. Puede existir llamada saliente no alcanzable por lectura estática. |
| 5 | **233 ramas remotas** | Se revisaron nombres y muestreos, no un diff de cada una. |
| 6 | **Quién controla los dominios de los bots** (`splashblot.com`, `sure.am`, `instinct.com`) y si siguen con permisos de escritura | Requiere acceso a settings de GitHub. |
| 7 | **Escaneo de CVEs de la imagen** | `trivy`/`grype`/`cosign` no están instalados en la máquina. |
| 8 | **Contenido de `config/credentials.yml.enc`** | Cifrado; `master.key` no está en el repo (lo cual confirma que la clave no fue commiteada). |
| 9 | **Imágenes de terceros en `compose.example.ai.yml`** (`pipelock`, `openclaw`, `open-webui`, `ollama`) | Su código no está en este repo. |

---

## 8. Veredicto final

> **🟢 LIMPIO — se puede self-hostear**, siempre que antes del primer arranque se cumplan los 3 puntos obligatorios:
> 1. **`SECRET_KEY_BASE` propio** (nunca el default `a7523c3d…`)
> 2. **`POSTHOG_FEEDBACK_ENABLED=false`**
> 3. **No ejecutar** `rake data_migration:eu_plaid_webhooks` ni `rake demo_data:default`
>
> Es una aplicación con postura de seguridad **por encima de la media** para el ecosistema self-hosted. Los 17 hallazgos son deuda técnica, no código hostil.

*Auditoría realizada por 6 agentes especializados + barrido mecánico. Todas las citas `file:line` fueron verificadas. Ninguna sección se completó sin ejecutar las búsquedas correspondientes.*
