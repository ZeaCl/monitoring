# 📊 ZEA Monitoring Stack

Observabilidad completa del ecosistema ZEA: **Prometheus + Grafana + Alertmanager + exporters**.

## 🏗️ Arquitectura

```
┌──────────────────────────────────────────────────────────┐
│                    Grafana :3000                          │
│  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────────┐ │
│  │   EDA    │ │ Postgres │ │ Health │ │   Business    │ │
│  └──────────┘ └──────────┘ └────────┘ └──────────────┘ │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────┴───────────────────────────────────┐
│                  Prometheus :9090                         │
│     ┌──────────────────────────────────────┐             │
│     │  19 alert rules → Alertmanager :9093 │             │
│     └──────────────────────────────────────┘             │
└──────┬──────────────┬──────────────┬─────────────────────┘
       │              │              │
  ┌────┴────┐   ┌────┴────┐   ┌────┴─────────────┐
  │ postgres │   │  redis  │   │ fm_* / thalamus  │
  │ exporter │   │ exporter│   │ cranium / soma   │
  │  :9187   │   │  :9121  │   │ /metrics (c/u)   │
  └─────────┘   └─────────┘   └──────────────────┘
```

## 🚀 Quickstart

```bash
cd monitoring
cp .env.example .env
docker compose up -d
```

| Servicio | URL | Default credentials |
|---|---|---|
| **Grafana** | http://localhost:3000 | `admin` / `admin` (cambiar en `.env`) |
| **Prometheus** | http://localhost:9090 | — |
| **Alertmanager** | http://localhost:9093 | — |

## 📁 Estructura

```
monitoring/
├── docker-compose.yml              ← Prometheus + Grafana + Alertmanager + exporters
├── .env.example                    ← Variables de entorno
├── prometheus/
│   └── prometheus.yml              ← Scrape config: targets, intervalos, alertmanager
├── grafana/
│   ├── dashboards/
│   │   ├── eda.json                ← Eventos Redis, jobs Oban, proyecciones
│   │   ├── postgres.json           ← Conexiones, queries, locks, cache
│   │   ├── services-health.json    ← Uptime, latencia, errores, BEAM
│   │   └── business.json           ← Funds, payments, LPs, capital calls
│   └── provisioning/
│       ├── datasources/default.yml  ← Prometheus como datasource default
│       └── dashboards/default.yml   ← Auto-load desde grafana/dashboards/
├── alerts/
│   └── rules.yml                   ← 19 reglas de alerta (Postgres, Redis, EDA, services, BEAM)
├── alertmanager/
│   └── alertmanager.yml            ← Routing por severidad, notificaciones
└── .env.example
```

## 📊 Dashboards

### 📡 EDA — Eventos, Jobs & Proyecciones
Métricas de Redis, Oban, proyecciones y message consumers.
- Redis: throughput, clients, memory, keys per DB
- Oban: job rate, latency p95, queue length, success rate
- Proyecciones: lag, eventos procesados, errores
- Consumers: stream messages, PEL, processing latency

Template: `$service` → filtra por fm_*, thalamus, cranium, cerebelum.

### 🗄️ Postgres — Conexiones, Queries & Locks
Métricas del postgres-exporter para todas las bases.
- Conexiones activas, pool %, sesiones por estado
- Transaction rate, row ops (CRUD), deadlocks
- Locks por tipo, sesiones esperando, max TX duration
- Cache hit ratio, disk vs cache reads, DB size, vacuum

Template: `$database` → filtra por base (thalamus_prod, cranium_prod, etc.).

### 🏥 Services Health — Uptime, Latencia & Errores
Estado de todos los servicios del ecosistema.
- Up/down grid (todos los servicios)
- HTTP request rate, latencia p50/p95/p99, heatmap
- BEAM memory & process count
- Ecto query rate & latency p95
- Status codes 2xx/4xx/5xx, 5xx error rate %

Template: `$service` → todos los jobs registrados.

### 💼 Business Metrics — Funds, Payments & LPs
KPIs de negocio (requiere instrumentación custom en fm_*).
- Overview: total funds, payments, committed, called, LPs, distributions
- Funds por día, por tipo, committed by fund
- Payments: volumen, monto USD, por status
- LPs: evolución, committed vs called, call progress %
- Capital calls: emitidas, distribuciones, fulfillment rate

Template: `$org` → ZEA, Südlich, etc.

## 🚨 Alertas

19 reglas en 6 grupos:

| Grupo | Reglas | Severidad |
|---|---|---|
| **postgres** | TooManyConnections (>80%), Critical (>95%), Deadlocks, LowCacheHitRatio (<90%), ReplicationLag | warning / critical |
| **redis** | OutOfMemory (>90%), MemoryCritical (>98%), TooManyConnections (>500) | warning / critical |
| **services** | ServiceDown, ServiceFlapping, HighErrorRate (>5%), HighErrorRateCritical (>20%), HighLatency (p95>5s) | warning / critical |
| **eda** | EDALagHigh (>5m), EDALagCritical (>30m), EventsBacklog (>1k), ObanQueueBacklog (>500) | warning / critical |
| **elixir** | HighMemoryUsage (>1GB), ProcessCountHigh (>100k) | warning |

### Configurar notificaciones

Editar `alertmanager/alertmanager.yml`:

```yaml
receivers:
  - name: "default"
    slack_configs:
      - channel: "#zea-alerts"
        api_url: ${SLACK_WEBHOOK_URL}
        title: "[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

Las alertas `critical` van a un receiver separado (ej: `#zea-oncall`).

## ➕ Agregar un servicio nuevo

### 1. Instrumentar el servicio

Agregar `prom_ex` en Elixir:

```elixir
# mix.exs
{:prom_ex, "~> 1.9"}

# config/config.exs
config :mi_servicio, MiServicioWeb.Endpoint,
  prom_ex: [
    plugins: [
      PromEx.Plugins.Ecto,
      PromEx.Plugins.Oban,
      PromEx.Plugins.Phoenix,
      PromEx.Plugins.Application,
      PromEx.Plugins.BEAM
    ]
  ]

# router.ex
get("/metrics", do: conn |> send_resp(200, PromEx.export()))
```

### 2. Agregar target en Prometheus

```yaml
# prometheus/prometheus.yml
- job_name: "mi_servicio"
  static_configs:
    - targets: ["mi_servicio:4084"]
```

### 3. Agregar a los dashboards

Los dashboards de **Services Health** y **EDA** usan template variable `$service` (regex `fm_.*|thalamus|cranium|cerebelum|soma|...`). Si tu servicio matchea ese regex, aparece automáticamente. Si no, editar la regex en el dashboard.

### 4. Agregar métricas de negocio (opcional)

Para el dashboard **Business**, exponer métricas custom desde la app:

```elixir
# En tu Application o GenServer
PromEx.Metrics.Counter.declare(
  name: :funds_created_total,
  help: "Total funds created",
  labels: [:org, :fund_type]
)

PromEx.Metrics.Counter.inc(name: :funds_created_total, labels: ["ZEA", "venture"])
```

## 📝 Crear un dashboard nuevo

1. Crear el JSON en `grafana/dashboards/mi-dashboard.json`
2. El provisioning lo carga automáticamente en Grafana (<10s)
3. También se puede crear desde la UI de Grafana y exportar el JSON

## 🔗 Integración con compose de producción

Ver `AGENTS.md` para el snippet de docker-compose.prod.yml.

Los exporters necesitan acceso de red a los servicios:
- `postgres-exporter` → `postgres:5432`
- `redis-exporter` → `redis:6379`

Prometheus scrapea a los exporters y a los `/metrics` de cada servicio (cuando estén en la misma red Docker).

## 📐 Puertos

| Servicio | Puerto | Descripción |
|---|---|---|
| Prometheus | 9090 | Scraping + UI |
| Grafana | 3000 | Dashboards |
| Alertmanager | 9093 | Gestión de alertas |
| postgres-exporter | 9187 | Métricas Postgres |
| redis-exporter | 9121 | Métricas Redis |

## 🛠️ Operaciones

```bash
# Ver estado de todos los servicios
docker compose ps

# Ver logs
docker compose logs -f prometheus

# Recargar configuración de Prometheus sin reiniciar
curl -X POST http://localhost:9090/-/reload

# Ver reglas de alerta cargadas
curl -s http://localhost:9090/api/v1/rules | jq '.data.groups[] | .name, (.rules[] | .name)'

# Ver alertas disparadas
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | select(.state=="firing")'

# Ver estado de Alertmanager
curl -s http://localhost:9093/api/v2/status | jq
```
