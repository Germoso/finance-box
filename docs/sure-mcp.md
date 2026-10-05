# Sure — referencia MCP

Cómo acceder a los datos financieros desde un asistente (esta guía está pensada
para que **cualquier sesión nueva** pueda trabajar sin contexto previo).

---

## 1. Qué es y dónde corre

| | |
|---|---|
| **App** | [Sure](https://github.com/we-promise/sure) — gestor de finanzas personales, fork de Maybe Finance |
| **URL** | `http://127.0.0.1:3000` (loopback) · `http://192.168.18.41:3000` (LAN) |
| **Despliegue** | Docker Compose, proyecto **`sure-deploy`**, en [`deploy/`](../deploy/) |
| **Servicios** | `web` · `worker` (Sidekiq) · `db` (Postgres 16) · `redis` |
| **Versión** | commit `14638a701` — ver [`UPSTREAM.md`](../UPSTREAM.md) |
| **Auditoría** | [`auditoria-sure.md`](auditoria-sure.md) → 🟢 sin código malicioso |

```bash
cd deploy && docker compose ps        # estado
docker compose logs -f web            # logs
```

---

## 2. Cómo conectarse

### A. Servidor MCP nativo en OpenCode ✅ (el configurado)

```bash
cd /home/germoso/code/finance-box
opencode mcp list
# ✓ azure-devops  connected
# ✓ sure          connected
```

Config en **`./opencode.json`** (config de **proyecto**, no global):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "sure": {
        "type": "remote",
        "url": "http://127.0.0.1:3000/mcp",
        "headers": { "Authorization": "Bearer <MCP_API_TOKEN>" }
      }
    }
  }
}
```

> **Por qué de proyecto y no global**: el alcance del MCP es `read_write`
> (incluye `delete_transaction`). Si estuviera en la config global, **cualquier
> sesión de cualquier directorio** tendría acceso a tus finanzas sin las reglas
> de `AGENTS.md`. Verificado:
> `jobradar → solo azure-devops` · `finance-box → azure-devops + sure`.
>
> `opencode.json` está en `.gitignore` (permisos **600**) porque lleva el
> token. En el repo va **`opencode.json.example`** con el placeholder.
> Tras clonar: `cp opencode.json.example opencode.json` y reemplazar el token
> por el de `deploy/.env`.

Las herramientas aparecen como **`tools.sure.*`**. Los esquemas las descubre
OpenCode solo (`search({namespace:"sure"})`); **no hace falta memorizarlos**.

### B. Fallback por HTTP (si el server nativo no conecta)

```bash
TOKEN=$(grep '^MCP_API_TOKEN=' deploy/.env | cut -d= -f2)

# 1. handshake
curl -X POST http://127.0.0.1:3000/mcp \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize"}'

# 2. descubrir herramientas
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'

# 3. ejecutar una
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call",
       "params":{"name":"get_accounts","arguments":{}}}'
```

`HTTP 401` → token incorrecto o `MCP_USER_EMAIL` no coincide con un usuario.

---

## 3. Las 21 herramientas

### Lectura

| Herramienta | Cuándo | Parámetros clave |
|---|---|---|
| `get_accounts` | ¿cuánto tengo/debo? | `include_balance_series` (historial) |
| `get_balance_sheet` | patrimonio neto y su evolución | `period`, `start_date`/`end_date`, `interval` |
| `get_income_statement` | ingresos/gastos por categoría en un periodo | `start_date`, `end_date`, `group_by: "month"`, `compare_previous_period` |
| `get_transactions` | buscar movimientos concretos | filtros por nombre, fecha, monto, cuenta, categoría, tag, `types` |
| `get_budget` | presupuesto del mes | `month` (`"2026-10"`), `prior_months` (0–11) |
| `get_holdings` | cartera de inversiones | `accounts`, `securities` |
| `get_recurring_transactions` | suscripciones y cuotas | `status`, `upcoming_within_days` |
| `get_merchants` | comercios disponibles | `search` |
| `get_categories` | árbol de categorías | paginación |
| `get_tags` | tags disponibles | paginación |
| `search_family_files` | buscar en documentos subidos | `query`, `max_results` |

### Escritura

| Herramienta | Notas |
|---|---|
| `create_transaction` | **negativo = ingreso, positivo = gasto** (o pasar `type`). Acepta `external_id` → **idempotente** para imports |
| `update_transaction` | nombre, notas, categoría, comercio, tags |
| **`delete_transaction`** | ⚠️ **irreversible — confirmar siempre antes** |
| `create_goal` | meta de ahorro; debe ligarse a cuentas *Depository* de una misma moneda |
| `create_tag` / `update_tag` | tags únicos por familia |
| `create_category` / `update_category` | 2 niveles; `parent_id` crea subcategoría |
| `update_budget` | `budgeted_spending`, `expected_income`, `categories[]` |
| `import_bank_statement` | requiere un `pdf_import_id` ya subido |

### Preview (no disponibles hasta activar *Settings → Preferences*)

`get_insights` · `get_bills` · `get_bill_details` · `get_bill_audit` ·
`get_paycheck_plan` · `get_statement_coverage` · `record_valuation` ·
`upload_account_statement` … (ver `docs/hosting/mcp.md` del upstream)

---

## 4. Contexto de esta instancia

| Dato | Valor | Cómo verificarlo |
|---|---|---|
| Moneda base | **DOP** 🇩🇴 | `SELECT currency FROM families` |
| Cotización USD→DOP | automática (`yahoo_finance`) | `EXCHANGE_RATE_PROVIDER` en `deploy/.env` |
| Tasa de referencia cargada | BCRD venta **60.6902** (2–5 oct 2026) | `SELECT * FROM exchange_rates` |
| Cuentas | consultar, **no dar por cierto** | `get_accounts` |
| Transacciones | consultar | `get_transactions` |

> **Hasta el 2026-10-05 había 0 transacciones**, 4 cuentas manuales
> (`is_linked: false`) y 1 usuario. Eso cambia: **siempre consultar**.

---

## 5. Seguridad

| Tema | Detalle |
|---|---|
| **Alcance del token** | `read_write` sobre toda la familia → incluye `delete_transaction` |
| **Dónde vive el token** | `deploy/.env` (**600**, fuera de git) y `~/.config/opencode/opencode.json` (**600**) |
| **Rotarlo** | regenerar `MCP_API_TOKEN` en `deploy/.env` → `docker compose up -d --force-recreate web worker` → el viejo muere |
| **Exposición** | puerto publicado solo en `127.0.0.1` y `192.168.18.41`. **Tailscale (`100.80.162.18`) NO publicado** (verificado: HTTP 000) |
| **Nunca subir** | `.env` a git, en ningún caso |

---

## 6. Troubleshooting

| Síntoma | Causa / solución |
|---|---|
| `401 unauthorized` | token mal o `MCP_USER_EMAIL` no coincide con un usuario |
| `opencode mcp list` → *needs authentication* o no aparece | falta el header. Re-crear la config **de proyecto** (sin `--global`):<br>`opencode mcp add sure --url http://127.0.0.1:3000/mcp --header "Authorization=Bearer $TOKEN"`<br>y asegurarse de quitar la entrada de `~/.config/opencode/opencode.json` si se había agregado con `--global` |
| OAuth falla con *"Refusing to send credentials to non-https"* | OpenCode exige TLS salvo en `localhost`/`127.0.0.1`. Con URL `192.168.18.41` nunca funciona. **Solución adoptada: token estático por header.** |
| `ERR_CONNECTION_REFUSED` en un puerto raro | es un **callback OAuth de un intento viejo** (puerto aleatorio, un solo uso). No se usa a mano; reintentar con `/mcps`. |
| Cotizaciones en 1:1 otra vez | falta la tasa. Ver [`setup.md`](setup.md) → troubleshooting IPv6 / `gai.conf` |
| La variable no llega al contenedor | debe estar en `x-app-env` de `deploy/compose.yml`; verificar con `docker compose config \| grep VAR` |
| Perder datos al mover la carpeta | el `name: sure-deploy` de `compose.yml` fija el proyecto. **No quitar.** |

---

## 7. Errores que ya se cometieron (no repetir)

1. **`x-app-env` no declaraba `MCP_API_TOKEN` ni `EXCHANGE_RATE_PROVIDER`** →
   `docker compose config` daba 0 ocurrencias y las variables nunca llegaban.
   Siempre verificar con `docker compose config | grep <VAR>`.
2. **tmpfs con sintaxis corta** → modo 775 (root) y la app no escribía en
   `/rails/tmp`. Usar forma larga con `mode: 0o1777`.
3. **`cap_drop: ALL` en redis** quitó `SETUID` → loop de
   `setpriv: setresuid failed`. Redis necesita `cap_add: [SETUID, SETGID]`.
4. **IPv6 sin ruta** → fetch de Yahoo intermitente (3/5). Arreglado con
   `gai.conf` + `disable_ipv6` (8/8), **no** forzando DNS a 8.8.8.8.
