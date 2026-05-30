# Стек мониторинга (Prometheus + Grafana + Loki)

Минимальный учебный observability-стек для сервиса `orimi` (Django).

Живёт отдельной папкой рядом с сервисами. Связь с `orimi` — через общую
docker-сеть `monitoring` (а не через git): Prometheus скрейпит контейнер
`imagebot-web` из репозитория orimi, подключённый к той же сети.

## Что внутри

| Сервис | Роль | Доступ |
|---|---|---|
| **Grafana** | UI: дашборды и Explore | http://127.0.0.1:3000 (admin / admin) |
| **Prometheus** | сбор метрик | http://127.0.0.1:9090 |
| **Loki** | хранилище логов | внутри сети |
| **Promtail** | сбор docker-логов → Loki | внутри сети |
| **cAdvisor** | метрики контейнеров | внутри сети |

## Запуск

```bash
# 1. Один раз создать общую сеть (её же использует orimi/docker-compose.yml)
docker network create monitoring

# 2. Пересобрать и поднять orimi с django-prometheus (в репозитории orimi)
cd orimi
docker compose up -d --build

# 3. Поднять стек мониторинга (в папке orimi-monitoring)
cd ../orimi-monitoring
docker compose up -d
```

## Проверка

1. **Prometheus targets** — http://127.0.0.1:9090/targets
   Цели `django`, `cadvisor`, `prometheus` должны быть `UP`.

2. **Метрики приложения** — Grafana → Explore → datasource Prometheus:
   ```promql
   rate(django_http_requests_total_by_method_total[5m])
   histogram_quantile(0.95, rate(django_http_requests_latency_seconds_by_view_method_bucket[5m]))
   ```

3. **Метрики контейнеров** — там же:
   ```promql
   container_memory_usage_bytes{name="imagebot-web"}
   rate(container_cpu_usage_seconds_total{name="imagebot-web"}[5m])
   ```

4. **Логи** — Grafana → Explore → datasource Loki:
   ```logql
   {container="imagebot-web"}
   {container="imagebot-web"} |= "ERROR"
   ```

## Остановка

```bash
docker compose down        # оставить данные
docker compose down -v     # удалить тома
```

## Заметки

- `/metrics` закрыт публично в `nginx.conf` (`deny all`). Prometheus скрейпит
  контейнер `imagebot-web:8000` напрямую по сети `monitoring`.
- При 4 воркерах gunicorn метрики агрегируются через
  `PROMETHEUS_MULTIPROC_DIR` (см. `orimi/gunicorn.conf.py`).
- cAdvisor рассчитан на Linux-хост; на macOS/Docker Desktop часть метрик
  может быть недоступна — на боевом Linux-сервере работает полностью.
- Promtail помечен Grafana как deprecated в пользу Alloy. Для учёбы оставлен
  как самый простой и распространённый вариант.