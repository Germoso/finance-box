# Sure — despliegue local

Instancia self-hosteada de **Sure** (gestor de finanzas personales, fork de Maybe Finance).

## Estado

| | |
|---|---|
| **URL** | http://192.168.18.41:3000 |
| **Registro** | http://192.168.18.41:3000/registration/new ← **el primer usuario queda `super_admin`** |
| **Imagen** | `sure:local` — construida desde fuente en `/home/germoso/code/sure` |
| **Commit** | `14638a70181d9adadc75e7d47d5b2975ee18bf46` (embebido en la imagen) |
| **Creado** | 2026-10-04 |
| **Auditoría** | [`../docs/auditoria-sure.md`](../docs/auditoria-sure.md) → 🟢 sin código malicioso |

## Archivos

| Archivo | Qué es |
|---|---|
| `compose.yml` | Orquestación **endurecida** (ver abajo) |
| `.env` | Secretos generados localmente. **permisos 600, NO subir a git** |

## Comandos

```bash
cd /home/germoso/code/finance-box/deploy

docker compose ps                 # estado
docker compose logs -f web        # logs de la app
docker compose logs -f worker     # logs de Sidekiq
docker compose restart web        # reiniciar
docker compose down               # parar (conserva datos)
docker compose up -d              # arrancar

# Health check
curl -fsS http://192.168.18.41:3000/up && echo OK

# Backup de la base de datos
docker compose exec -T db pg_dump -U sure sure_production | gzip > "backup-$(date +%F).sql.gz"
```

## Endurecimiento aplicado (vs. `compose.example.yml` del upstream)

| Control | Valor |
|---|---|
| `privileged` | `false` |
| `security_opt` | `no-new-privileges:true` |
| `cap_drop` | `ALL` (web/worker/redis) · db conserva solo `CHOWN SETUID SETGID FOWNER DAC_OVERRIDE` |
| `read_only` rootfs | `true` (escritura solo en `/tmp`, `/rails/tmp`, `/rails/log` como tmpfs `0o1777`, y el volumen `/rails/storage`) |
| `mem_limit` | web 1g · worker 768m · db 512m · redis 256m |
| `pids_limit` | web 300 · worker 200 · db 100 · redis 50 |
| `user` | `1000:1000` (no-root) |
| Puertos | **solo `192.168.18.41:3000`** — no expuesto en las demás interfaces |
| `docker.sock` | **no montado** |
| Red | bridge aislada `sure_net` |

### Privacidad (hallazgos de la auditoría)

| Hallazgo | Estado |
|---|---|
| **B1** `SECRET_KEY_BASE` público hardcodeado | ✅ clave propia generada (`68ec8b62…`) |
| **B1** Claves de Active Record Encryption | ✅ 3 claves explícitas en `.env` (no derivadas de la clave pública) |
| **B2** Telemetría PostHog activa por defecto | ✅ `POSTHOG_FEEDBACK_ENABLED=false` |
| IA → OpenAI / Sentry / Skylight / Langfuse / Logtail | ✅ todos vacíos — **nada sale de la máquina** |
| **B3** `bin/codex-env` (altera auth de Postgres) | ✅ nunca ejecutado; el build solo copia el código |
| **B4** `rake data_migration:eu_plaid_webhooks` | ⚠️ **nunca ejecutarla** |

## MCP — conexión desde un asistente de IA

Configurado el **2026-10-04** · `docs/hosting/mcp.md`

| | |
|---|---|
| **Endpoint** | `POST http://192.168.18.41:3000/mcp` |
| **Auth** | `Authorization: Bearer $MCP_API_TOKEN` |
| **Token** | generado localmente (64 hex) en `.env` — **permisos 600** |
| **Usuario vinculado** | `MCP_USER_EMAIL` en `.env` (la cuenta `super_admin`) |
| **Verificado** | `initialize` → HTTP 200 · `tools/list` → **21 herramientas** · token inválido → HTTP 401 |

**Herramientas expuestas** (todas las del registro base):
`get_transactions` · `get_recurring_transactions` · `get_accounts` · `get_holdings` ·
`get_balance_sheet` · `get_income_statement` · `get_budget` · `get_merchants` ·
`get_tags` · `get_categories` · `search_family_files` · `import_bank_statement` ·
`create_goal` · `create_tag` · `update_tag` · `create_category` · `update_category` ·
`update_transaction` · `create_transaction` · **`delete_transaction`** · `update_budget`

> ⚠️ **El token da acceso de LECTURA Y ESCRITURA a toda la familia**, incluido
> `delete_transaction`. Tratarlo como una credencial de nivel administrador.
> Para rotarlo: regenerar en `.env` y `docker compose up -d --force-recreate web worker`.

### No incluido (preview features apagadas)

`get_insights` · `get_bills` · `get_bill_audit` · `get_paycheck_plan` ·
`get_statement_coverage` · `record_valuation` · `upload_account_statement`…

→ aparecen al activar **Settings → Preferences → Preview features**.

### Ejemplo

```bash
TOKEN=$(grep '^MCP_API_TOKEN=' .env | cut -d= -f2)
curl -X POST http://192.168.18.41:3000/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"get_accounts","arguments":{}}}'
```

---

## Cotizaciones automáticas (USD → DOP)

Configurado el **2026-10-04**.

| | |
|---|---|
| **Proveedor** | `EXCHANGE_RATE_PROVIDER=yahoo_finance` → `Provider::YahooFinance` |
| **Cache** | 8 filas, rango `2026-07-30 .. 2026-10-05` |
| **Tasa de referencia** | BCRD venta `60.6902` (2 a 5 oct 2026) cargada a mano |
| **Job programado** | `ImportMarketDataJob` — diario 22:00 (lun–vie), sincroniza FX + precios |
| **Bajo demanda** | `ExchangeRate.find_or_fetch_rate` cachea al primer uso |
| **Verificado** | 8/8 fetch exitosos (antes: 3/5) |

### Problema resuelto: IPv6 sin ruta

**Síntoma**: fetch intermitente a `query1.finance.yahoo.com` (3/5 y 4/5 OK).

**Causa** (medida dentro del contenedor):
```
getent hosts      → solo AAAA (2001:4998:...)
ip -6 route       → SIN ruta IPv6 de salida
/etc/gai.conf     → vacío → glibc prefiere IPv6
Addrinfo Ruby     → [IPv6, IPv6, IPv4, IPv4]  ← intentaba IPv6 primero
```

**Corrección aplicada**: montar `./gai.conf` en modo solo lectura con
`precedence ::ffff:0/96 100`, más `sysctls: net.ipv6.conf.*.disable_ipv6=1`.

```
Addrinfo Ruby     → [69.147.92.12, 69.147.92.11, IPv6...]  ← IPv4 primero
resultado         → 8/8 ✅
```

> Se eligió esta vía en lugar de la del upstream (`dns: 8.8.8.8 / 1.1.1.1`)
> **para no enviar toda tu resolución DNS a Google/Cloudflare**. Efecto
> colateral positivo: se evita cualquier fuga por IPv6.

### Notas

- `ExchangeRate.provider` solo se lee de `ENV["EXCHANGE_RATE_PROVIDER"]` o de
  `Setting.exchange_rate_provider`. **Ambas variables deben estar en `x-app-env`**
  de `compose.yml`, o compose no las pasa al contenedor (mismo bug que con MCP).
- `DataCacheClearJob` hace `ExchangeRate.delete_all` — **no** está en
  `config/schedule.yml`, así que no se ejecuta solo. Si algún día se encola,
  las tasas se vuelven a traer bajo demanda.

---

## Notas

- **Puerto**: atado a `192.168.18.41`. Si tu IP cambia por DHCP, editar `PORT_BIND` en `.env` o usar `"3000:3000"` (expone en todas las interfaces).
- **RAM**: el stack usa ~1 GB con tope de 2.5 GB por `mem_limit`; el host tiene 32 GB de swap como red de seguridad.
- **DNS**: usa el del host (el upstream fuerza `8.8.8.8`/`1.1.1.1`; se quitó por privacidad). Si la sincronización de Yahoo Finance fallara, agregar bajo cada servicio:
  ```yaml
    dns:
      - 8.8.8.8
      - 1.1.1.1
  ```
- **Parches pendientes** (reportar upstream o aplicar): allowlist en `provider/simplefin.rb:41` y `import/mappings_controller.rb:37,41`; guardas en `api/v1/auth_controller.rb`; activar CSP.
