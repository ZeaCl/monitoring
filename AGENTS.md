# AGENTS.md — monitoring

Repositorio de observabilidad del ecosistema ZEA. Stack: **Prometheus + Grafana + exporters**.

## 🎯 Objetivo

Proveer dashboards, alertas y métricas históricas para todos los microservicios del ecosistema FundManagement y servicios core de ZEA Platform.

## 📦 Estructura esperada

```
monitoring/
├── docker-compose.yml              ← Prometheus + Grafana + exporters (autoconteido)
├── .env.example
├── prometheus/
│   └── prometheus.yml              ← scrape config: targets, intervalos
├── grafana/
│   ├── dashboards/
│   │   ├── eda.json                ← eventos Redis, jobs Oban, proyecciones
│   │   ├── postgres.json           ← conexiones, queries lentas, locks
│   │   ├── services-health.json    ← uptime, latencia, errores por servicio
│   │   └── business.json           ← funds creados, payments, LPs
│   └── provisioning/
│       ├── datasources.yml         ← conecta Grafana → Prometheus
│       └── dashboards.yml          ← carga automática de dashboards
├── alerts/
│   └── rules.yml                   ← Alertmanager rules
└── README.md
```

## 🔗 Integración con el ecosistema

Este repo es un **servicio más** en el compose de producción. Se agrega a `zea/zea/docker-compose.prod.yml`:

```yaml
monitoring-prometheus:
  image: prom/prometheus:v2.54.0
  volumes:
    - ../monitoring/prometheus:/etc/prometheus
  command: --config.file=/etc/prometheus/prometheus.yml
  ports: ["9090:9090"]

monitoring-grafana:
  image: grafana/grafana:11.2.0
  volumes:
    - ../monitoring/grafana/dashboards:/etc/grafana/dashboards
    - ../monitoring/grafana/provisioning:/etc/grafana/provisioning
  environment:
    GF_SECURITY_ADMIN_USER: ${GRAFANA_USER:-admin}
    GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
  ports: ["3000:3000"]

monitoring-postgres-exporter:
  image: prometheuscommunity/postgres-exporter:v0.16.0
  environment:
    DATA_SOURCE_NAME: "postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/fundmanagement?sslmode=disable"
  ports: ["9187:9187"]

monitoring-redis-exporter:
  image: oliver006/redis_exporter:v1.62.0
  environment:
    REDIS_ADDR: "redis:6379"
    REDIS_PASSWORD: ${REDIS_PASSWORD}
  ports: ["9121:9121"]
```

## 📊 Servicios a monitorear

Cada servicio necesita exponer `/metrics` en formato Prometheus:

| Servicio | Puerto | Agregar `prom_ex` |
|---|---|---|
| fm_funds | 4082 | ✅ |
| fm_payments | 4085 | ✅ |
| fm_investors | 4086 | ✅ |
| fm_capital_calls | 4083 | ✅ |
| fm_commitments | 4087 | ✅ |
| thalamus | 4000 | ✅ |
| cranium | 4000 | ✅ |
| cerebelum | 4001 | ✅ |
| postgres | 5432 | ← postgres_exporter |
| redis | 6379 | ← redis_exporter |

### Instrumentación por servicio (ejemplo Elixir)

```elixir
# mix.exs
{:prom_ex, "~> 1.9"}

# config/config.exs
config :fm_funds, FmFundsWeb.Endpoint,
  prom_ex: [
    plugins: [
      PromEx.Plugins.Ecto,
      PromEx.Plugins.Oban,
      PromEx.Plugins.Application
    ]
  ]

# router.ex
get("/metrics", do: conn |> send_resp(200, PromEx.export()))
```

## 🚀 Para arrancar

```bash
cd zea/monitoring
cp .env.example .env
docker compose up -d

# Grafana en http://localhost:3000 (admin / ver .env)
# Prometheus en http://localhost:9090
```

## 📋 Issues iniciales (ver GitHub Issues)

1. **Setup del stack base** — docker-compose, Prometheus config, Grafana provisioning
2. **Dashboard EDA** — eventos Redis, jobs Oban, proyecciones locales
3. **Dashboard Postgres** — conexiones, queries, locks
4. **Dashboard Services Health** — uptime, latencia, errores
5. **Dashboard Business** — métricas de negocio (funds, payments, LPs)
6. **Alertas** — reglas para too_many_connections, servicios caídos, lag de proyecciones
7. **README + docs** — cómo agregar un servicio nuevo, cómo crear un dashboard
8. **Integración con compose prod** — PR a zea/zea agregando los servicios

## 🔗 Referencias

- `../zea/zea/docker-compose.prod.yml` — compose de producción donde se integra
- `../zea/infra/issues/6` — problema de conexiones Postgres que este repo ayuda a detectar
- `../domains/fundmanagement_subdomains_libs/fm_funds/.wiki/eda-architecture.md` — eventos a monitorear
- `prom_ex` docs: https://hexdocs.pm/prom_ex
