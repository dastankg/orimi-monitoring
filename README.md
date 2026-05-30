# orimi-monitoring

Минимальный учебный observability-стек: **Prometheus + Grafana + Loki** (+ Promtail и cAdvisor).
Самостоятельный репозиторий — разворачивается отдельно от мониторируемых сервисов.

## Что внутри

| Сервис | Роль | Доступ |
|---|---|---|
| **Grafana** | UI: дашборды и Explore | http://127.0.0.1:3000 (admin / admin) |
| **Prometheus** | сбор метрик | http://127.0.0.1:9090 |
| **Loki** | хранилище логов | внутри сети |
| **Promtail** | сбор docker-логов → Loki | внутри сети |
| **cAdvisor** | метрики контейнеров | внутри сети |

## Как это связано с приложением

Стек и приложение (`orimi`, Django) лежат в **разных** репозиториях и связаны
не через git, а через общую docker-сеть `monitoring`. Prometheus скрейпит
контейнер приложения по имени напрямую внутри этой сети, минуя nginx.

Требования к мониторируемому сервису:
1. Контейнер приложения подключён к внешней сети `monitoring`.
2. Приложение отдаёт метрики на `/metrics` (в orimi — через `django-prometheus`).
3. Цель прописана в [`prometheus/prometheus.yml`](prometheus/prometheus.yml)
   (по умолчанию — `imagebot-web:8000`).

## Запуск

```bash
# 1. Создать общую сеть (один раз; её же должно использовать приложение)
docker network create monitoring

# 2. Поднять стек (из корня этого репозитория)
docker compose up -d
```

Приложение `orimi` поднимается из своего репозитория и должно быть подключено к
сети `monitoring` (см. его `docker-compose.yml`).

## Проверка

1. **Prometheus targets** — http://127.0.0.1:9090/targets
   Цели `django`, `cadvisor`, `prometheus` должны быть `UP`.
   (`django` будет `DOWN`, пока не запущено приложение в сети `monitoring` — это нормально.)

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

## Доступ к Grafana с сервера

Порты забинжены на `127.0.0.1` (наружу закрыты). Открыть UI с локальной машины — через SSH-туннель:

```bash
ssh -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 user@SERVER_IP
# затем локально открыть http://localhost:3000
```

## Остановка

```bash
docker compose down        # оставить данные
docker compose down -v     # удалить тома (prometheus_data, grafana_data, loki_data)
```

## Заметки

- **Сеть `monitoring`** должна существовать до `docker compose up` и быть общей
  с приложением. Если приложение в другой compose-сети — Prometheus не достучится.
- При нескольких воркерах gunicorn метрики приложения агрегируются через
  `PROMETHEUS_MULTIPROC_DIR` (настраивается на стороне приложения).
- `/metrics` приложения должен быть закрыт от публичного доступа (в orimi — `deny`
  в nginx); Prometheus ходит к контейнеру напрямую по сети `monitoring`.
- cAdvisor рассчитан на Linux-хост; на macOS/Docker Desktop часть метрик может
  быть недоступна.
- Promtail помечен Grafana как deprecated в пользу Alloy. Оставлен как самый
  простой и распространённый вариант для изучения.

## Дизайн

Подробное описание решений — в [`docs/2026-05-30-design.md`](docs/2026-05-30-design.md).