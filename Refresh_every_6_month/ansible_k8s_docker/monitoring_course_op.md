# Мониторинг для DevOps: Ежегодный/Полугодовой курс-освежитель

**Цель:** Освежить в памяти ключевые концепции мониторинга за 2-3 часа практики и узнать 1-2 новые продвинутые техники.

**Формат:** Каждый раздел состоит из:
1. **Краткой теории (Напоминалка)**: Самое главное, что вы могли забыть
2. **Практического задания**: Реальная задача, которую нужно решить
3. **Бонусного задания (для роста)**: Задача посложнее или с использованием новой фичи

**Предварительные требования:**
- Базовое понимание Linux/Unix
- Доступ к серверу или виртуальной машине
- Docker установлен (для большинства заданий)
- Базовые знания командной строки

---

## Модуль 1: Основы мониторинга и метрики (20 минут)

### 🎯 Напоминалка

**Четыре золотых сигнала (Four Golden Signals):**
```
1. Latency (Задержка)      - Время ответа на запросы
2. Traffic (Трафик)        - Количество запросов
3. Errors (Ошибки)         - Процент неудачных запросов
4. Saturation (Насыщение)  - Загрузка ресурсов (CPU, память, диск)
```

**Типы метрик:**
```
Counter   - Монотонно возрастающее значение (запросы, ошибки)
Gauge     - Текущее значение (CPU, память, температура)
Histogram - Распределение значений (latency buckets)
Summary   - Статистика за период (percentiles)
```

**USE Method (для ресурсов):**
```
Utilization - Среднее время занятости ресурса
Saturation  - Степень перегрузки
Errors      - Количество ошибок
```

**RED Method (для сервисов):**
```
Rate     - Запросов в секунду
Errors   - Количество ошибок
Duration - Время ответа
```

**Уровни мониторинга:**
```
┌─────────────────────────────────┐
│   Application (APM)             │  - Код, транзакции
├─────────────────────────────────┤
│   Service/Container             │  - Docker, K8s
├─────────────────────────────────┤
│   Operating System              │  - CPU, RAM, Disk
├─────────────────────────────────┤
│   Infrastructure                │  - Network, Hardware
└─────────────────────────────────┘
```

**Ключевые метрики Linux:**
```bash
# CPU
top, htop
mpstat -P ALL 1

# Memory
free -m
vmstat 1

# Disk I/O
iostat -x 1
iotop

# Network
iftop
nethogs
ss -s

# Process
ps aux --sort=-%mem | head
ps aux --sort=-%cpu | head
```

**Метрики приложений:**
```
- Request rate (req/s)
- Error rate (%)
- Response time (ms) - p50, p95, p99
- Active connections
- Queue depth
- Database query time
- Cache hit ratio
```

### 💻 Задание

Настрой базовый мониторинг системы:

1. **Установи и запусти Node Exporter** (для сбора метрик хоста):
```bash
# Через Docker
docker run -d \
  --name node-exporter \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  prom/node-exporter:latest \
  --path.rootfs=/host

# Проверка
curl http://localhost:9100/metrics | head -20
```

2. **Изучи основные метрики**:
```bash
# CPU
curl -s http://localhost:9100/metrics | grep node_cpu_seconds_total

# Memory
curl -s http://localhost:9100/metrics | grep node_memory

# Disk
curl -s http://localhost:9100/metrics | grep node_disk

# Network
curl -s http://localhost:9100/metrics | grep node_network
```

3. **Создай простой bash скрипт** для мониторинга (`monitor.sh`):
```bash
#!/bin/bash

echo "=== System Monitoring Report ==="
echo "Date: $(date)"
echo ""

# CPU Usage
echo "CPU Usage:"
top -bn1 | grep "Cpu(s)" | awk '{print "  User: " $2 ", System: " $4 ", Idle: " $8}'

# Memory Usage
echo ""
echo "Memory Usage:"
free -h | awk 'NR==2{printf "  Total: %s, Used: %s (%.2f%%)\n", $2, $3, $3*100/$2}'

# Disk Usage
echo ""
echo "Disk Usage:"
df -h / | awk 'NR==2{printf "  Total: %s, Used: %s (%s)\n", $2, $3, $5}'

# Load Average
echo ""
echo "Load Average:"
uptime | awk -F'load average:' '{print "  " $2}'

# Top 5 processes by CPU
echo ""
echo "Top 5 processes by CPU:"
ps aux --sort=-%cpu | head -6 | tail -5 | awk '{printf "  %s: %.1f%%\n", $11, $3}'

# Top 5 processes by Memory
echo ""
echo "Top 5 processes by Memory:"
ps aux --sort=-%mem | head -6 | tail -5 | awk '{printf "  %s: %.1f%%\n", $11, $4}'
```

4. Запусти скрипт:
```bash
chmod +x monitor.sh
./monitor.sh
```

### 🚀 Бонус (новое)

**Настрой cAdvisor** для мониторинга Docker контейнеров:
```bash
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  gcr.io/cadvisor/cadvisor:latest

# Открой в браузере
http://localhost:8080
```

**Создай свой custom exporter** на Python:
```python
# custom_exporter.py
from prometheus_client import start_http_server, Gauge, Counter
import time
import random

# Создаем метрики
request_gauge = Gauge('app_requests_in_progress', 'Number of requests in progress')
request_counter = Counter('app_requests_total', 'Total number of requests')
error_counter = Counter('app_errors_total', 'Total number of errors')

def process_request():
    """Симулируем обработку запроса"""
    request_gauge.inc()
    request_counter.inc()
    
    # Случайная ошибка
    if random.random() < 0.1:
        error_counter.inc()
    
    time.sleep(random.uniform(0.1, 0.5))
    request_gauge.dec()

if __name__ == '__main__':
    start_http_server(8000)
    print("Exporter started on port 8000")
    
    while True:
        process_request()
        time.sleep(random.uniform(0.5, 2))
```

---

## Модуль 2: Prometheus - сбор и хранение метрик (30 минут)

### 🎯 Напоминалка

**Архитектура Prometheus:**
```
┌─────────────┐
│   Targets   │ ← HTTP Pull (scrape)
│  (Metrics)  │
└──────┬──────┘
       │
   ┌───▼────┐
   │ Prom-  │
   │ etheus │ ← Time Series DB (TSDB)
   │ Server │
   └───┬────┘
       │
   ┌───▼────┐
   │ Alert- │
   │ manager│
   └────────┘
```

**Prometheus config structure:**
```yaml
global:
  scrape_interval: 15s      # Как часто собирать метрики
  evaluation_interval: 15s  # Как часто проверять правила

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

**PromQL основы:**
```promql
# Instant vector - текущее значение
node_cpu_seconds_total

# Range vector - значения за период
node_cpu_seconds_total[5m]

# Фильтры
node_cpu_seconds_total{mode="idle"}
node_cpu_seconds_total{mode!="idle"}
node_cpu_seconds_total{mode=~"user|system"}

# Агрегация
sum(node_cpu_seconds_total)
avg(node_cpu_seconds_total)
max(node_cpu_seconds_total)
min(node_cpu_seconds_total)
count(node_cpu_seconds_total)

# По label
sum(node_cpu_seconds_total) by (mode)
sum(node_cpu_seconds_total) by (cpu)

# Функции
rate(node_cpu_seconds_total[5m])           # Скорость изменения
irate(node_cpu_seconds_total[5m])          # Мгновенная скорость
increase(node_cpu_seconds_total[5m])       # Увеличение за период
delta(node_cpu_seconds_total[5m])          # Изменение
```

**Распространенные запросы:**
```promql
# CPU utilization
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage %
100 - ((node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100)

# Network traffic
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# HTTP request rate
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Latency percentiles (для histogram)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**Metric types в деталях:**
```promql
# Counter - только растет
http_requests_total
# Используй rate() или increase()
rate(http_requests_total[5m])

# Gauge - может расти и падать
node_memory_MemAvailable_bytes
# Используй напрямую или с функциями агрегации
avg(node_memory_MemAvailable_bytes)

# Histogram - распределение значений
http_request_duration_seconds_bucket
http_request_duration_seconds_sum
http_request_duration_seconds_count
# Используй histogram_quantile()
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Summary - предрасчитанные квантили
http_request_duration_seconds{quantile="0.95"}
```

**Recording rules** (для оптимизации):
```yaml
groups:
  - name: example
    interval: 30s
    rules:
    - record: job:node_cpu_utilization:avg
      expr: 100 - (avg by (job) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Alerting rules:**
```yaml
groups:
  - name: alerts
    rules:
    - alert: HighCPUUsage
      expr: job:node_cpu_utilization:avg > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.instance }}"
        description: "CPU usage is {{ $value }}%"
```

### 💻 Задание

Настрой полноценный Prometheus:

1. **Создай docker-compose.yml**:
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    pid: host
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    restart: unless-stopped

volumes:
  prometheus-data:
```

2. **Создай prometheus.yml**:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Загрузка правил алертов
rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

3. **Создай alerts.yml**:
```yaml
groups:
  - name: system_alerts
    rules:
    - alert: HighCPUUsage
      expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "CPU usage is above 80% (current value: {{ $value }}%)"

    - alert: HighMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Memory usage is above 90% (current value: {{ $value }}%)"

    - alert: DiskSpaceLow
      expr: (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100 > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Low disk space"
        description: "Disk usage is above 85% (current value: {{ $value }}%)"

    - alert: InstanceDown
      expr: up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Instance {{ $labels.instance }} down"
        description: "{{ $labels.instance }} has been down for more than 1 minute"
```

4. **Запусти stack**:
```bash
docker-compose up -d

# Проверка
docker-compose ps
curl http://localhost:9090/api/v1/targets
```

5. **Открой Prometheus UI** и попробуй запросы:
```
Перейди: http://localhost:9090

Попробуй запросы:
- node_cpu_seconds_total
- rate(node_cpu_seconds_total[5m])
- 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
- node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

### 🚀 Бонус (новое)

**Настрой Service Discovery** для автоматического обнаружения целей:

**File-based SD** (`file_sd.json`):
```json
[
  {
    "targets": ["node-exporter:9100"],
    "labels": {
      "job": "node",
      "env": "production"
    }
  },
  {
    "targets": ["cadvisor:8080"],
    "labels": {
      "job": "containers",
      "env": "production"
    }
  }
]
```

Добавь в `prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'dynamic-targets'
    file_sd_configs:
      - files:
        - '/etc/prometheus/file_sd.json'
        refresh_interval: 30s
```

**Настрой Pushgateway** для метрик batch jobs:
```bash
docker run -d \
  --name pushgateway \
  -p 9091:9091 \
  prom/pushgateway

# Push метрику
echo "backup_duration_seconds 125.5" | curl --data-binary @- http://localhost:9091/metrics/job/backup/instance/db1

# Добавь в prometheus.yml
scrape_configs:
  - job_name: 'pushgateway'
    static_configs:
      - targets: ['pushgateway:9091']
    honor_labels: true
```

**Recording rules для производительности**:
```yaml
# recording_rules.yml
groups:
  - name: performance_rules
    interval: 30s
    rules:
    # CPU utilization per instance
    - record: instance:node_cpu_utilization:rate5m
      expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    
    # Memory utilization per instance
    - record: instance:node_memory_utilization:ratio
      expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
    
    # Request rate per job
    - record: job:http_requests:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job)
```

---

## Модуль 3: Grafana - визуализация данных (30 минут)

### 🎯 Напоминалка

**Архитектура Grafana:**
```
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Data     │◄─────│ Grafana  │◄─────│ Users    │
│ Sources  │      │ Server   │      │          │
└──────────┘      └──────────┘      └──────────┘
   │                    │
   │                    │
   ▼                    ▼
Prometheus       Dashboards
InfluxDB         Alerts
Elasticsearch    Users
Loki             Teams
```

**Типы панелей:**
```
Graph        - Временные ряды
Stat         - Одно значение
Gauge        - Шкала
Bar Gauge    - Горизонтальные полоски
Table        - Таблица
Heatmap      - Тепловая карта
Logs         - Логи
```

**Переменные дашборда:**
```
Query      - Из данных (label_values(metric, label))
Custom     - Список значений
Constant   - Константа
Interval   - Временной интервал
Data source - Выбор источника данных
```

**Полезные функции Grafana:**
```
$__interval        - Динамический интервал
$__rate_interval   - Рекомендуемый интервал для rate()
$timeFilter        - Временной фильтр
$__from / $__to    - Начало/конец периода

# Пример с переменной
rate(http_requests_total{job="$job"}[$__rate_interval])
```

**Templating examples:**
```promql
# Переменная instance
label_values(node_cpu_seconds_total, instance)

# Переменная job
label_values(up, job)

# Переменная mountpoint
label_values(node_filesystem_size_bytes, mountpoint)

# Использование в запросе
node_filesystem_avail_bytes{instance="$instance", mountpoint="$mountpoint"}
```

**Alert channels:**
```
Email
Slack
PagerDuty
Webhook
Telegram
Discord
Teams
OpsGenie
```

**Dashboard best practices:**
```
1. Используй Row для группировки панелей
2. Добавляй описания к панелям
3. Используй переменные для гибкости
4. Указывай единицы измерения
5. Используй цветовые пороги
6. Добавляй ссылки на runbook'и
7. Группируй связанные метрики
8. Используй consistent naming
```

### 💻 Задание

Настрой Grafana и создай dashboard:

1. **Добавь Grafana в docker-compose.yml**:
```yaml
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped
    depends_on:
      - prometheus

volumes:
  grafana-data:
```

2. **Создай provisioning для автоматической настройки** (`grafana/provisioning/datasources/prometheus.yml`):
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

3. **Создай provisioning для dashboard** (`grafana/provisioning/dashboards/dashboard.yml`):
```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

4. **Запусти Grafana**:
```bash
# Создай директории
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards

docker-compose up -d grafana

# Открой в браузере
http://localhost:3000
# Login: admin
# Password: admin
```

5. **Создай System Monitoring Dashboard** вручную:

**Panel 1: CPU Usage**
- Visualization: Time series
- Query: `100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
- Legend: CPU Usage %
- Unit: Percent (0-100)
- Threshold: Yellow at 70, Red at 90

**Panel 2: Memory Usage**
- Visualization: Time series
- Query: `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100`
- Legend: Memory Usage %
- Unit: Percent (0-100)

**Panel 3: Disk Usage**
- Visualization: Gauge
- Query: `100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)`
- Unit: Percent (0-100)
- Threshold: Green 0-70, Yellow 70-85, Red 85-100

**Panel 4: Network Traffic**
- Visualization: Time series
- Query A: `rate(node_network_receive_bytes_total[5m]) / 1024 / 1024`
- Query B: `rate(node_network_transmit_bytes_total[5m]) / 1024 / 1024`
- Unit: MB/s

**Panel 5: Top Processes by CPU**
- Visualization: Table
- Query: `topk(5, irate(process_cpu_seconds_total[5m]))`

6. **Создай переменные для dashboard**:
- Variable: instance
  - Type: Query
  - Query: `label_values(node_cpu_seconds_total, instance)`
  
Измени запросы на использование переменной:
```promql
100 - (avg(irate(node_cpu_seconds_total{instance="$instance", mode="idle"}[5m])) * 100)
```

### 🚀 Бонус (новое)

**Создай JSON dashboard через provisioning** (`grafana/provisioning/dashboards/system-overview.json`):
```json
{
  "dashboard": {
    "title": "System Overview",
    "tags": ["system", "monitoring"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "type": "timeseries",
        "title": "CPU Usage",
        "targets": [
          {
            "expr": "100 - (avg(irate(node_cpu_seconds_total{instance=\"$instance\",mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 70, "color": "yellow"},
                {"value": 90, "color": "red"}
              ]
            }
          }
        }
      }
    ],
    "templating": {
      "list": [
        {
          "name": "instance",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(node_cpu_seconds_total, instance)",
          "refresh": 1
        }
      ]
    }
  }
}
```

**Настрой Alerting в Grafana**:
1. Configuration → Alerting → Contact points
2. Создай Email contact point
3. Создай Alert rule:
   - Name: High CPU Alert
   - Query: `avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100 < 20`
   - Condition: WHEN last() OF query(A) IS BELOW 20
   - For: 5m

**Установи Grafana plugins**:
```bash
# Установка через UI
Configuration → Plugins → Search

# Полезные плагины:
- Pie Chart
- Worldmap Panel
- Clock Panel
- Status Panel

# Через CLI (в контейнере)
docker exec grafana grafana-cli plugins install grafana-piechart-panel
docker restart grafana
```

---

## Модуль 4: Логирование и централизация логов (30 минут)

### 🎯 Напоминалка

**Уровни логирования:**
```
TRACE   - Детальная информация для отладки
DEBUG   - Отладочная информация
INFO    - Информационные сообщения
WARN    - Предупреждения
ERROR   - Ошибки, не критичные для работы
FATAL   - Критические ошибки, приложение падает
```

**Structured logging (JSON):**
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "api",
  "message": "Database connection failed",
  "error": "connection timeout",
  "user_id": "12345",
  "request_id": "abc-123",
  "duration_ms": 5000
}
```

**ELK Stack:**
```
Elasticsearch  - Хранение и поиск
Logstash       - Обработка и парсинг
Kibana         - Визуализация
```

**Alternative: Loki Stack:**
```
Loki           - Хранение логов
Promtail       - Агент сбора
Grafana        - Визуализация
```

**Log aggregation patterns:**
```
┌──────────┐
│   App    │────┐
└──────────┘    │
                │    ┌─────────┐    ┌──────────────┐
┌──────────┐    ├───►│ Log     │───►│ Centralized  │
│   App    │────┤    │ Shipper │    │ Log Storage  │
└──────────┘    │    └─────────┘    └──────────────┘
                │
┌──────────┐    │
│   App    │────┘
└──────────┘
```

**Полезные команды для логов:**
```bash
# journalctl (systemd)
journalctl -u nginx                  # Логи сервиса
journalctl -f                        # Follow логи
journalctl --since "1 hour ago"
journalctl -p err                    # Только ошибки
journalctl --disk-usage              # Размер логов

# Docker logs
docker logs <container>
docker logs -f <container>
docker logs --tail 100 <container>
docker logs --since 1h <container>

# Традиционные инструменты:
# tail
tail -f /var/log/syslog
tail -n 100 /var/log/nginx/access.log
tail -f /var/log/app.log | grep ERROR

# grep
grep "ERROR" /var/log/app.log
grep -r "connection timeout" /var/log/
grep -C 5 "OutOfMemory" app.log  # 5 строк контекста

# awk
awk '{print $1, $4, $7}' /var/log/nginx/access.log
awk '/ERROR/ {count++} END {print count}' app.log

# sed
sed -n '/ERROR/,/END/p' app.log

# less/more
less +F /var/log/syslog  # Follow mode
```

### 💻 Задание

Настрой Loki Stack для сбора логов:

1. **Добавь Loki в docker-compose.yml** (из предыдущего модуля):

yaml

```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped
    depends_on:
      - loki

volumes:
  loki-data:
```

2. **Создай loki-config.yml**:

yaml

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

# Retention (опционально)
limits_config:
  retention_period: 744h  # 31 день
```

3. **Создай promtail-config.yml**:

yaml

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Системные логи
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: my-server
          __path__: /var/log/*.log

  # Docker контейнеры
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log

  # Nginx access logs
  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          type: access
          __path__: /var/log/nginx/access.log

  # Nginx error logs
  - job_name: nginx-error
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          type: error
          __path__: /var/log/nginx/error.log

  # Application logs с парсингом JSON
  - job_name: app-json
    static_configs:
      - targets:
          - localhost
        labels:
          job: app
          format: json
          __path__: /var/log/app/*.json
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp
      - labels:
          level:
      - timestamp:
          source: timestamp
          format: RFC3339
```

4. **Запусти Loki Stack**:

bash

```bash
docker-compose up -d loki promtail

# Проверка
curl http://localhost:3100/ready
curl http://localhost:3100/metrics
docker logs promtail
```

5. **Добавь Loki в Grafana**:

- Открой Grafana ([http://localhost:3000](http://localhost:3000))
- Configuration → Data Sources → Add data source
- Выбери Loki
- URL: `http://loki:3100`
- Save & Test

6. **Используй LogQL для запросов**:

logql

```logql
# Все логи контейнера
{container="prometheus"}

# Фильтр по уровню
{job="app"} |= "ERROR"

# Regex фильтр
{job="nginx"} |~ "status=5.."

# Несколько меток
{job="app", level="error"}

# Исключение
{job="app"} != "debug"

# Count по времени
count_over_time({job="app"}[5m])

# Rate errors
rate({job="app"} |= "ERROR" [5m])

# Агрегация
sum(rate({job="app"} |= "ERROR" [5m])) by (level)

# Top N источников ошибок
topk(5, sum(rate({job="app"} |= "ERROR" [5m])) by (host))
```

7. **Создай Dashboard для логов**:

- New Dashboard → Add Panel
- Data source: Loki
- Query: `{job="app"} |= "ERROR"`
- Visualization: Logs
- Добавь фильтры по времени и severity

### 🚀 Бонус (новое)

**Настрой структурированное логирование в приложении**:

**Python example (structlog)**:

python

```python
# structured_logging.py
import structlog
import logging

# Настройка structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

log = structlog.get_logger()

# Использование
log.info("user_login", user_id=123, ip="192.168.1.1")
log.error("database_error", 
          error="connection timeout", 
          db="postgres", 
          retry_count=3)

# С контекстом
log = log.bind(request_id="abc-123")
log.info("processing_request", endpoint="/api/users")
log.debug("query_executed", duration_ms=45.2)
```

**Node.js example (winston)**:

javascript

```javascript
// structured-logging.js
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'user-service' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    })
  ]
});

// Использование
logger.info('User login', { 
  userId: 123, 
  ip: '192.168.1.1',
  userAgent: 'Mozilla/5.0'
});

logger.error('Database error', {
  error: 'Connection timeout',
  database: 'postgres',
  retryCount: 3
});
```

**Настрой алерты в Loki**:

Создай `loki-rules.yml`:

yaml

```yaml
groups:
  - name: logs
    interval: 1m
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({job="app"} |= "ERROR" [5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"

      - alert: DatabaseConnectionErrors
        expr: |
          count_over_time({job="app"} |= "database connection" |= "ERROR" [5m]) > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Database connection issues"
          description: "{{ $value }} database errors in 5 minutes"

      - alert: ApplicationCrash
        expr: |
          count_over_time({job="app"} |= "FATAL" [1m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Application crash detected"
          description: "FATAL error occurred"
```

**Используй log sampling для высоконагруженных приложений**:

python

````python
import random
import structlog

log = structlog.get_logger()

def log_with_sampling(level, message, sample_rate=0.1, **kwargs):
    """Логирует только sample_rate процентов сообщений"""
    if random.random() < sample_rate:
        getattr(log, level)(message, **kwargs)

# Логируем только 10% debug сообщений
log_with_sampling('debug', 'request_processed', 
                  sample_rate=0.1,
                  endpoint='/api/users',
                  duration_ms=12.3)

# Всегда логируем errors
log.error('critical_error', error='something bad happened')

````

---

## Модуль 5: Alerting и уведомления (30 минут)

### 🎯 Напоминалка

**Alertmanager архитектура:**
```
┌─────────────┐
│ Prometheus  │ ──alerts──> ┌──────────────┐
│  (metrics)  │             │ Alertmanager │
└─────────────┘             │              │
                            │  - Grouping  │
┌─────────────┐             │  - Routing   │
│    Loki     │ ──alerts──> │  - Silencing │
│   (logs)    │             │  - Inhibition│
└─────────────┘             └──────┬───────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
               ┌────────┐    ┌─────────┐   ┌──────────┐
               │ Email  │    │  Slack  │   │PagerDuty │
               └────────┘    └─────────┘   └──────────┘
```

**Alert States:**
```
Inactive  - Условие не выполнено
Pending   - Условие выполнено, но for: не прошел
Firing    - Условие выполнено и for: прошел
````

**Хорошие практики алертов:**

**1. Severity Levels:**

yaml

```yaml
critical  - Требует немедленного действия (пейджер 24/7)
warning   - Требует внимания в рабочее время
info      - Информационный, может быть в dashboard
```

**2. Alert Design:**

yaml

```yaml
# ПЛОХО: Too broad
- alert: HighCPU
  expr: node_cpu > 50

# ХОРОШО: Specific and actionable
- alert: CriticalCPUUsage
  expr: |
    100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
  for: 10m
  labels:
    severity: critical
    component: infrastructure
  annotations:
    summary: "Critical CPU usage on {{ $labels.instance }}"
    description: "CPU usage is {{ $value }}% for more than 10 minutes"
    runbook: "https://wiki.company.com/runbooks/high-cpu"
    dashboard: "https://grafana.company.com/d/node-dashboard"
```

**3. Annotation Best Practices:**

yaml

```yaml
annotations:
  summary: "Краткое описание проблемы"
  description: "Детали с переменными: {{ $value }}, {{ $labels.instance }}"
  runbook: "Ссылка на runbook с решением"
  dashboard: "Ссылка на dashboard для анализа"
  impact: "Что сломано для пользователей"
  action: "Что нужно сделать немедленно"
```

**Alertmanager конфигурация:**

yaml

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@company.com'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'password'

# Шаблоны для группировки
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s        # Ждать другие алерты перед отправкой
  group_interval: 10s    # Интервал между группами
  repeat_interval: 12h   # Повторять каждые 12ч если не resolved
  
  routes:
    # Critical алерты - PagerDuty 24/7
    - match:
        severity: critical
      receiver: pagerduty
      group_wait: 10s
      repeat_interval: 5m
    
    # Warning - Slack в рабочее время
    - match:
        severity: warning
      receiver: slack-warnings
      group_wait: 5m
      repeat_interval: 4h
    
    # Database алерты - отдельный канал
    - match_re:
        service: database|postgres|mysql
      receiver: database-team
    
    # Development алерты - отдельный получатель
    - match:
        environment: dev
      receiver: dev-team

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@company.com'
        headers:
          Subject: '{{ .GroupLabels.alertname }}'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts-warnings'
        title: '{{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
            *Runbook:* {{ .Annotations.runbook }}
          {{ end }}
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .GroupLabels.alertname }}'

  - name: 'database-team'
    email_configs:
      - to: 'dba@company.com'
    slack_configs:
      - channel: '#database-alerts'

inhibit_rules:
  # Если нода недоступна, не шли алерты о сервисах на ней
  - source_match:
      severity: 'critical'
      alertname: 'InstanceDown'
    target_match:
      severity: 'warning'
    equal: ['instance']
  
  # Если весь кластер недоступен, не шли алерты о нодах
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: 'Instance.*|Node.*'
    equal: ['cluster']
```

**Примеры правил алертов:**

**Infrastructure Alerts:**

yaml

```yaml
groups:
  - name: infrastructure
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} has been down for more than 5 minutes"
          runbook: "https://wiki.company.com/runbooks/instance-down"

      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}%"

      - alert: DiskWillFillIn4Hours
        expr: |
          predict_linear(node_filesystem_free_bytes{mountpoint="/"}[1h], 4*3600) < 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk will fill in 4 hours on {{ $labels.instance }}"
          description: "Based on current trend, disk will be full in ~4 hours"

      - alert: HighNetworkErrors
        expr: |
          rate(node_network_receive_errs_total[5m]) > 100
          or
          rate(node_network_transmit_errs_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High network errors on {{ $labels.instance }}"
          description: "{{ $value }} errors per second on {{ $labels.device }}"
```

**Application Alerts:**

yaml

```yaml
groups:
  - name: application
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
          /
          rate(http_requests_total[5m])
          * 100 > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanize }}%"
          impact: "Users experiencing failures"
          action: "Check application logs and recent deployments"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.service }}"
          description: "95th percentile is {{ $value }}s"

      - alert: LowThroughput
        expr: |
          rate(http_requests_total[5m]) < 10
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Low throughput on {{ $labels.service }}"
          description: "Only {{ $value }} req/s (normally >100)"
          impact: "Possible service degradation"

      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} crash looping"
          description: "Pod has restarted {{ $value }} times in 15 minutes"
```

**Database Alerts:**

yaml

```yaml
groups:
  - name: database
    interval: 30s
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"
          description: "Database {{ $labels.instance }} is unreachable"
          impact: "Application cannot access database"
          action: "Check database logs and restart if needed"

      - alert: PostgreSQLTooManyConnections
        expr: |
          sum(pg_stat_activity_count) by (instance) 
          / 
          pg_settings_max_connections 
          * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL too many connections"
          description: "{{ $value | humanize }}% connections used on {{ $labels.instance }}"

      - alert: PostgreSQLSlowQueries
        expr: |
          rate(pg_stat_activity_max_tx_duration{state="active"}[5m]) > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL slow queries detected"
          description: "Queries running longer than 60s on {{ $labels.instance }}"

      - alert: PostgreSQLReplicationLag
        expr: |
          pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag"
          description: "Replication lag is {{ $value }}s on {{ $labels.instance }}"
```

**Business Metrics Alerts:**

yaml

```yaml
groups:
  - name: business
    interval: 1m
    rules:
      - alert: LowSignupRate
        expr: |
          rate(user_signups_total[1h]) < 5
        for: 30m
        labels:
          severity: warning
          team: growth
        annotations:
          summary: "Low signup rate detected"
          description: "Only {{ $value }} signups/hour (normally >20)"
          impact: "Revenue impact, possible funnel issue"

      - alert: HighCheckoutAbandonment
        expr: |
          (
            rate(checkout_started_total[1h])
            -
            rate(checkout_completed_total[1h])
          )
          /
          rate(checkout_started_total[1h])
          * 100 > 70
        for: 15m
        labels:
          severity: warning
          team: payments
        annotations:
          summary: "High checkout abandonment rate"
          description: "{{ $value | humanize }}% abandonment (normally <50%)"

      - alert: PaymentGatewayDown
        expr: |
          rate(payment_transactions_total{status="failed"}[5m])
          /
          rate(payment_transactions_total[5m])
          * 100 > 10
        for: 2m
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "Payment gateway issues"
          description: "{{ $value }}% failure rate"
          impact: "Customers cannot complete purchases"
          action: "Check payment provider status immediately"
```

### 💻 Задание

Настрой полноценную систему алертинга:

1. **Добавь Alertmanager в docker-compose.yml**:

yaml

```yaml
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped

volumes:
  alertmanager-data:
```

2. **Создай alertmanager.yml**:

yaml

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'email-default'
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    - match:
        severity: critical
      receiver: critical-alerts
      group_wait: 10s
      repeat_interval: 10m
    
    - match:
        severity: warning
      receiver: warning-alerts
      repeat_interval: 6h

receivers:
  - name: 'email-default'
    email_configs:
      - to: 'your-email@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'your-email@example.com'
        auth_password: 'your-app-password'
        headers:
          Subject: '[ALERT] {{ .GroupLabels.alertname }}'
        html: |
          <h2>Alert: {{ .GroupLabels.alertname }}</h2>
          {{ range .Alerts }}
          <p><b>Summary:</b> {{ .Annotations.summary }}</p>
          <p><b>Description:</b> {{ .Annotations.description }}</p>
          <p><b>Severity:</b> {{ .Labels.severity }}</p>
          <hr>
          {{ end }}

  - name: 'critical-alerts'
    email_configs:
      - to: 'oncall@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'your-email@example.com'
        auth_password: 'your-app-password'
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }}'

  - name: 'warning-alerts'
    email_configs:
      - to: 'team@example.com'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance']
```

3. **Обнови prometheus.yml** для использования Alertmanager:

yaml

```yaml
# Добавь в существующий prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'

rule_files:
  - "alerts.yml"
  - "recording_rules.yml"
```

4. **Создай комплексный набор алертов** (обнови `alerts.yml`):

yaml

```yaml
groups:
  - name: critical_alerts
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }} on {{ $labels.instance }} has been down for more than 5 minutes"
          impact: "Service unavailable"
          action: "Check instance logs and restart if needed"

      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 15m
        labels:
          severity: critical
        annotations:
          summary: "Critical CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% for more than 15 minutes"
          impact: "Performance degradation"
          action: "Investigate running processes and scale if needed"

      - alert: OutOfMemory
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Out of memory on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}%"
          impact: "Risk of OOM killer"
          action: "Free memory or scale vertically"

      - alert: DiskFull
        expr: |
          (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk almost full on {{ $labels.instance }}"
          description: "Disk usage is {{ $value | humanize }}%"
          impact: "Application may fail"
          action: "Clean up disk space immediately"

  - name: warning_alerts
    interval: 1m
    rules:
      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}%"

      - alert: HighDiskUsage
        expr: |
          (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 75
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High disk usage on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} is {{ $value | humanize }}% full"

      - alert: HighNetworkTraffic
        expr: |
          rate(node_network_receive_bytes_total[5m]) > 100000000  # 100MB/s
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High network traffic on {{ $labels.instance }}"
          description: "Receiving {{ $value | humanize }}B/s on {{ $labels.device }}"

  - name: prometheus_alerts
    interval: 30s
    rules:
      - alert: PrometheusConfigReloadFailed
        expr: prometheus_config_last_reload_successful == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Prometheus config reload failed"
          description: "Prometheus configuration reload has failed"
          action: "Check Prometheus logs and fix configuration"

      - alert: PrometheusTooManyRestarts
        expr: |
          changes(process_start_time_seconds{job="prometheus"}[15m]) > 2
        labels:
          severity: warning
        annotations:
          summary: "Prometheus restarted too many times"
          description: "Prometheus has restar {{ $value }} times in 15 minutes"

````

5. **Запусти и протестируй**:
```bash
# Перезапусти Prometheus с новой конфигурацией
docker-compose up -d prometheus alertmanager

# Проверь статус алертов
curl http://localhost:9090/api/v1/alerts | jq

# Проверь Alertmanager
curl http://localhost:9093/api/v2/status | jq

# Открой UI
# Prometheus: http://localhost:9090/alerts
# Alertmanager: http://localhost:9093
```

6. **Создай тестовый алерт**:
```yaml
# test-alert.yml
groups:
  - name: test
    rules:
      - alert: AlwaysFiring
        expr: vector(1)
        labels:
          severity: warning
        annotations:
          summary: "Test alert - always firing"
          description: "This is a test alert"
```

Добавь в prometheus.yml и перезагрузи конфигурацию:
```bash
docker exec prometheus kill -HUP 1
```

### 🚀 Бонус (новое)

**1. Настрой Slack интеграцию**:

Создай Slack webhook: https://api.slack.com/messaging/webhooks

Обнови `alertmanager.yml`:
```yaml
receivers:
  - name: 'slack-critical'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts-critical'
        title: ':fire: {{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
            *Summary:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
            *Instance:* {{ .Labels.instance }}
            {{ if .Annotations.runbook }}*Runbook:* {{ .Annotations.runbook }}{{ end }}
            {{ if .Annotations.dashboard }}*Dashboard:* {{ .Annotations.dashboard }}{{ end }}
          {{ end }}
        send_resolved: true
        actions:
          - type: button
            text: 'Runbook :green_book:'
            url: '{{ .Annotations.runbook }}'
          - type: button
            text: 'Query :mag:'
            url: '{{ .GeneratorURL }}'
          - type: button
            text: 'Silence :no_bell:'
            url: '{{ .ExternalURL }}/#/silences/new?filter=%7B'
```

**2. Создай custom шаблоны для уведомлений**:

Создай `alertmanager-templates.tmpl`:
```go
{{ define "slack.default.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.default.text" }}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
*Description:* {{ .Annotations.description }}
*Details:*
  {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
  {{ end }}
{{ end }}
{{ end }}

{{ define "email.default.subject" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }} on {{ .GroupLabels.instance }}
{{ end }}

{{ define "email.default.html" }}
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .alert { padding: 15px; margin: 10px 0; border-radius: 5px; }
        .critical { background-color: #ff4444; color: white; }
        .warning { background-color: #ffaa00; color: white; }
        .info { background-color: #4444ff; color: white; }
        .resolved { background-color: #44ff44; color: white; }
    </style>
</head>
<body>
    <h2>Alert Notification</h2>
    {{ range .Alerts }}
    <div class="alert {{ .Labels.severity }}">
        <h3>{{ .Annotations.summary }}</h3>
        <p><strong>Description:</strong> {{ .Annotations.description }}</p>
        <p><strong>Severity:</strong> {{ .Labels.severity }}</p>
        <p><strong>Instance:</strong> {{ .Labels.instance }}</p>
        <p><strong>Started:</strong> {{ .StartsAt }}</p>
        {{ if .Annotations.runbook }}
        <p><a href="{{ .Annotations.runbook }}" style="color: white;">View Runbook</a></p>
        {{ end }}
    </div>
    {{ end }}
</body>
</html>
{{ end }}
```

Обнови `alertmanager.yml`:
```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@company.com'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'password'

templates:
  - '/etc/alertmanager/alertmanager-templates.tmpl'

receivers:
  - name: 'email-with-template'
    email_configs:
      - to: 'team@example.com'
        headers:
          Subject: '{{ template "email.default.subject" . }}'
        html: '{{ template "email.default.html" . }}'
```

**3. Настрой Silences через API**:
```bash
# Создать silence на 1 час
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "InstanceDown",
        "isRegex": false
      }
    ],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "endsAt": "'$(date -u -d '+1 hour' +%Y-%m-%dT%H:%M:%SZ)'",
    "createdBy": "admin",
    "comment": "Maintenance window"
  }'

# Список активных silences
curl http://localhost:9093/api/v2/silences | jq

# Удалить silence
curl -X DELETE http://localhost:9093/api/v2/silence/<silence-id>
```

**4. Настрой routing по времени (business hours)**:
```yaml
route:
  receiver: 'default'
  routes:
    # Критичные алерты всегда через PagerDuty
    - match:
        severity: critical
      receiver: pagerduty
      continue: true  # Продолжить обработку для других receivers
    
    # Warning только в рабочее время на email
    - match:
        severity: warning
      receiver: email-business-hours
      active_time_intervals:
        - business_hours
    
    # Warning вне рабочего времени только в Slack (менее шумно)
    - match:
        severity: warning
      receiver: slack-after-hours

time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '18:00'
        weekdays: ['monday:friday']
        months: ['january:december']
        days_of_month: ['1:31']
```

**5. Настрой Dead Man's Switch** (алерт который должен всегда срабатывать):
```yaml
# В alerts.yml
groups:
  - name: meta
    interval: 30s
    rules:
      - alert: DeadMansSwitch
        expr: vector(1)
        labels:
          severity: info
        annotations:
          summary: "Alerting pipeline is healthy"
          description: "This alert should always be firing. If it stops, check Prometheus and Alertmanager"
```

Если этот алерт перестанет приходить - значит что-то сломалось в мониторинге!

---
## Модуль 6: Распределенный трейсинг (30 минут)

### 🎯 Напоминалка

**Что такое трейсинг:**
```

Request → Service A → Service B → Service C → Database
   |          |          |          |          |
 Span 1     Span 2     Span 3     Span 4     Span 5

- Trace = коллекция всех Spans для одного request 
- Span = единица работы (вызов функции, HTTP request, DB query)

```

**OpenTelemetry архитектура:**
```
┌──────────────┐
│ Application  │
│  + OTel SDK  │
└──────┬───────┘
       │ (traces, metrics, logs)
       ▼
┌──────────────┐
│ OTel         │
│ Collector    │ ← агрегация, фильтрация
└──────┬───────┘
       │
   ┌───┴────┬────────┐
   ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐
│Jaeger│ │Tempo │ │Zipkin│
└──────┘ └──────┘ └──────┘

```

**Типы spans:**
```

CLIENT - HTTP client call, DB query SERVER - HTTP server receiving request  
PRODUCER - Отправка сообщения в queue CONSUMER - Получение сообщения из queue INTERNAL - Внутренняя операция

````

**Span attributes (важные теги):**
```yaml
# HTTP
http.method: GET
http.url: https://api.example.com/users
http.status_code: 200
http.route: /users/:id

# Database
db.system: postgresql
db.name: users_db
db.statement: SELECT * FROM users WHERE id = ?
db.operation: SELECT

# RPC
rpc.system: grpc
rpc.service: UserService
rpc.method: GetUser

# Messaging
messaging.system: rabbitmq
messaging.destination: user.events
messaging.operation: send
```

**Sampling strategies:**
```yaml
Always On    - 100% (dev/staging)
Always Off   - 0%
Probabilistic - N% (например 10%)
Rate Limiting - X traces/second
Tail-based   - Семплируй на основе результата (ошибки = 100%)
```

### 💻 Задание

Настрой Jaeger для распределенного трейсинга:

1. **Добавь Jaeger в docker-compose.yml**:
```yaml
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "5775:5775/udp"     # zipkin.thrift
      - "6831:6831/udp"     # jaeger.thrift compact
      - "6832:6832/udp"     # jaeger.thrift binary
      - "5778:5778"         # serve configs
      - "16686:16686"       # UI
      - "14268:14268"       # jaeger.thrift
      - "14250:14250"       # model.proto
      - "9411:9411"         # Zipkin compatible
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      - COLLECTOR_OTLP_ENABLED=true
    restart: unless-stopped
```

2. **Создай тестовое приложение с трейсингом** (Python):
```python
# app_with_tracing.py
from flask import Flask, request
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
import requests
import time
import random

# Настройка трейсинга
resource = Resource.create({"service.name": "frontend-service"})
trace.set_tracer_provider(TracerProvider(resource=resource))
tracer = trace.get_tracer(__name__)

# Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)

trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

app = Flask(__name__)

# Автоматическая инструментация
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route('/')
def index():
    with tracer.start_as_current_span("index-handler"):
        # Симуляция работы
        time.sleep(random.uniform(0.01, 0.05))
        return "Hello from Frontend!"

@app.route('/api/users')
def get_users():
    with tracer.start_as_current_span("get-users"):
        # Добавляем атрибуты к span
        span = trace.get_current_span()
        span.set_attribute("http.route", "/api/users")
        span.set_attribute("user.count", 10)
        
        # Вызов базы данных (симуляция)
        users = fetch_users_from_db()
        
        # Вызов другого сервиса
        try:
            enriched_users = enrich_user_data(users)
            return {"users": enriched_users}
        except Exception as e:
            span.record_exception(e)
            span.set_attribute("error", True)
            raise

def fetch_users_from_db():
    with tracer.start_as_current_span("db-query") as span:
        span.set_attribute("db.system", "postgresql")
        span.set_attribute("db.statement", "SELECT * FROM users")
        
        # Симуляция DB query
        time.sleep(random.uniform(0.05, 0.15))
        return [
            {"id": 1, "name": "Alice"},
            {"id": 2, "name": "Bob"}
        ]

def enrich_user_data(users):
    with tracer.start_as_current_span("enrich-users") as span:
        # Вызов другого микросервиса
        try:
            response = requests.get(
                "http://backend:8080/api/user-details",
                timeout=2
            )
            response.raise_for_status()
            
            span.set_attribute("http.status_code", response.status_code)
            return response.json()
            
        except requests.exceptions.Timeout:
            span.set_attribute("error", True)
            span.set_attribute("error.type", "timeout")
            # Fallback
            return users
        except Exception as e:
            span.record_exception(e)
            raise

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

3. **Создай backend сервис**:
```python
# backend_service.py
from flask import Flask
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.flask import FlaskInstrumentor
import time
import random

resource = Resource.create({"service.name": "backend-service"})
trace.set_tracer_provider(TracerProvider(resource=resource))
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)

trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

@app.route('/api/user-details')
def user_details():
    with tracer.start_as_current_span("fetch-user-details"):
        # Симуляция обработки
        time.sleep(random.uniform(0.1, 0.3))
        
        # Случайная ошибка для демонстрации
        if random.random() < 0.1:
            span = trace.get_current_span()
            span.set_attribute("error", True)
            return {"error": "Service temporarily unavailable"}, 503
        
        return {
            "users": [
                {"id": 1, "name": "Alice", "role": "Admin"},
                {"id": 2, "name": "Bob", "role": "User"}
            ]
        }

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

4. **Dockerfile для приложений**:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install \
    flask \
    requests \
    opentelemetry-api \
    opentelemetry-sdk \
    opentelemetry-instrumentation-flask \
    opentelemetry-instrumentation-requests \
    opentelemetry-exporter-jaeger-thrift

COPY app_with_tracing.py .

CMD ["python", "app_with_tracing.py"]
```

5. **Добавь сервисы в docker-compose.yml**:
```yaml
  frontend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: frontend
    ports:
      - "5000:5000"
    environment:
      - JAEGER_AGENT_HOST=jaeger
    depends_on:
      - jaeger
      - backend

  backend:
    build:
      context: .
      dockerfile: Dockerfile.backend
    container_name: backend
    ports:
      - "8080:8080"
    environment:
      - JAEGER_AGENT_HOST=jaeger
    command: python backend_service.py
    depends_on:
      - jaeger
```

6. **Запусти и тестируй**:
```bash
# Запусти все сервисы
docker-compose up -d jaeger frontend backend

# Сгенерируй трафик
for i in {1..20}; do
  curl http://localhost:5000/api/users
  sleep 1
done

# Открой Jaeger UI
http://localhost:16686

# Выбери service: frontend-service
# Посмотри на трейсы
```

7. **Анализ трейсов в Jaeger**:
- Service: выбери `frontend-service`
- Operation: выбери `get-users`
- Посмотри на водопад (waterfall) вызовов
- Найди медленные spans
- Найди spans с ошибками
- Сравни разные трейсы

### 🚀 Бонус (новое)

**1. Настрой контекстное логирование с trace ID**:
```python
from opentelemetry import trace
import logging
import json

class TraceContextLogFilter(logging.Filter):
    """Добавляет trace_id и span_id в логи"""
    def filter(self, record):
        span = trace.get_current_span()
        if span != trace.INVALID_SPAN:
            ctx = span.get_span_context()
            record.trace_id = format(ctx.trace_id, '032x')
            record.span_id = format(ctx.span_id, '016x')
        else:
            record.trace_id = ''
            record.span_id = ''
        return True

# Настройка логгера
logging.basicConfig(
    format='{"time": "%(asctime)s", "level": "%(levelname)s", "trace_id": "%(trace_id)s", "span_id": "%(span_id)s", "message": "%(message)s"}',
    level=logging.INFO
)

logger = logging.getLogger(__name__)
logger.addFilter(TraceContextLogFilter())

# Теперь в логах будет trace_id!
@app.route('/api/users')
def get_users():
    logger.info("Fetching users")  # Будет с trace_id
    # ...
```

**2. Используй tail-based sampling** для оптимизации:
```python
from opentelemetry.sdk.trace.sampling import (
    TraceIdRatioBased,
    ParentBasedTraceIdRatio,
)

# Семплируй 10% успешных и 100% ошибочных трейсов
sampler = ParentBasedTraceIdRatio(0.1)  # 10% by default

trace_provider = TracerProvider(
    resource=resource,
    sampler=sampler
)
```

**3. Добавь custom spans для бизнес-логики**:
```python
@app.route('/checkout')
def checkout():
    with tracer.start_as_current_span("checkout-flow") as span:
        # Шаг 1: Валидация корзины
        with tracer.start_as_current_span("validate-cart") as cart_span:
            cart = validate_cart()
            cart_span.set_attribute("cart.items", len(cart))
            cart_span.set_attribute("cart.total", cart['total'])
        
        # Шаг 2: Проверка инвентаря
        with tracer.start_as_current_span("check-inventory") as inv_span:
            available = check_inventory(cart)
            inv_span.set_attribute("inventory.available", available)
            if not available:
                inv_span.set_attribute("error", True)
                return {"error": "Items out of stock"}, 400
        
        # Шаг 3: Обработка платежа
        with tracer.start_as_current_span("process-payment") as pay_span:
            pay_span.set_attribute("payment.method", "credit_card")
            try:
                payment = process_payment(cart)
                pay_span.set_attribute("payment.success", True)
                pay_span.set_attribute("payment.id", payment['id'])
            except PaymentError as e:
                pay_span.record_exception(e)
                pay_span.set_attribute("error", True)
                return {"error": "Payment failed"}, 400
        
        span.set_attribute("checkout.success", True)
        return {"order_id": create_order(cart, payment)}
```

**4. Интеграция с Grafana Tempo** (более легковесная альтернатива Jaeger):
```yaml
  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    command: [ "-config.file=/etc/tempo.yaml" ]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
      - tempo-data:/tmp/tempo
    ports:
      - "14268:14268"  # jaeger ingest
      - "3200:3200"    # tempo query
      - "4317:4317"    # otlp grpc
      - "4318:4318"    # otlp http

volumes:
  tempo-data:
```

`tempo.yaml`:
```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    jaeger:
      protocols:
        thrift_http:
          endpoint: 0.0.0.0:14268
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/blocks
```

---
## Модуль 7: Application Performance Monitoring (APM) (30 минут)

### 🎯 Напоминалка

**APM - что мониторим:**

```
┌─────────────────────────────────────┐
│ User Experience                     │
├─────────────────────────────────────┤
│ • Page Load Time                    │
│ • Time to First Byte (TTFB)        │
│ • First Contentful Paint (FCP)     │
│ • Largest Contentful Paint (LCP)   │
│ • Cumulative Layout Shift (CLS)    │
└─────────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ Application Layer                   │
├─────────────────────────────────────┤
│ • Request Rate                      │
│ • Error Rate                        │
│ • Response Time (p50, p95, p99)    │
│ • Throughput                        │
│ • Apdex Score                       │
└─────────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ Code Level                          │
├─────────────────────────────────────┤
│ • Function Execution Time           │
│ • Database Query Performance        │
│ • External API Calls                │
│ • Memory Allocations                │
│ • CPU Profiling                     │
└─────────────────────────────────────┘
```

**Ключевые метрики APM:**

**1. Apdex Score** (Application Performance Index):

```
Apdex = (Satisfied + Tolerating/2) / Total Requests

Satisfied:   Response time ≤ T (target)
Tolerating:  T < Response time ≤ 4T
Frustrated:  Response time > 4T

Пример: T = 0.5s
- 0.3s  → Satisfied
- 1.2s  → Tolerating
- 3.0s  → Frustrated

Score: 0.0-1.0 (1.0 = идеально)
```

**2. Percentiles:**

```
p50 (median)  - 50% requests быстрее
p95           - 95% requests быстрее (хорошо для SLA)
p99           - 99% requests быстрее (tail latency)
p99.9         - для критичных систем

Почему не среднее?
Average: [10ms, 10ms, 10ms, 5000ms] = 1257ms
p95:     [10ms, 10ms, 10ms, 5000ms] = 10ms
```

**3. Service Level Objectives (SLO):**

yaml

```yaml
SLI (Indicator):   Availability = successful_requests / total_requests
SLO (Objective):   99.9% availability
SLA (Agreement):   99.9% or credit

Error Budget:      0.1% = 43 minutes/month downtime
```

**4. Golden Signals для APM:**

yaml

````yaml
Latency:      How long to process requests
Traffic:      How many requests
Errors:       Rate of failed requests  
Saturation:   How "full" your service is
```

**Инструменты APM:**
```
Commercial:
- New Relic
- Datadog APM
- Dynatrace
- AppDynamics

Open Source:
- Elastic APM
- SigNoz
- Grafana Tempo + Prometheus
- Jaeger
````

### 💻 Задание

Настрой полноценный APM с Elastic Stack:

1. **Добавь Elastic APM в docker-compose.yml**:

yaml

```yaml
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    restart: unless-stopped

  apm-server:
    image: docker.elastic.co/apm/apm-server:8.11.0
    container_name: apm-server
    ports:
      - "8200:8200"
    command: >
      apm-server -e
        -E apm-server.rum.enabled=true
        -E apm-server.host=0.0.0.0:8200
        -E output.elasticsearch.hosts=["elasticsearch:9200"]
        -E apm-server.kibana.enabled=true
        -E apm-server.kibana.host=kibana:5601
    depends_on:
      - elasticsearch
      - kibana
    restart: unless-stopped

volumes:
  elasticsearch-data:
```

2. **Создай приложение с Elastic APM instrumentation**:

**Python/Flask example:**

python

```python
# app_with_apm.py
from flask import Flask, request, jsonify
from elasticapm.contrib.flask import ElasticAPM
import time
import random
import psycopg2
from redis import Redis

app = Flask(__name__)

# Конфигурация Elastic APM
app.config['ELASTIC_APM'] = {
    'SERVICE_NAME': 'my-flask-app',
    'SERVER_URL': 'http://apm-server:8200',
    'ENVIRONMENT': 'production',
    'CAPTURE_BODY': 'all',
    'TRANSACTION_SAMPLE_RATE': 1.0,  # 100% в dev, 0.1 в prod
}

apm = ElasticAPM(app)
redis_client = Redis(host='redis', port=6379, decode_responses=True)

@app.route('/')
def index():
    return jsonify({
        'status': 'ok',
        'service': 'my-flask-app'
    })

@app.route('/api/users')
def get_users():
    """Endpoint с различными операциями для APM"""
    
    # Custom span для кеша
    with apm.capture_span('check_cache', span_type='cache'):
        cache_key = 'users:all'
        cached = redis_client.get(cache_key)
        
        if cached:
            apm.tag(cache='hit')
            return jsonify({'users': eval(cached), 'source': 'cache'})
        
        apm.tag(cache='miss')
    
    # Database query (автоматически отслеживается)
    users = fetch_users_from_db()
    
    # External API call
    with apm.capture_span('enrich_user_data', span_type='external.http'):
        enriched = enrich_users(users)
    
    # Сохраняем в кеш
    with apm.capture_span('save_to_cache', span_type='cache'):
        redis_client.setex(cache_key, 300, str(enriched))
    
    return jsonify({'users': enriched, 'source': 'database'})

@app.route('/api/slow')
def slow_endpoint():
    """Медленный endpoint для тестирования"""
    # Симуляция медленной операции
    time.sleep(random.uniform(2, 5))
    return jsonify({'message': 'This was slow'})

@app.route('/api/error')
def error_endpoint():
    """Endpoint с ошибками"""
    if random.random() < 0.5:
        raise Exception("Random error occurred!")
    return jsonify({'message': 'Success'})

@app.route('/api/transactions')
def complex_transaction():
    """Сложная транзакция с множеством операций"""
    
    # Шаг 1: Валидация
    with apm.capture_span('validate_request', span_type='app'):
        time.sleep(0.05)
        apm.label(validation='passed')
    
    # Шаг 2: Получение данных
    with apm.capture_span('fetch_data', span_type='db.postgresql'):
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM transactions LIMIT 10")
        data = cursor.fetchall()
        cursor.close()
        conn.close()
    
    # Шаг 3: Обработка
    with apm.capture_span('process_data', span_type='app'):
        time.sleep(0.1)
        processed = [{'id': row[0], 'amount': row[1]} for row in data]
    
    # Шаг 4: Сохранение результата
    with apm.capture_span('save_result', span_type='cache'):
        redis_client.setex(f'tx:result:{random.randint(1,1000)}', 
                          600, 
                          str(processed))
    
    return jsonify({'transactions': processed, 'count': len(processed)})

def fetch_users_from_db():
    """Получение пользователей из БД"""
    # Автоматическая инструментация DB queries
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Медленный запрос
    time.sleep(random.uniform(0.1, 0.3))
    cursor.execute("SELECT id, name, email FROM users")
    
    users = [
        {'id': row[0], 'name': row[1], 'email': row[2]}
        for row in cursor.fetchall()
    ]
    
    cursor.close()
    conn.close()
    return users

def enrich_users(users):
    """Обогащение данных пользователей"""
    import requests
    
    # APM автоматически отследит HTTP requests
    for user in users:
        try:
            # Внешний API вызов
            response = requests.get(
                f'http://external-api:8000/user/{user["id"]}/details',
                timeout=1
            )
            if response.ok:
                user['details'] = response.json()
        except Exception as e:
            apm.capture_exception()
            user['details'] = None
    
    return users

def get_db_connection():
    """Получение подключения к БД"""
    return psycopg2.connect(
        host='postgres',
        database='mydb',
        user='user',
        password='password'
    )

# Custom метрики
@app.before_request
def before_request():
    """Добавляем custom tags к транзакции"""
    elasticapm.set_user_context(
        user_id=request.headers.get('X-User-ID'),
        username=request.headers.get('X-Username')
    )
    
    elasticapm.set_custom_context({
        'user_agent': request.headers.get('User-Agent'),
        'request_id': request.headers.get('X-Request-ID'),
        'api_version': 'v1'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

**Node.js/Express example:**

javascript

```javascript
// app.js
const express = require('express');
const apm = require('elastic-apm-node').start({
  serviceName: 'my-node-app',
  serverUrl: 'http://apm-server:8200',
  environment: 'production',
  captureBody: 'all',
  transactionSampleRate: 1.0
});

const app = express();
const redis = require('redis');
const { Pool } = require('pg');

const redisClient = redis.createClient({
  host: 'redis',
  port: 6379
});

const pgPool = new Pool({
  host: 'postgres',
  database: 'mydb',
  user: 'user',
  password: 'password'
});

app.use(express.json());

app.get('/api/products', async (req, res) => {
  // Custom span
  const span = apm.startSpan('fetch_products', 'db');
  
  try {
    // Check cache
    const cached = await redisClient.get('products:all');
    if (cached) {
      apm.setLabel('cache', 'hit');
      if (span) span.end();
      return res.json(JSON.parse(cached));
    }
    
    apm.setLabel('cache', 'miss');
    
    // Fetch from DB
    const result = await pgPool.query('SELECT * FROM products LIMIT 100');
    const products = result.rows;
    
    // Save to cache
    await redisClient.setex('products:all', 300, JSON.stringify(products));
    
    if (span) span.end();
    res.json(products);
    
  } catch (error) {
    apm.captureError(error);
    if (span) span.end();
    res.status(500).json({ error: error.message });
  }
});

app.get('/api/checkout', async (req, res) => {
  const transaction = apm.currentTransaction;
  
  // Set custom context
  transaction.setCustomContext({
    cart_items: req.query.items,
    payment_method: req.query.payment
  });
  
  // Multiple spans
  const validateSpan = apm.startSpan('validate_cart', 'app');
  await simulateWork(100);
  if (validateSpan) validateSpan.end();
  
  const inventorySpan = apm.startSpan('check_inventory', 'db');
  await simulateWork(200);
  if (inventorySpan) inventorySpan.end();
  
  const paymentSpan = apm.startSpan('process_payment', 'external');
  await simulateWork(500);
  if (paymentSpan) paymentSpan.end();
  
  res.json({ order_id: Math.random().toString(36) });
});

function simulateWork(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

3. **Запусти Elastic APM Stack**:

bash

````bash
# Запусти все сервисы
docker-compose up -d elasticsearch kibana apm-server

# Подожди пока поднимутся (30-60 сек)
docker logs -f kibana

# Запусти приложение
docker-compose up -d app

# Сгенерируй трафик
for i in {1..100}; do
  curl http://localhost:5000/api/users
  curl http://localhost:5000/api/transactions
  curl http://localhost:5000/api/slow
  curl http://localhost:5000/api/error || true
  sleep 0.5
done
```

4. **Открой Kibana и настрой APM**:
```
1. Открой: http://localhost:5601
2. Перейди в: Observability → APM
3. Выбери сервис: my-flask-app
4. Изучи:
   - Transactions: распределение времени ответа
   - Errors: все ошибки с stacktrace
   - Metrics: throughput, latency, error rate
   - Service Map: зависимости между сервисами
````

5. **Создай дашборд для APM метрик в Grafana**:

bash

```bash
# APM метрики доступны через Elasticsearch
# Настрой Elasticsearch data source в Grafana
# URL: http://elasticsearch:9200
# Index: apm-*
```

Queries для Grafana:

json

```json
// Average response time
{
  "query": {
    "bool": {
      "filter": [
        {"range": {"@timestamp": {"gte": "now-15m"}}},
        {"term": {"processor.event": "transaction"}}
      ]
    }
  },
  "aggs": {
    "response_time": {
      "date_histogram": {"field": "@timestamp", "interval": "1m"},
      "aggs": {
        "avg_duration": {"avg": {"field": "transaction.duration.us"}}
      }
    }
  }
}

// Error rate
{
  "query": {
    "bool": {
      "filter": [
        {"range": {"@timestamp": {"gte": "now-15m"}}},
        {"term": {"processor.event": "error"}}
      ]
    }
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {"field": "@timestamp", "interval": "1m"},
      "aggs": {
        "error_count": {"value_count": {"field": "error.id"}}
      }
    }
  }
}
```

### 🚀 Бонус (новое)

**1. Настрой Real User Monitoring (RUM)**:

**Frontend instrumentation:**

html

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>My App with RUM</title>
    <script src="https://unpkg.com/@elastic/apm-rum@5.12.0/dist/bundles/elastic-apm-rum.umd.min.js"></script>
    <script>
        // Инициализация RUM
        var apm = window.elasticApm.init({
            serviceName: 'my-frontend',
            serverUrl: 'http://localhost:8200',
            serviceVersion: '1.0.0',
            environment: 'production'
        });
        
        // Custom transaction
        function searchProducts(query) {
            var transaction = apm.startTransaction('Search Products', 'custom');
            
            // Span для API call
            var span = apm.startSpan('API Call', 'external.http');
            
            fetch('/api/search?q=' + query)
                .then(response => response.json())
                .then(data => {
                    if (span) span.end();
                    if (transaction) transaction.end();
                    displayResults(data);
                })
                .catch(error => {
                    apm.captureError(error);
                    if (span) span.end();
                    if (transaction) transaction.end();
                });
        }
        
        // Track user actions
        document.addEventListener('click', function(e) {
            if (e.target.matches('.product-item')) {
                apm.setCustomContext({
                    product_id: e.target.dataset.productId,
                    product_name: e.target.dataset.productName
                });
            }
        });
        
        // Web Vitals
        apm.observe('longtask', function(list) {
            list.getEntries().forEach(function(entry) {
                apm.captureError(new Error('Long Task detected: ' + entry.duration + 'ms'));
            });
        });
    </script>
</head>
<body>
    <h1>My Application</h1>
    <input type="text" id="search" placeholder="Search products...">
    <div id="results"></div>
    
    <script>
        document.getElementById('search').addEventListener('input', function(e) {
            searchProducts(e.target.value);
        });
    </script>
</body>
</html>
```

**2. Создай custom metrics в приложении**:

python

```python
from elasticapm import Client

apm_client = Client({
    'SERVICE_NAME': 'my-app',
    'SERVER_URL': 'http://apm-server:8200'
})

# Custom метрики
class MetricsCollector:
    def __init__(self, apm_client):
        self.apm = apm_client
    
    def track_business_metric(self, metric_name, value, labels=None):
        """Отправка бизнес-метрики"""
        self.apm.gauge(metric_name, value, labels=labels)
    
    def track_cart_value(self, user_id, cart_total):
        """Трекинг стоимости корзины"""
        self.apm.gauge('cart.total', cart_total, labels={
            'user_id': user_id
        })
    
    def track_conversion(self, funnel_step, success):
        """Трекинг конверсии"""
        self.apm.counter('conversion.steps', labels={
            'step': funnel_step,
            'success': str(success)
        })

metrics = MetricsCollector(apm_client)

@app.route('/api/add-to-cart')
def add_to_cart():
    # ... добавление в корзину
    
    # Трекинг метрики
    metrics.track_cart_value(
        user_id=request.headers.get('X-User-ID'),
        cart_total=calculate_cart_total()
    )
    
    metrics.track_conversion('add_to_cart', True)
    
    return jsonify({'success': True})
```

**3. Настрой SLO мониторинг**:

python

```python
# slo_monitor.py
from elasticapm import Client
import time

class SLOMonitor:
    """Мониторинг SLO/SLA"""
    
    def __init__(self, apm_client):
        self.apm = apm_client
        self.slo_target = 0.999  # 99.9%
        self.latency_target_ms = 500
    
    def check_availability_slo(self):
        """Проверка SLO по доступности"""
        total_requests = self.get_total_requests()
        successful_requests = self.get_successful_requests()
        
        availability = successful_requests / total_requests if total_requests > 0 else 1.0
        error_budget_remaining = (self.slo_target - availability) * total_requests
        
        self.apm.gauge('slo.availability', availability)
        self.apm.gauge('slo.error_budget_remaining', error_budget_remaining)
        
        if availability < self.slo_target:
            self.apm.capture_message(
                f'SLO violation: Availability {availability:.4f} below target {self.slo_target}',
                level='warning'
            )
        
        return {
            'availability': availability,
            'target': self.slo_target,
            'error_budget': error_budget_remaining,
            'in_compliance': availability >= self.slo_target
        }
    
    def check_latency_slo(self):
        """Проверка SLO по латентности"""
        p95_latency = self.get_p95_latency()
        
        self.apm.gauge('slo.latency_p95', p95_latency)
        
        if p95_latency > self.latency_target_ms:
            self.apm.capture_message(
                f'SLO violation: p95 latency {p95_latency}ms above target {self.latency_target_ms}ms',
                level='warning'
            )
        
        return {
            'p95_latency': p95_latency,
            'target': self.latency_target_ms,
            'in_compliance': p95_latency <= self.latency_target_ms
        }

# Периодическая проверка SLO
def monitor_slo():
    monitor = SLOMonitor(apm_client)
    
    while True:
        availability_status = monitor.check_availability_slo()
        latency_status = monitor.check_latency_slo()
        
        print(f"Availability SLO: {availability_status}")
        print(f"Latency SLO: {latency_status}")
        
        time.sleep(60)  # Каждую минуту
```

**4. Профилирование производительности**:

python

```python
from elasticapm.contrib.flask import ElasticAPM
import cProfile
import pstats
from io import StringIO

@app.route('/api/profile')
def profile_endpoint():
    """Endpoint с профилированием"""
    
    # CPU profiling
    profiler = cProfile.Profile()
    profiler.enable()
    
    # Код для профилирования
    result = expensive_operation()
    
    profiler.disable()
    
    # Сохраняем результаты в APM
    s = StringIO()
    stats = pstats.Stats(profiler, stream=s)
    stats.sort_stats('cumulative')
    stats.print_stats(20)
    
    apm.capture_message(
        'Profile results',
        custom={'profile': s.getvalue()}
    )
    
    return result

# Memory profiling
from memory_profiler import profile

@profile  # Декоратор для memory profiling
def memory_intensive_operation():
    large_list = [i for i in range(1000000)]
    return sum(large_list)
```

---
## Модуль 8: Синтетический мониторинг и Uptime (25 минут)

### 🎯 Напоминалка

**Синтетический мониторинг vs Real User Monitoring:**
```
Synthetic Monitoring:
✓ Проактивный (обнаруживает проблемы до пользователей)
✓ Контролируемые условия
✓ Регулярные проверки 24/7
✓ Географическое распределение
✗ Не отражает реальный user experience

Real User Monitoring (RUM):
✓ Реальный user experience
✓ Реальные устройства и сети
✓ Бизнес-метрики
✗ Реактивный (пользователи уже пострадали)
````

**Типы синтетических проверок:**

yaml

````yaml
HTTP/HTTPS Check:
  - Status code
  - Response time
  - SSL certificate
  - Response body contains text

TCP Check:
  - Port availability
  - Connection time

DNS Check:
  - DNS resolution time
  - Correct IP returned

Browser Check (Headless):
  - Full page load
  - JavaScript execution
  - Form submission
  - Multi-step transactions
```

**Локации для проверок:**
```
Multiple Geographic Locations:
- North America (US-East, US-West)
- Europe (London, Frankfurt)
- Asia (Tokyo, Singapore)
- South America (São Paulo)

Цель: Обнаружить региональные проблемы
````

### 💻 Задание

Настрой синтетический мониторинг с Blackbox Exporter и Uptime Kuma:

1. **Добавь Blackbox Exporter в docker-compose.yml**:

yaml

```yaml
  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml
    command:
      - '--config.file=/etc/blackbox_exporter/config.yml'
    restart: unless-stopped
```

2. **Создай blackbox.yml**:

yaml

```yaml
modules:
  # HTTP 2xx check
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []  # defaults to 2xx
      method: GET
      preferred_ip_protocol: "ip4"
      follow_redirects: true
      fail_if_ssl: false
      fail_if_not_ssl: false

  # HTTP check with POST
  http_post_2xx:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"key": "value"}'

  # HTTP check с проверкой содержимого
  http_content_check:
    prober: http
    http:
      fail_if_body_not_matches_regexp:
        - "Welcome"
        - "Status: OK"
      fail_if_body_matches_regexp:
        - "Error"
        - "Exception"

  # HTTPS с проверкой SSL
  https_ssl_check:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]
      fail_if_ssl: false
      fail_if_not_ssl: true
      tls_config:
        insecure_skip_verify: false

  # TCP check
  tcp_connect:
    prober: tcp
    timeout: 5s

  # ICMP (ping) check
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"

  # DNS check
  dns_check:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"

  # SSH check
  ssh_banner:
    prober: tcp
    timeout: 5s
    tcp:
      query_response:
        - expect: "^SSH-2.0-"

  # PostgreSQL check
  postgres_check:
    prober: tcp
    timeout: 5s
    tcp:
      query_response:
        - send: "\x00\x00\x00\x08\x04\xd2\x16\x2f"
```

3. **Обнови prometheus.yml** для Blackbox:

yaml

```yaml
scrape_configs:
  # ... существующие jobs

  # HTTP endpoints
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com
          - http://localhost:5000
          - http://localhost:3000
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  # TCP ports
  - job_name: 'blackbox-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - localhost:5432  # PostgreSQL
          - localhost:6379  # Redis
          - localhost:9090  # Prometheus
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  # ICMP (ping)
  - job_name: 'blackbox-icmp'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
          - 8.8.8.8        # Google DNS
          - 1.1.1.1        # Cloudflare DNS
          - example.com
    relabel_configs:
      - source_labels: [**address**] target_label: __param_target 
	    - source_labels: [__param_target] target_label: instance 
        - target_label: **address** replacement: blackbox-exporter:9115

````

4. **Создай алерты для синтетических проверок**:
```yaml
# synthetic_alerts.yml
groups:
  - name: blackbox_alerts
    rules:
      # Endpoint недоступен
      - alert: EndpointDown
        expr: probe_success == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for more than 5 minutes"
          impact: "Service unavailable for users"

      # Медленный ответ
      - alert: SlowResponse
        expr: probe_duration_seconds > 3
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Slow response from {{ $labels.instance }}"
          description: "Response time is {{ $value }}s (threshold: 3s)"

      # SSL сертификат истекает
      - alert: SSLCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expiring soon for {{ $labels.instance }}"
          description: "SSL certificate expires in {{ $value }} days"
          action: "Renew SSL certificate"

      # SSL сертификат истек
      - alert: SSLCertExpired
        expr: probe_ssl_earliest_cert_expiry - time() <= 0
        labels:
          severity: critical
        annotations:
          summary: "SSL certificate expired for {{ $labels.instance }}"
          description: "SSL certificate has expired"
          impact: "HTTPS connections will fail"

      # HTTP status code не 2xx
      - alert: HTTPStatusCode5xx
        expr: probe_http_status_code >= 500
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "HTTP 5xx error on {{ $labels.instance }}"
          description: "Status code: {{ $value }}"

      - alert: HTTPStatusCode4xx
        expr: probe_http_status_code >= 400 and probe_http_status_code < 500
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "HTTP 4xx error on {{ $labels.instance }}"
          description: "Status code: {{ $value }}"
```

5. **Добавь Uptime Kuma** (красивый UI для uptime мониторинга):
```yaml
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data
    restart: unless-stopped

volumes:
  uptime-kuma-data:
```

6. **Запусти и настрой**:
```bash
# Запусти сервисы
docker-compose up -d blackbox-exporter uptime-kuma

# Проверь Blackbox
curl "http://localhost:9115/probe?module=http_2xx&target=https://example.com"

# Настрой Uptime Kuma
# Открой: http://localhost:3001
# Создай аккаунт
# Добавь мониторы:
#   - HTTP(s) для веб-сайтов
#   - TCP для портов
#   - Ping для серверов
```

7. **Создай дашборд для синтетического мониторинга в Grafana**:

Импортируй готовый дашборд: ID 7587 (Prometheus Blackbox Exporter)

Или создай свой с панелями:
````

Panel 1: Uptime % Query: avg_over_time(probe_success[24h]) * 100

Panel 2: Response Time Query: probe_duration_seconds

Panel 3: SSL Certificate Days Left Query: (probe_ssl_earliest_cert_expiry - time()) / 86400

Panel 4: HTTP Status Codes Query: probe_http_status_code

Panel 5: Availability Map Type: Status History Query: probe_success

````

### 🚀 Бонус (новое)

**1. Создай Multi-Step Browser Check с Playwright**:
```python
# synthetic_browser_check.py
from playwright.sync_api import sync_playwright
from prometheus_client import Gauge, Counter, start_http_server
import time

# Метрики
check_duration = Gauge('synthetic_check_duration_seconds', 
                       'Duration of synthetic check', 
                       ['check_name', 'step'])
check_success = Gauge('synthetic_check_success', 
                     'Success status of synthetic check',
                     ['check_name'])
check_errors = Counter('synthetic_check_errors_total',
                      'Total errors in synthetic checks',
                      ['check_name', 'error_type'])

def run_login_flow_check():
    """Multi-step синтетическая проверка: логин и покупка"""
    
    check_name = 'ecommerce_purchase_flow'
    start_time = time.time()
    
    try:
        with sync_playwright() as p:
            browser = p.chromium.launch(headless=True)
            page = browser.new_page()
            
            # Step 1: Загрузка главной страницы
            step_start = time.time()
            page.goto('https://example-shop.com')
            check_duration.labels(check_name=check_name, step='homepage').set(
                time.time() - step_start
            )
            
            # Step 2: Поиск товара
            step_start = time.time()
            page.fill('input[name="search"]', 'laptop')
            page.click('button[type="submit"]')
            page.wait_for_selector('.product-list')
            check_duration.labels(check_name=check_name, step='search').set(
                time.time() - step_start
            )
            
            # Step 3: Добавление в корзину
            step_start = time.time()
            page.click('.product-item:first-child .add-to-cart')
            page.wait_for_selector('.cart-notification')
            check_duration.labels(check_name=check_name, step='add_to_cart').set(
                time.time() - step_start
            )
            
            # Step 4: Checkout
            step_start = time.time()
            page.click('a[href="/cart"]')
            page.wait_for_selector('.checkout-button')
            page.click('.checkout-button')
            page.wait_for_url('**/checkout')
            check_duration.labels(check_name=check_name, step='checkout').set(
                time.time() - step_start
            )
            
            # Step 5: Заполнение формы
            step_start = time.time()
            page.fill('input[name="email"]', 'test@example.com')
            page.fill('input[name="card_number"]', '4242424242424242')
            page.fill('input[name="exp_date"]', '12/25')
            page.fill('input[name="cvv"]', '123')
            check_duration.labels(check_name=check_name, step='fill_form').set(
                time.time() - step_start
            )
            
            # Step 6: Submit и проверка успеха
            step_start = time.time()
            page.click('button[type="submit"]')
            page.wait_for_selector('.order-success', timeout=10000)
            check_duration.labels(check_name=check_name, step='submit').set(
                time.time() - step_start
            )
            
            browser.close()
            
            # Успех
            check_success.labels(check_name=check_name).set(1)
            
            total_duration = time.time() - start_time
            print(f"✓ Check {check_name} passed in {total_duration:.2f}s")
            
    except Exception as e:
        check_success.labels(check_name=check_name).set(0)
        check_errors.labels(
            check_name=check_name,
            error_type=type(e).__name__
        ).inc()
        print(f"✗ Check {check_name} failed: {e}")

def run_api_workflow_check():
    """API workflow проверка"""
    import requests
    
    check_name = 'api_workflow'
    
    try:
        # Step 1: Get auth token
        step_start = time.time()
        auth_response = requests.post('https://api.example.com/auth', json={
            'username': 'test',
            'password': 'test123'
        }, timeout=5)
        check_duration.labels(check_name=check_name, step='auth').set(
            time.time() - step_start
        )
        
        if auth_response.status_code != 200:
            raise Exception(f"Auth failed: {auth_response.status_code}")
        
        token = auth_response.json()['token']
        
        # Step 2: Create resource
        step_start = time.time()
        create_response = requests.post(
            'https://api.example.com/resources',
            headers={'Authorization': f'Bearer {token}'},
            json={'name': 'test-resource'},
            timeout=5
        )
        check_duration.labels(check_name=check_name, step='create').set(
            time.time() - step_start
        )
        
        resource_id = create_response.json()['id']
        
        # Step 3: Get resource
        step_start = time.time()
        get_response = requests.get(
            f'https://api.example.com/resources/{resource_id}',
            headers={'Authorization': f'Bearer {token}'},
            timeout=5
        )
        check_duration.labels(check_name=check_name, step='get').set(
            time.time() - step_start
        )
        
        # Step 4: Delete resource
        step_start = time.time()
        delete_response = requests.delete(
            f'https://api.example.com/resources/{resource_id}',
            headers={'Authorization': f'Bearer {token}'},
            timeout=5
        )
        check_duration.labels(check_name=check_name, step='delete').set(
            time.time() - step_start
        )
        
        check_success.labels(check_name=check_name).set(1)
        print(f"✓ API workflow check passed")
        
    except Exception as e:
        check_success.labels(check_name=check_name).set(0)
        check_errors.labels(
            check_name=check_name,
            error_type=type(e).__name__
        ).inc()
        print(f"✗ API workflow check failed: {e}")

if __name__ == '__main__':
    # Запусти Prometheus metrics server
    start_http_server(8000)
    print("Metrics available at http://localhost:8000")
    
    # Запускай проверки каждые 5 минут
    while True:
        print(f"\n{'='*50}")
        print(f"Running synthetic checks at {time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"{'='*50}")
        
        run_login_flow_check()
        run_api_workflow_check()
        
        time.sleep(300)  # 5 минут
```

**2. Создай Geographic Distributed Monitoring**:
```yaml
# docker-compose для разных регионов
# us-east.docker-compose.yml
version: '3.8'

services:
  blackbox-us-east:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-us-east
    ports:
      - "9116:9115"
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml
    environment:
      - LOCATION=us-east
    restart: unless-stopped
```

Prometheus config для нескольких локаций:
```yaml
scrape_configs:
  - job_name: 'blackbox-us-east'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: instance
        replacement: example.com
      - target_label: region
        replacement: us-east
      - target_label: __address__
        replacement: blackbox-us-east:9115

  - job_name: 'blackbox-eu-west'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: instance
        replacement: example.com
      - target_label: region
        replacement: eu-west
      - target_label: __address__
        replacement: blackbox-eu-west:9115
```

**3. Настрой Status Page** с использованием Cachet:
```yaml
  cachet:
    image: cachethq/docker:latest
    container_name: cachet
    ports:
      - "8001:8000"
    environment:
      - DB_DRIVER=sqlite
      - APP_KEY=base64:yourapplicationkey
      - APP_URL=http://localhost:8001
    volumes:
      - cachet-data:/var/www/html/database
    restart: unless-stopped

volumes:
  cachet-data:
```

Или используй **Upptime** (GitHub-based):
```yaml
# .github/workflows/upptime.yml
name: Upptime CI
on:
  schedule:
    - cron: "*/5 * * * *"
  workflow_dispatch:

jobs:
  release:
    name: Check status
    runs-on: ubuntu-latest
    steps:
      - uses: upptime/upptime@v1.28.0
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

`.upptimerc.yml`:
```yaml
owner: your-username
repo: upptime

sites:
  - name: Website
    url: https://example.com
    maxResponseTime: 5000
  - name: API
    url: https://api.example.com
    maxResponseTime: 3000
  - name: Blog
    url: https://blog.example.com

status-website:
  cname: status.example.com
  name: Status Page
  introTitle: "Service Status"
  introMessage: Real-time status and uptime monitoring
```

---
## Модуль 9: Мониторинг инфраструктуры и облачных сервисов (30 минут)

### 🎯 Напоминалка

**Уровни мониторинга инфраструктуры:**

```
┌─────────────────────────────────────┐
│ Cloud Services (AWS/GCP/Azure)      │
│ - EC2, S3, RDS, Lambda              │
│ - Billing, Quotas, API limits       │
└─────────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ Kubernetes / Orchestration          │
│ - Cluster health                    │
│ - Pod/Node metrics                  │
│ - Resource quotas                   │
└─────────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ Containers (Docker)                 │
│ - Container metrics                 │
│ - Image vulnerabilities             │
└─────────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ Infrastructure (VMs, Bare Metal)    │
│ - CPU, Memory, Disk, Network        │
│ - Hardware health                   │
└─────────────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│ Network                             │
│ - Switches, Routers                 │
│ - Bandwidth, Latency                │
│ - SNMP, NetFlow                     │
└─────────────────────────────────────┘
```

**Kubernetes мониторинг - ключевые компоненты:**

yaml

```yaml
kube-state-metrics:
  # Метрики о состоянии K8s объектов
  - Deployments: replicas, available, unavailable
  - Pods: phase, restarts, conditions
  - Nodes: capacity, allocatable, conditions
  - PersistentVolumes: phase, capacity
  - Jobs: succeeded, failed, active

metrics-server:
  # Реальные метрики использования ресурсов
  - CPU usage (по контейнерам/подам/нодам)
  - Memory usage
  - Используется для HPA

cAdvisor:
  # Container-level метрики
  - CPU/Memory usage
  - Network I/O
  - Filesystem I/O
  - Встроен в kubelet
```

**Cloud Provider метрики:**

yaml

````yaml
AWS CloudWatch:
  - EC2: CPU, Network, Disk I/O
  - RDS: Connections, IOPS, Storage
  - S3: Requests, Bandwidth, Storage
  - Lambda: Invocations, Duration, Errors
  - ELB: Request Count, Latency, HTTP codes
  - Billing: Estimated charges

GCP Monitoring:
  - Compute Engine: CPU, Disk, Network
  - Cloud SQL: Queries, Connections
  - Cloud Storage: Operations, Bandwidth
  - Cloud Functions: Executions, Memory
  - Load Balancer: Request rate, Latency

Azure Monitor:
  - Virtual Machines: CPU, Memory, Disk
  - SQL Database: DTU, Storage, Connections
  - Storage: Transactions, Ingress/Egress
  - Functions: Execution count, Duration
  - Application Insights: APM метрики
```

**SNMP мониторинг (Network devices):**
```
SNMP OIDs (Object Identifiers):
- .1.3.6.1.2.1.1.1.0        # System description
- .1.3.6.1.2.1.1.3.0        # Uptime
- .1.3.6.1.2.1.2.2.1.10.*   # Interface inbound octets
- .1.3.6.1.2.1.2.2.1.16.*   # Interface outbound octets
- .1.3.6.1.2.1.25.1.1.0     # Host CPU load

SNMP Versions:
v1: Basic, no encryption
v2c: Community strings, better performance
v3: Authentication + Encryption (рекомендуется)
````

**Cost Monitoring:**

yaml

```yaml
Cloud Cost Metrics:
  - Daily/Monthly spend by service
  - Cost per application/team
  - Unutilized resources
  - Reserved vs On-Demand usage
  - Savings opportunities

Infrastructure Cost:
  - Compute: Instance types, utilization
  - Storage: Type, size, IOPS
  - Network: Data transfer, NAT gateways
  - Managed Services: RDS, Lambda, etc.
```

### 💻 Задание

Настрой комплексный мониторинг Kubernetes кластера:

1. **Установи kube-state-metrics**:

yaml

```yaml
# kube-state-metrics-deployment.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
  - apiGroups: [""]
    resources:
      - configmaps
      - secrets
      - nodes
      - pods
      - services
      - resourcequotas
      - replicationcontrollers
      - limitranges
      - persistentvolumeclaims
      - persistentvolumes
      - namespaces
      - endpoints
    verbs: ["list", "watch"]
  - apiGroups: ["apps"]
    resources:
      - statefulsets
      - daemonsets
      - deployments
      - replicasets
    verbs: ["list", "watch"]
  - apiGroups: ["batch"]
    resources:
      - cronjobs
      - jobs
    verbs: ["list", "watch"]
  - apiGroups: ["autoscaling"]
    resources:
      - horizontalpodautoscalers
    verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: monitoring
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
        - name: kube-state-metrics
          image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1
          ports:
            - name: http-metrics
              containerPort: 8080
            - name: telemetry
              containerPort: 8081
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 5
            timeoutSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  ports:
    - name: http-metrics
      port: 8080
      targetPort: http-metrics
    - name: telemetry
      port: 8081
      targetPort: telemetry
  selector:
    app: kube-state-metrics
```

Примени:

bash

```bash
kubectl create namespace monitoring
kubectl apply -f kube-state-metrics-deployment.yaml
```

2. **Настрой Prometheus для Kubernetes мониторинга**:

Обнови `prometheus.yml`:

yaml

```yaml
scrape_configs:
  # ... существующие jobs

  # Kubernetes API server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # Kubernetes nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

  # Kubernetes nodes (Kubelet)
  - job_name: 'kubernetes-nodes-kubelet'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

  # Kubernetes pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # kube-state-metrics
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.monitoring.svc.cluster.local:8080']

  # cAdvisor (встроен в kubelet)
  - job_name: 'kubernetes-cadvisor'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

3. **Создай Kubernetes-specific алерты**:

yaml

```yaml
# kubernetes_alerts.yml
groups:
  - name: kubernetes_cluster
    rules:
      # Нода недоступна
      - alert: KubernetesNodeNotReady
        expr: kube_node_status_condition{condition="Ready",status="true"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Kubernetes node not ready"
          description: "Node {{ $labels.node }} has been unready for more than 5 minutes"

      # Высокое использование CPU на ноде
      - alert: KubernetesNodeHighCPU
        expr: |
          (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100 > 80
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on node {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}%"

      # Высокое использование памяти на ноде
      - alert: KubernetesNodeHighMemory
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory on node {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}%"

      # Pod в CrashLoopBackOff
      - alert: KubernetesPodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash looping"
          description: "Pod is restarting frequently"

      # Pod не может запуститься
      - alert: KubernetesPodNotReady
        expr: |
          sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown"}) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
          description: "Pod has been in {{ $labels.phase }} state for more than 10 minutes"

      # Deployment replicas mismatch
      - alert: KubernetesDeploymentReplicasMismatch
        expr: |
          kube_deployment_spec_replicas != kube_deployment_status_replicas_available
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} replicas mismatch"
          description: "Desired: {{ $value }}, Available: {{ $labels.replicas_available }}"

      # StatefulSet replicas mismatch
      - alert: KubernetesStatefulSetReplicasMismatch
        expr: |
          kube_statefulset_status_replicas_ready != kube_statefulset_status_replicas
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} replicas mismatch"

      # DaemonSet pods не на всех нодах
      - alert: KubernetesDaemonSetRolloutStuck
        expr: |
          kube_daemonset_status_number_ready / kube_daemonset_status_desired_number_scheduled * 100 < 100
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset }} rollout stuck"

      # Job failed
      - alert: KubernetesJobFailed
        expr: |
          kube_job_status_failed > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Job {{ $labels.namespace }}/{{ $labels.job_name }} failed"

      # PVC pending
      - alert: KubernetesPersistentVolumeClaimPending
        expr: |
          kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} pending"

      # Container OOMKilled
      - alert: KubernetesContainerOOMKilled
        expr: |
          (kube_pod_container_status_restarts_total - kube_pod_container_status_restarts_total offset 10m >= 1)
          and ignoring (reason) min_over_time(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[10m]) == 1
        labels:
          severity: warning
        annotations:
          summary: "Container OOMKilled in {{ $labels.namespace }}/{{ $labels.pod }}"
          description: "Container {{ $labels.container }} was OOMKilled"

  - name: kubernetes_resources
    rules:
      # Высокое использование CPU контейнером
      - alert: KubernetesContainerHighCPU
        expr: |
          sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace, pod, container)
          / 
          sum(kube_pod_container_resource_limits{resource="cpu"}) by (namespace, pod, container)
          * 100 > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage in container"
          description: "Container {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} using {{ $value }}% of limit"

      # Высокое использование памяти контейнером
      - alert: KubernetesContainerHighMemory
        expr: |
          sum(container_memory_working_set_bytes{container!=""}) by (namespace, pod, container)
          /
          sum(kube_pod_container_resource_limits{resource="memory"}) by (namespace, pod, container)
          * 100 > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage in container"
          description: "Container {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} using {{ $value }}% of limit"

      # Недостаточно ресурсов на ноде
      - alert: KubernetesNodeResourcePressure
        expr: |
          kube_node_status_condition{condition=~"MemoryPressure|DiskPressure|PIDPressure",status="true"} == 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.node }} under {{ $labels.condition }}"
```

4. **Создай комплексный Kubernetes дашборд в Grafana**:

Импортируй готовые дашборды:

- **Kubernetes Cluster Monitoring**: ID 7249
- **Kubernetes Pod Resources**: ID 6417
- **Node Exporter Full**: ID 1860

Или создай свой с панелями:

**Panel 1: Cluster Overview**

promql

```promql
# Всего нод
count(kube_node_info)

# Нод Ready
sum(kube_node_status_condition{condition="Ready",status="true"})

# Всего Pods
count(kube_pod_info)

# Running Pods
count(kube_pod_status_phase{phase="Running"})
```

**Panel 2: Resource Usage**

promql

```promql
# CPU Requests vs Allocatable
sum(kube_pod_container_resource_requests{resource="cpu"}) 
/ 
sum(kube_node_status_allocatable{resource="cpu"}) * 100

# Memory Requests vs Allocatable
sum(kube_pod_container_resource_requests{resource="memory"}) 
/ 
sum(kube_node_status_allocatable{resource="memory"}) * 100
```

**Panel 3: Pod Status by Phase**

promql

```promql
sum(kube_pod_status_phase{phase="Running"})
sum(kube_pod_status_phase{phase="Pending"})
sum(kube_pod_status_phase{phase="Failed"})
```

**Panel 4: Top Pods by CPU**

promql

```promql
topk(10, 
  sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace, pod)
)
```

**Panel 5: Top Pods by Memory**

promql

```promql
topk(10,
  sum(container_memory_working_set_bytes{container!=""}) by (namespace, pod)
)
```

**Panel 6: Network Traffic**

promql

```promql
# Inbound
sum(rate(container_network_receive_bytes_total[5m])) by (namespace, pod)

# Outbound
sum(rate(container_network_transmit_bytes_total[5m])) by (namespace, pod)
```

5. **Тестирование мониторинга**:

bash

```bash
# Создай тестовое приложение
kubectl create deployment test-app --image=nginx --replicas=3

# Посмотри метрики
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Открой Prometheus
http://localhost:9090

# Попробуй запросы:
# - kube_deployment_status_replicas{deployment="test-app"}
# - kube_pod_info{created_by_name="test-app"}
# - sum(rate(container_cpu_usage_seconds_total{pod=~"test-app.*"}[5m]))

# Симулируй проблему
kubectl scale deployment test-app --replicas=10
kubectl delete pod -l app=test-app --force --grace-period=0

# Смотри алерты в Alertmanager
http://localhost:9093
```

### 🚀 Бонус (новое)

**1. Настрой AWS CloudWatch Exporter** для мониторинга AWS ресурсов:

yaml

```yaml
  cloudwatch-exporter:
    image: prom/cloudwatch-exporter:latest
    container_name: cloudwatch-exporter
    ports:
      - "9106:9106"
    volumes:
      - ./cloudwatch-exporter.yml:/config/config.yml
      - ~/.aws:/root/.aws:ro
    command:
      - '/bin/cloudwatch_exporter'
      - '/config/config.yml'
    restart: unless-stopped
```

`cloudwatch-exporter.yml`:

yaml

```yaml
region: us-east-1
metrics:
  # EC2 Instances
  - aws_namespace: AWS/EC2
    aws_metric_name: CPUUtilization
    aws_dimensions:
      - InstanceId
    aws_statistics:
      - Average
    period_seconds: 300
    range_seconds: 600

  - aws_namespace: AWS/EC2
    aws_metric_name: NetworkIn
    aws_dimensions:
      - InstanceId
    aws_statistics:
      - Sum
    period_seconds: 300

  # RDS
  - aws_namespace: AWS/RDS
    aws_metric_name: DatabaseConnections
    aws_dimensions:
      - DBInstanceIdentifier
    aws_statistics:
      - Average
    period_seconds: 300

  - aws_namespace: AWS/RDS
    aws_metric_name: ReadLatency
    aws_dimensions:
      - DBInstanceIdentifier
    aws_statistics:
      - Average
    period_seconds: 300

  # ELB
  - aws_namespace: AWS/ELB
    aws_metric_name: RequestCount
    aws_dimensions:
      - LoadBalancerName
    aws_statistics:
      - Sum
    period_seconds: 300

  - aws_namespace: AWS/ELB
    aws_metric_name: Latency
    aws_dimensions:
      - LoadBalancerName
    aws_statistics:
      - Average
    period_seconds: 300

  # Lambda
  - aws_namespace: AWS/Lambda
    aws_metric_name: Invocations
    aws_dimensions:
      - FunctionName
    aws_statistics:
      - Sum
    period_seconds: 300

  - aws_namespace: AWS/Lambda
    aws_metric_name: Duration
    aws_dimensions:
      - FunctionName
    aws_statistics:
      - Average
    period_seconds: 300

  - aws_namespace: AWS/Lambda
    aws_metric_name: Errors
    aws_dimensions:
      - FunctionName
    aws_statistics:
      - Sum
    period_seconds: 300

  # S3
  - aws_namespace: AWS/S3
    aws_metric_name: NumberOfObjects
    aws_dimensions:
      - BucketName
      - StorageType
    aws_statistics:
      - Average
    period_seconds: 86400  # Once per day

  # Billing
  - aws_namespace: AWS/Billing
    aws_metric_name: EstimatedCharges
    aws_dimensions:
      - Currency
    aws_statistics:
      - Maximum
    period_seconds: 86400
```

Добавь в `prometheus.yml`:

yaml

```yaml
scrape_configs:
  - job_name: 'cloudwatch'
    static_configs:
      - targets: ['cloudwatch-exporter:9106']
```

**2. Мониторинг Network с SNMP Exporter**:

yaml

```yaml
  snmp-exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp-exporter
    ports:
      - "9116:9116"
    volumes:
      - ./snmp.yml:/etc/snmp_exporter/snmp.yml
    command:
      - '--config.file=/etc/snmp_exporter/snmp.yml'
    restart: unless-stopped
```

`snmp.yml` (пример для Cisco):

yaml

```yaml
auths:
  public_v2:
    community: public
    security_level: noAuthNoPriv
    auth_protocol: MD5
    priv_protocol: DES
    version: 2

modules:
  if_mib:
    walk:
      - 1.3.6.1.2.1.2.2.1.2   # ifDescr
      - 1.3.6.1.2.1.2.2.1.10  # ifInOctets
      - 1.3.6.1.2.1.2.2.1.16  # ifOutOctets
      - 1.3.6.1.2.1.2.2.1.8   # ifOperStatus
    
    lookups:
      - source_indexes: [ifIndex]
        lookup: ifDescr
    
    overrides:
      ifDescr:
        type: DisplayString
      ifOperStatus:
        type: gauge
```

Prometheus config:

yaml

```yaml
scrape_configs:
  - job_name: 'snmp'
    static_configs:
      - targets:
          - 192.168.1.1  # Switch IP
          - 192.168.1.2  # Router IP
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
```

**3. Cost Monitoring Dashboard**:

Создай панели в Grafana:

promql

```promql
# AWS Estimated Charges (last value)
aws_billing_estimated_charges_maximum{currency="USD"}

# Daily cost trend
increase(aws_billing_estimated_charges_maximum{currency="USD"}[1d])

# Cost by service (requires detailed billing)
sum by (service) (aws_cloudwatch_billing_estimated_charges_average)

# Top 10 most expensive resources
topk(10, 
  sum by (resource_id) (aws_resource_cost_daily)
)

# Unutilized resources cost
sum(aws_ec2_cpu_utilization_average < 10) * avg(aws_ec2_pricing_hourly)

# Kubernetes cost by namespace
sum by (namespace) (
  avg_over_time(container_cpu_usage_seconds_total[1h]) * 
  scalar(aws_ec2_pricing_hourly / 8)  # assuming 8 vCPUs per instance
) * 24 * 30  # Monthly estimate
```

**4. Infrastructure as Code Monitoring**:

Мониторинг Terraform state:

python

```python
# terraform_exporter.py
from prometheus_client import start_http_server, Gauge
import json
import subprocess
import time

# Метрики
terraform_resource_count = Gauge('terraform_resource_count', 
                                 'Number of resources in Terraform state',
                                 ['workspace', 'type'])
terraform_drift_detected = Gauge('terraform_drift_detected',
                                'Drift detected in Terraform',
                                ['workspace'])

def check_terraform_state():
    """Проверка Terraform state"""
    try:
        # Get resource count
        result = subprocess.run(
            ['terraform', 'state', 'list'],
            capture_output=True,
            text=True,
            check=True
        )
        
        resources = result.stdout.strip().split('\n')
        resource_types = {}
        
        for resource in resources:
            if resource:
                resource_type = resource.split('.')[0]
                resource_types[resource_type] = resource_types.get(resource_type, 0) + 1
        
        workspace = subprocess.run(
            ['terraform', 'workspace', 'show'],
            capture_output=True,
            text=True,
            check=True
        ).stdout.strip()
        
        # Update metrics
        for rtype, count in resource_types.items():
            terraform_resource_count.labels(
                workspace=workspace,
                type=rtype
            ).set(count)
        
        # Check for drift
        plan_result = subprocess.run(
            ['terraform', 'plan', '-detailed-exitcode'],
            capture_output=True
        )
        
        # Exit code 2 means changes detected
        if plan_result.returncode == 2:
            terraform_drift_detected.labels(workspace=workspace).set(1)
            print(f"⚠️  Drift detected in {workspace}")
        else:
            terraform_drift_detected.labels(workspace=workspace).set(0)
            print(f"✓ No drift in {workspace}")
            
    except Exception as e:
        print(f"Error checking Terraform: {e}")

if __name__ == '__main__':
    start_http_server(8000)
    print("Terraform exporter started on :8000")
    
    while True:
        check_terraform_state()
        time.sleep(300)  # Check every 5 minutes
```

---

**Чеклист модуля 9:**

- ✅ Настроил kube-state-metrics
- ✅ Создал Kubernetes алерты
- ✅ Построил K8s дашборды
- ✅Интегрировал cloud monitoring (AWS/GCP/Azure)
- ✅ Настроил network monitoring (SNMP)
- ✅ Мониторинг стоимости инфраструктуры


---
## Модуль 10: Best Practices, Incident Management и On-Call (30 минут)

### 🎯 Напоминалка

**Incident Response Process:**

```
┌─────────────────────────────────────────┐
│ 1. Detection                            │
│    - Alert fires                        │
│    - User report                        │
│    - Monitoring anomaly                 │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│ 2. Triage & Assessment                  │
│    - Severity classification            │
│    - Impact assessment                  │
│    - Incident commander assignment      │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│ 3. Investigation                        │
│    - Check logs, metrics, traces        │
│    - Follow runbooks                    │
│    - Gather data                        │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│ 4. Mitigation                          │
│    - Apply fix/workaround              │
│    - Rollback if needed                │
│    - Scale resources                   │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│ 5. Resolution & Recovery               │
│    - Service restored                  │
│    - Monitoring confirmed              │
│    - Communication sent                │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│ 6. Post-Mortem                         │
│    - Root cause analysis               │
│    - Action items                      │
│    - Documentation update              │
└─────────────────────────────────────────┘
```

**Severity Levels:**

yaml

```yaml
SEV-1 (Critical):
  Impact: Complete service outage
  Response: Immediate, 24/7
  Example: Database down, site unreachable
  SLA: 15 min response time
  
SEV-2 (High):
  Impact: Major feature broken
  Response: Within 1 hour
  Example: Payment processing failing
  SLA: 1 hour response time
  
SEV-3 (Medium):
  Impact: Minor feature degraded
  Response: Within 4 hours
  Example: Slow API responses
  SLA: 4 hour response time
  
SEV-4 (Low):
  Impact: Cosmetic or minor issue
  Response: Next business day
  Example: UI bug, typo
  SLA: 24 hour response time
```

**On-Call Best Practices:**

yaml

```yaml
Rotation Schedule:
  - 1 week rotations (not too short, not too long)
  - Primary + Secondary on-call
  - Clear handoff procedures
  - Compensation (time off or pay)

Response Requirements:
  - Acknowledge within 5 minutes
  - Initial assessment within 15 minutes
  - Regular status updates (every 30 min)
  - Clear escalation path

Workload Management:
  - Max 2-3 incidents per shift
  - Escalate if overwhelmed
  - Hand off if shift ends during incident
  - Post-incident recovery time

Tools:
  - PagerDuty, OpsGenie, VictorOps
  - Slack/Teams war rooms
  - Incident tracking (Jira, Linear)
  - Video conferencing for major incidents
```

**Runbook Structure:**

yaml

```yaml
Runbook Template:
  Title: "[Service] - [Problem Description]"
  
  Overview:
    - What is this runbook for?
    - When to use it?
  
  Symptoms:
    - How do you know there's a problem?
    - What alerts fire?
    - What users experience?
  
  Impact:
    - Affected services/features
    - User impact level
    - Business impact
  
  Investigation:
    - Links to dashboards
    - Key metrics to check
    - Log queries to run
    - Common root causes
  
  Resolution Steps:
    - Step 1: Check X
    - Step 2: Try Y
    - Step 3: If still failing, do Z
  
  Escalation:
    - When to escalate?
    - Who to contact?
    - Contact information
  
  Prevention:
    - How to prevent this in future?
    - Monitoring improvements needed
  
  Related Links:
    - Dashboard: [URL]
    - Code: [GitHub link]
    - Previous incidents: [Links]
```

**Alert Fatigue - как бороться:**

yaml

```yaml
Symptoms:
  - Too many alerts (>10 per day per person)
  - Low signal-to-noise ratio
  - Alerts ignored/snoozed
  - Alert blindness

Solutions:
  1. Alert Hygiene:
     - Delete noisy alerts
     - Increase thresholds
     - Add proper for: durations
     - Use rate of change, not absolute values
  
  2. Alert Grouping:
     - Group related alerts
     - Send digest instead of individual
     - Use inhibition rules
  
  3. Alert Prioritization:
     - Critical alerts = page
     - Warning alerts = ticket
     - Info alerts = dashboard only
  
  4. Context-Aware Alerting:
     - Business hours vs off-hours
     - Maintenance windows
     - Expected anomalies (deploy, traffic spike)
  
  5. Self-Healing:
     - Auto-remediation where safe
     - Alert only if auto-fix fails
```

**SRE Principles:**

yaml

```yaml
Error Budget:
  Definition: (1 - SLO) × time_period
  Example: 
    SLO = 99.9%
    Error budget = 0.1% = 43 minutes/month
  
  Usage:
    - Track error budget consumption
    - Stop releases if budget exhausted
    - Balance velocity vs reliability

Toil Reduction:
  Toil: Manual, repetitive, automatable work
  
  Identify Toil:
    - Manual steps in runbooks
    - Repetitive tickets
    - Data entry tasks
  
  Reduce:
    - Automate with scripts
    - Self-service tools
    - Better monitoring (less investigation)
  
  Target: <50% time on toil

Blameless Post-Mortems:
  Focus: Systems and processes, not people
  
  Questions:
    - What happened?
    - Why did it happen?
    - How did we respond?
    - What can we improve?
  
  Action Items:
    - Concrete, assignable
    - Time-bound
    - Tracked to completion
```

**Capacity Planning:**

yaml

```yaml
Metrics to Track:
  - CPU/Memory utilization trends
  - Disk growth rate
  - Network bandwidth usage
  - Request rate growth
  - Database connection pool usage

Forecasting:
  - Linear regression for growth
  - Seasonal patterns
  - Business events (Black Friday, etc.)
  - New feature impact

Planning Horizon:
  - 3 months: Tactical (add instances)
  - 6 months: Strategic (upgrade tier)
  - 12 months: Architectural (redesign)

Buffer:
  - Always keep 20-30% headroom
  - Account for failure scenarios (N+1, N+2)
  - Peak vs average load
```

### 💻 Задание

Создай комплексную систему Incident Management:

1. **Настрой PagerDuty/Opsgenie** (или используй webhook для симуляции):

Создай webhook receiver для алертов:

python

```python
# incident_manager.py
from flask import Flask, request, jsonify
from datetime import datetime
import json
import sqlite3
import requests

app = Flask(__name__)

# База данных для инцидентов
def init_db():
    conn = sqlite3.connect('incidents.db')
    c = conn.cursor()
    c.execute('''
        CREATE TABLE IF NOT EXISTS incidents (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            alert_name TEXT,
            severity TEXT,
            status TEXT,
            started_at TIMESTAMP,
            acknowledged_at TIMESTAMP,
            resolved_at TIMESTAMP,
            description TEXT,
            assigned_to TEXT,
            duration_seconds INTEGER
        )
    ''')
    conn.commit()
    conn.close()

init_db()

# Severity mapping
SEVERITY_MAP = {
    'critical': 'SEV-1',
    'warning': 'SEV-2',
    'info': 'SEV-3'
}

@app.route('/webhook/alert', methods=['POST'])
def receive_alert():
    """Принимает алерты от Alertmanager"""
    data = request.json
    
    for alert in data.get('alerts', []):
        alert_name = alert['labels'].get('alertname', 'Unknown')
        severity = alert['labels'].get('severity', 'warning')
        status = alert['status']  # firing or resolved
        
        if status == 'firing':
            # Новый инцидент
            incident_id = create_incident(alert_name, severity, alert)
            
            # Отправка в Slack
            send_slack_notification(incident_id, alert_name, severity, alert)
            
            # Создание war room (если critical)
            if severity == 'critical':
                create_war_room(incident_id, alert_name)
            
            return jsonify({
                'status': 'ok',
                'incident_id': incident_id,
                'message': 'Incident created'
            })
            
        elif status == 'resolved':
            # Резолв инцидента
            resolve_incident(alert_name)
            return jsonify({
                'status': 'ok',
                'message': 'Incident resolved'
            })
    
    return jsonify({'status': 'ok'})

def create_incident(alert_name, severity, alert):
    """Создание инцидента в БД"""
    conn = sqlite3.connect('incidents.db')
    c = conn.cursor()
    
    description = alert.get('annotations', {}).get('description', '')
    
    c.execute('''
        INSERT INTO incidents 
        (alert_name, severity, status, started_at, description)
        VALUES (?, ?, ?, ?, ?)
    ''', (
        alert_name,
        SEVERITY_MAP.get(severity, 'SEV-3'),
        'open',
        datetime.now(),
        description
    ))
    
    incident_id = c.lastrowid
    conn.commit()
    conn.close()
    
    print(f"📢 Incident #{incident_id} created: {alert_name} ({severity})")
    return incident_id

def resolve_incident(alert_name):
    """Резолв инцидента"""
    conn = sqlite3.connect('incidents.db')
    c = conn.cursor()
    
    # Найти открытый инцидент
    c.execute('''
        SELECT id, started_at FROM incidents 
        WHERE alert_name = ? AND status = 'open'
        ORDER BY started_at DESC LIMIT 1
    ''', (alert_name,))
    
    result = c.fetchone()
    if result:
        incident_id, started_at = result
        resolved_at = datetime.now()
        started = datetime.fromisoformat(started_at)
        duration = (resolved_at - started).total_seconds()
        
        c.execute('''
            UPDATE incidents 
            SET status = 'resolved', 
                resolved_at = ?,
                duration_seconds = ?
            WHERE id = ?
        ''', (resolved_at, duration, incident_id))
        
        conn.commit()
        print(f"✅ Incident #{incident_id} resolved after {duration:.0f}s")
        
        # Отправка уведомления о резолве
        send_slack_notification(
            incident_id, 
            alert_name, 
            'resolved', 
            {'duration': duration}
        )
    
    conn.close()

def send_slack_notification(incident_id, alert_name, severity, data):
    """Отправка в Slack"""
    webhook_url = "YOUR_SLACK_WEBHOOK_URL"
    
    if severity == 'resolved':
        color = 'good'
        title = f"✅ Incident #{incident_id} Resolved"
        duration = data.get('duration', 0)
        message = f"Duration: {duration:.0f} seconds"
    elif severity == 'critical':
        color = 'danger'
        title = f"🚨 SEV-1 Incident #{incident_id}"
        message = data.get('annotations', {}).get('description', '')
    else:
        color = 'warning'
        title = f"⚠️ Incident #{incident_id}"
        message = data.get('annotations', {}).get('description', '')
    
    payload = {
        "attachments": [{
            "color": color,
            "title": title,
            "text": f"*Alert:* {alert_name}\n{message}",
            "fields": [
                {
                    "title": "Severity",
                    "value": SEVERITY_MAP.get(severity, 'SEV-3'),
                    "short": True
                },
                {
                    "title": "Runbook",
                    "value": f"<https://wiki.company.com/runbooks/{alert_name}|View>",
                    "short": True
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "Acknowledge",
                    "url": f"http://localhost:5000/incident/{incident_id}/ack"
                },
                {
                    "type": "button",
                    "text": "View Dashboard",
                    "url": "http://localhost:3000"
                }
            ]
        }]
    }
    
    try:
        requests.post(webhook_url, json=payload)
    except:
        pass  # Graceful failure if Slack not configured

def create_war_room(incident_id, alert_name):
    """Создание Slack war room для критичных инцидентов"""
    # В реальности используй Slack API для создания канала
    print(f"🔥 War room created: #incident-{incident_id}-{alert_name}")

@app.route('/incident/<int:incident_id>/ack', methods=['POST'])
def acknowledge_incident(incident_id):
    """Acknowledge инцидента"""
    conn = sqlite3.connect('incidents.db')
    c = conn.cursor()
    
    assigned_to = request.json.get('user', 'Unknown')
    
    c.execute('''
        UPDATE incidents 
        SET acknowledged_at = ?, assigned_to = ?
        WHERE id = ?
    ''', (datetime.now(), assigned_to, incident_id))
    
    conn.commit()
    conn.close()
    
    print(f"👤 Incident #{incident_id} acknowledged by {assigned_to}")
    
    return jsonify({'status': 'ok'})

@app.route('/incidents', methods=['GET'])
def list_incidents():
    """Список всех инцидентов"""
    conn = sqlite3.connect('incidents.db')
    c = conn.cursor()
    
    status = request.args.get('status', 'all')
    
    if status == 'open':
        c.execute('SELECT * FROM incidents WHERE status = "open" ORDER BY started_at DESC')
    else:
        c.execute('SELECT * FROM incidents ORDER BY started_at DESC LIMIT 100')
    
    columns = [desc[0] for desc in c.description]
    incidents = [dict(zip(columns, row)) for row in c.fetchall()]
    
    conn.close()
    
    return jsonify(incidents)

@app.route('/incidents/stats', methods=['GET'])
def incident_stats():
    """Статистика инцидентов"""
    conn = sqlite3.connect('incidents.db')
    c = conn.cursor()
    
    # MTTR (Mean Time To Resolve)
    c.execute('''
        SELECT AVG(duration_seconds) as mttr
        FROM incidents 
        WHERE status = 'resolved' AND duration_seconds IS NOT NULL
    ''')
    mttr = c.fetchone()[0] or 0
    
    # Incidents by severity
    c.execute('''
        SELECT severity, COUNT(*) as count
        FROM incidents
        GROUP BY severity
    ''')
    by_severity = dict(c.fetchall())
    
    # Open incidents
    c.execute('SELECT COUNT(*) FROM incidents WHERE status = "open"')
    open_count = c.fetchone()[0]
    
    # Total incidents
    c.execute('SELECT COUNT(*) FROM incidents')
    total_count = c.fetchone()[0]
    
    conn.close()
    
    return jsonify({
        'mttr_seconds': mttr,
        'mttr_minutes': mttr / 60,
        'open_incidents': open_count,
        'total_incidents': total_count,
        'by_severity': by_severity
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)
```

2. **Настрой Alertmanager для отправки в Incident Manager**:

Обнови `alertmanager.yml`:

yaml

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'incident-manager'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    - match:
        severity: critical
      receiver: incident-manager-critical
      group_wait: 10s
      repeat_interval: 10m

receivers:
  - name: 'incident-manager'
    webhook_configs:
      - url: 'http://incident-manager:5001/webhook/alert'
        send_resolved: true

  - name: 'incident-manager-critical'
    webhook_configs:
      - url: 'http://incident-manager:5001/webhook/alert'
        send_resolved: true
```

3. **Создай Runbook template и примеры**:

markdown

````markdown
# Runbook: High Memory Usage

## Overview
This runbook guides you through investigating and resolving high memory usage alerts.

**When to use:** When `HighMemoryUsage` alert fires

**Severity:** SEV-2 (High)

## Symptoms
- Alert: `HighMemoryUsage` firing
- Memory usage >85% for >10 minutes
- Application may be slow or unresponsive
- Potential OOM kills

## Impact
- **Users:** Slow application response times
- **Services:** Risk of OOM killer terminating processes
- **Business:** Degraded user experience, potential data loss

## Investigation

### 1. Check Current Memory Usage
```bash
# On the affected server
free -h
top -o %MEM

# In Kubernetes
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
```

**Dashboard:** http://grafana/d/memory-usage

**Key Metrics:**
- `node_memory_MemAvailable_bytes`
- `container_memory_working_set_bytes`

### 2. Identify Memory Hogs
```bash
# Top processes by memory
ps aux --sort=-%mem | head -20

# In Kubernetes
kubectl describe node 
kubectl get pods --all-namespaces -o wide | grep 
```

### 3. Check for Memory Leaks
```bash
# Application logs
tail -f /var/log/app.log | grep -i "memory\|heap"

# Java heap dump (if Java app)
jmap -dump:format=b,file=heap.bin 

# Python memory profiling
py-spy top --pid 
```

### 4. Review Recent Changes
- Check recent deployments: `kubectl rollout history deployment/app`
- Check git commits for last 24h
- Review configuration changes

## Resolution Steps

### Quick Fixes (Choose based on situation)

**Option A: Restart Service (if safe)**
```bash
# Kubernetes
kubectl rollout restart deployment/app

# Systemd
sudo systemctl restart myapp
```

**Option B: Scale Horizontally**
```bash
# Add more replicas
kubectl scale deployment/app --replicas=5

# HPA will auto-scale if configured
kubectl get hpa
```

**Option C: Increase Memory Limits**
```bash
# Edit deployment
kubectl edit deployment/app
# Increase resources.limits.memory

# Apply changes
kubectl apply -f deployment.yaml
```

**Option D: Clear Caches**
```bash
# Redis
redis-cli FLUSHDB

# Application cache (if exists)
curl -X POST http://localhost:8080/api/cache/clear
```

### Long-term Solutions

1. **Optimize Application:**
   - Review code for memory leaks
   - Optimize database queries
   - Implement pagination
   - Use connection pooling

2. **Add Memory Limits:**
   - Set proper resource requests/limits
   - Configure OOM score adjustments
   - Implement graceful degradation

3. **Monitoring Improvements:**
   - Add memory profiling
   - Track memory growth rate
   - Alert on memory leak patterns

## Escalation

**Escalate if:**
- Memory usage >95% and growing
- OOM kills occurring
- Service completely unresponsive
- Unable to identify root cause within 30 minutes

**Contact:**
- Primary: Platform Team (#platform-oncall)
- Secondary: SRE Lead (@sre-lead)
- Emergency: CTO (@cto)

## Prevention

1. **Set appropriate memory limits**
2. **Implement health checks with memory thresholds**
3. **Regular capacity planning reviews**
4. **Load testing before major releases**
5. **Memory profiling in staging**

## Post-Incident

- [ ] Update this runbook with new learnings
- [ ] Create post-mortem if SEV-1/SEV-2
- [ ] Implement action items
- [ ] Update monitoring/alerts if needed

## Related

- **Code:** https://github.com/company/app
- **Dashboard:** http://grafana/d/memory-dashboard
- **Previous Incidents:** 
  - INC-1234 (2024-12-15)
  - INC-5678 (2024-11-20)
- **Related Runbooks:**
  - High CPU Usage
  - OOM Killed Containers

---
**Last Updated:** 2025-01-15
**Maintained By:** Platform Team
````

4. **Создай Post-Mortem template**:

markdown

```markdown
# Post-Mortem: [Incident Title]

**Date:** 2025-01-15
**Authors:** @engineer1, @engineer2
**Status:** Draft | Review | Final
**Severity:** SEV-1

## Executive Summary
Brief (2-3 sentences) description of what happened, impact, and resolution.

## Impact
- **Duration:** 2 hours 15 minutes (14:30 - 16:45 UTC)
- **Users Affected:** ~15,000 (10% of daily active users)
- **Services Affected:** Payment processing, checkout flow
- **Revenue Impact:** ~$50,000 in lost transactions
- **SLO Impact:** 
  - Availability SLO: 99.95% → 99.87% (burned 8.3% of monthly error budget)
  - Latency SLO: p95 < 500ms → p95 1200ms

## Timeline (All times UTC)

**Detection:**
- 14:30 - Alert `HighErrorRate` fires
- 14:32 - On-call engineer acknowledges
- 14:35 - Incident declared SEV-2

**Investigation:**
- 14:38 - Checked application logs, found database connection errors
- 14:42 - Database team engaged
- 14:45 - Discovered database connection pool exhaustion
- 14:48 - Elevated to SEV-1 due to increasing impact

**Mitigation:**
- 14:50 - Attempted to increase connection pool size (failed - config error)
- 15:05 - Restarted application pods to reset connections
- 15:15 - Partial recovery, but still errors
- 15:30 - Identified root cause: database max_connections limit
- 15:35 - Temporarily increased max_connections on database
- 15:45 - Full recovery confirmed

**Resolution:**
- 16:00 - Monitoring confirmed stable
- 16:15 - Implemented permanent fix with proper connection pooling
- 16:45 - Incident closed

## Root Cause
The application was configured with a connection pool size of 20 connections per instance. During a traffic spike (2x normal load), we scaled from 10 to 30 instances, resulting in 600 total connections (30 instances × 20 connections).

The database `max_connections` was set to 500, which was exceeded. This caused new connection attempts to fail, resulting in 5xx errors for ~10% of requests.

**Contributing Factors:**
1. No connection pooling strategy across instances
2. Database `max_connections` not scaled with application
3. No alerts on database connection count
4. Traffic spike from marketing campaign (not communicated)

## What Went Well
- ✅ Alert fired within 2 minutes
- ✅ On-call engineer responded quickly
- ✅ Good communication in incident channel
- ✅ Database team responded within 10 minutes
- ✅ Rollback plan was available (though not needed)

## What Didn't Go Well
- ❌ Initial mitigation attempt failed due to config error
- ❌ Took 60 minutes to identify root cause
- ❌ No coordination with marketing on traffic spike
- ❌ No automated remediation
- ❌ Runbook was outdated

## Action Items

| Action | Owner | Priority | Due Date | Status |
|--------|-------|----------|----------|--------|
| Implement connection pooling with shared limits | @backend-team | P0 | 2025-01-20 | In Progress |
| Add alert on database connection count | @sre-team | P0 | 2025-01-18 | Done |
| Update database max_connections to 2000 | @dba-team | P0 | 2025-01-17 | Done |
| Create cross-team communication process for campaigns | @product | P1 | 2025-01-25 | Not Started |
| Add load testing for connection pooling scenarios | @qa-team | P1 | 2025-01-30 | Not Started |
| Update runbook with database connection issues | @sre-team | P2 | 2025-01-22 | In Progress |
| Implement circuit breaker for database connections | @backend-team | P2 | 2025-02-15 | Not Started |

## Lessons Learned

### Technical
1. Connection pooling needs to be coordinated across all instances
2. Database limits should scale with application instances
3. Need better observability into connection pool usage

### Process
1. Marketing campaigns should trigger load testing
2. Cross-team communication channels need improvement
3. Runbooks need regular reviews and updates

### Monitoring
1. Need alerts on database connection metrics
2. Should monitor connection pool utilization
3. Traffic forecasting would help prevent surprises

## Appendix

### Metrics & Graphs
![Error Rate](link-to-graph)
![Database Connections](link-to-graph)
![Response Time](link-to-graph)

### Related Documents
- [Runbook: Database Connection Issues](link)
- [Architecture: Database Connection Pooling](link)
- [Previous Incident: INC-1234](link)

### Customer Communication
**Status Page Updates:**
- 14:45 - "Investigating elevated error rates"
- 15:15 - "Identified issue, implementing fix"
- 16:00 - "Issue resolved, monitoring"

**Customer Support:**
- 150 support tickets created
- Average response time: 15 minutes
- 95% resolved within 2 hours

---
**Review Meeting:** 2025-01-18 @ 15:00 UTC
**Attendees:** Engineering, Product, Support, Leadership
```

5. **Создай On-Call dashboard в Grafana**:

json

```json
{
  "dashboard": {
    "title": "On-Call Dashboard",
    "panels": [
      {
        "title": "Open Incidents",
        "targets": [{
          "expr": "count(ALERTS{alertstate=\"firing\"})"
        }]
      },
      {
        "title": "Incident Rate (24h)",
        "targets": [{
          "expr": "increase(incident_created_total[24h])"
        }]
      },
      {
        "title": "MTTR (Mean Time To Resolve)",
        "targets": [{
          "expr": "avg(incident_duration_seconds)"
        }]
      },
      {
        "title": "On-Call Response Time",
        "targets": [{
          "expr": "histogram_quantile(0.95, incident_acknowledge_time_seconds)"
        }]
      },
      {
        "title": "Incidents by Severity",
        "targets": [{
          "expr": "sum by(severity) (incident_created_total)"
        }]
      },
      {
        "title": "Error Budget Remaining",
        "targets": [{
          "expr": "(1 - (sum(rate(http_requests_total{status=~\"5..\"}[30d])) / sum(rate(http_requests_total[30d])))) * 100"
        }]
      }
    ]
  }
}
```

### 🚀 Бонус (новое)

**1. Automated Incident Response (Self-Healing)**:

python

```python
# auto_remediation.py
from prometheus_client import Counter, Histogram
import subprocess
import time

remediation_attempts = Counter('auto_remediation_attempts_total',
                               'Total auto-remediation attempts',
                               ['alert', 'action', 'result'])
remediation_duration = Histogram('auto_remediation_duration_seconds',
                                 'Duration of auto-remediation',
                                 ['alert', 'action'])

class AutoRemediation:
    """Автоматическое устранение проблем"""
    
    def __init__(self):
        self.actions = {
            'HighMemoryUsage': self.handle_high_memory,
            'HighCPUUsage': self.handle_high_cpu,
            'PodCrashLooping': self.handle_crash_loop,
            'DiskSpaceLow': self.handle_disk_space,
        }
    
    def handle_alert(self, alert_name, labels):
        """Обработка алерта"""
        if alert_name not in self.actions:
            print(f"No auto-remediation for {alert_name}")
            return False
        
        print(f"🤖 Auto-remediation triggered for {alert_name}")
        
        with remediation_duration.labels(alert=alert_name, action='total').time():
            try:
                success = self.actions[alert_name](labels)
                
                result = 'success' if success else 'failure'
                remediation_attempts.labels(
                    alert=alert_name,
                    action='remediate',
                    result=result
                ).inc()
                
                return success
                
            except Exception as e:
                print(f"Auto-remediation failed: {e}")
                remediation_attempts.labels(
                    alert=alert_name,
                    action='remediate',
                    result='error'
                ).inc()
                return False
    
    def handle_high_memory(self, labels):
        """High memory: restart service"""
        instance = labels.get('instance')
        pod = labels.get('pod')
        
        if pod:
            # Kubernetes pod restart
            print(f"Restarting pod {pod}")
            result = subprocess.run(
                ['kubectl', 'delete', 'pod', pod, '--grace-period=30'],
                capture_output=True
            )
            return result.returncode == 0
        else:
            # Systemd service restart
            print(f"Restarting serviceon {instance}") # Only in safe scenarios! return True

def handle_high_cpu(self, labels):
    """High CPU: scale up"""
    deployment = labels.get('deployment')
    
    if deployment:
        print(f"Scaling up {deployment}")
        result = subprocess.run(
            ['kubectl', 'scale', 'deployment', deployment, '--replicas=+2'],
            capture_output=True
        )
        return result.returncode == 0
    return False

def handle_crash_loop(self, labels):
    """Crash loop: check recent deployments and rollback if needed"""
    deployment = labels.get('deployment')
    
    if deployment:
        # Check if recent deployment (last 15 min)
        print(f"Checking recent deployments for {deployment}")
        
        result = subprocess.run(
            ['kubectl', 'rollout', 'history', 'deployment', deployment],
            capture_output=True,
            text=True
        )
        
        # If deployment is recent, rollback
        # (simplified logic)
        print(f"Rolling back {deployment}")
        rollback = subprocess.run(
            ['kubectl', 'rollout', 'undo', 'deployment', deployment],
            capture_output=True
        )
        
        return rollback.returncode == 0
    return False

def handle_disk_space(self, labels):
    """Disk space: clean up old logs and temp files"""
    instance = labels.get('instance')
    
    print(f"Cleaning up disk space on {instance}")
    
    # Clean up old logs
    subprocess.run([
        'ssh', instance,
        'find /var/log -name "*.log.*" -mtime +7 -delete'
    ])
    
    # Clean up temp files
    subprocess.run([
        'ssh', instance,
        'find /tmp -mtime +7 -delete'
    ])
    
    # Clean up Docker if available
    subprocess.run([
        'ssh', instance,
        'docker system prune -af --volumes'
    ])
    
    return True


# Использование

remediation = AutoRemediation()

def on_alert(alert_name, labels):
    # Try auto-remediation first
    if remediation.handle_alert(alert_name, labels):
        print(f"✅ Auto-remediation successful for {alert_name}")

        # Still create incident for tracking
        create_incident(
            alert_name,
            labels,
            auto_fixed=True
        )
    else:
        print(f"❌ Auto-remediation failed, escalating")

        # Create incident and page on-call
        create_incident(
            alert_name,
            labels,
            auto_fixed=False
        )
        page_oncall(alert_name, labels)


```

**2. Chaos Engineering для тестирования мониторинга**:
```python
# chaos_testing.py
import random
import subprocess
import time
from datetime import datetime

class ChaosMonkey:
    """Chaos Engineering для тестирования monitoring и incident response"""
    
    def __init__(self):
        self.experiments = [
            self.kill_random_pod,
            self.increase_latency,
            self.fill_disk,
            self.exhaust_connections,
            self.cpu_spike
        ]
    
    def run_experiment(self):
        """Запуск случайного эксперимента"""
        experiment = random.choice(self.experiments)
        
        print(f"\n{'='*60}")
        print(f"🐵 Chaos Experiment: {experiment.__name__}")
        print(f"Time: {datetime.now()}")
        print(f"{'='*60}\n")
        
        # Record start
        start_time = time.time()
        
        # Run experiment
        experiment()
        
        # Wait and observe
        print("⏳ Waiting 5 minutes to observe effects...")
        time.sleep(300)
        
        # Check if monitoring detected it
        duration = time.time() - start_time
        self.verify_detection(experiment.__name__, duration)
    
    def kill_random_pod(self):
        """Убить случайный pod"""
        # Get all pods
        result = subprocess.run(
            ['kubectl', 'get', 'pods', '-o', 'jsonpath={.items[*].metadata.name}'],
            capture_output=True,
            text=True
        )
        
        pods = result.stdout.split()
        if pods:
            victim = random.choice(pods)
            print(f"💀 Killing pod: {victim}")
            subprocess.run(['kubectl', 'delete', 'pod', victim, '--force'])
    
    def increase_latency(self):
        """Добавить сетевую задержку"""
        print(f"🐌 Adding 200ms latency to all pods")
        # Using tc (traffic control) - requires privileges
        subprocess.run([
            'kubectl', 'exec', '-it', 'pod-name', '--',
            'tc', 'qdisc', 'add', 'dev', 'eth0', 'root', 'netem', 'delay', '200ms'
        ])
    
    def fill_disk(self):
        """Заполнить диск"""
        print(f"💾 Filling disk to 90%")
        # Create large file
        subprocess.run([
            'kubectl', 'exec', 'pod-name', '--',
            'dd', 'if=/dev/zero', 'of=/tmp/largefile', 'bs=1M', 'count=1000'
        ])
    
    def exhaust_connections(self):
        """Исчерпать database connections"""
        print(f"🔌 Opening 100 database connections and holding them")
        # Python script to open connections
        script = '''
import psycopg2
import time
conns = []
for i in range(100):
    conns.append(psycopg2.connect("dbname=mydb"))
time.sleep(600)
'''
        subprocess.run(['python', '-c', script])
    
    def cpu_spike(self):
        """CPU spike"""
        print(f"🔥 Creating CPU spike")
        subprocess.run([
            'kubectl', 'exec', 'pod-name', '--',
            'stress-ng', '--cpu', '4', '--timeout', '300s'
        ])
    
    def verify_detection(self, experiment, duration):
        """Проверить, обнаружил ли мониторинг проблему"""
        print(f"\n{'='*60}")
        print(f"Verification:")
        print(f"{'='*60}")
        
        # Check for alerts
        result = subprocess.run(
            ['curl', '-s', 'http://localhost:9090/api/v1/alerts'],
            capture_output=True,
            text=True
        )
        
        import json
        alerts = json.loads(result.stdout)
        
        active_alerts = [
            a['labels']['alertname'] 
            for a in alerts.get('data', {}).get('alerts', [])
            if a['state'] == 'firing'
        ]
        
        if active_alerts:
            print(f"✅ Monitoring detected the issue!")
            print(f"   Active alerts: {', '.join(active_alerts)}")
        else:
            print(f"❌ Monitoring did NOT detect the issue!")
            print(f"   This is a problem - improve monitoring!")
        
        # Check incident manager
        result = subprocess.run(
            ['curl', '-s', 'http://localhost:5001/incidents?status=open'],
            capture_output=True,
            text=True
        )
        
        incidents = json.loads(result.stdout)
        if incidents:
            print(f"✅ Incident created: {len(incidents)} open incidents")
        else:
            print(f"❌ No incident created")
        
        print(f"\nExperiment duration: {duration:.0f}s")

# Schedule chaos experiments
if __name__ == '__main__':
    monkey = ChaosMonkey()
    
    print("🐵 Chaos Monkey Started")
    print("Running experiments every hour...")
    
    while True:
        # Run during business hours only (optional)
        hour = datetime.now().hour
        if 9 <= hour <= 17:  # 9 AM to 5 PM
            monkey.run_experiment()
        else:
            print(f"⏰ Outside business hours, skipping")
        
        time.sleep(3600)  # 1 hour
```

**3. Alert Quality Dashboard**:

Создай метрики для оценки качества алертов:
```python
# alert_quality_metrics.py
from prometheus_client import Counter, Gauge, Histogram

# Метрики качества алертов
alert_fired = Counter('alert_fired_total', 'Total alerts fired', ['alert', 'severity'])
alert_acknowledged = Counter('alert_acknowledged_total', 'Alerts acknowledged', ['alert'])
alert_false_positive = Counter('alert_false_positive_total', 'False positive alerts', ['alert'])
alert_actionable = Counter('alert_actionable_total', 'Alerts that required action', ['alert'])

time_to_acknowledge = Histogram('alert_time_to_acknowledge_seconds',
                                'Time to acknowledge alert',
                                ['alert'])
time_to_resolve = Histogram('alert_time_to_resolve_seconds',
                            'Time to resolve alert',
                            ['alert'])

# Alert Quality Score
alert_quality_score = Gauge('alert_quality_score',
                           'Alert quality score (0-100)',
                           ['alert'])

def calculate_alert_quality(alert_name):
    """
    Расчет качества алерта:
    - Signal/Noise ratio
    - False positive rate
    - Actionability rate
    - Response time
    """
    total = alert_fired.labels(alert=alert_name)._value.get()
    false_pos = alert_false_positive.labels(alert=alert_name)._value.get()
    actionable = alert_actionable.labels(alert=alert_name)._value.get()
    
    if total == 0:
        return 100
    
    # Metrics
    false_positive_rate = false_pos / total
    actionability_rate = actionable / total
    
    # Score calculation (0-100)
    score = (
        (1 - false_positive_rate) * 50 +  # 50% weight on low false positives
        actionability_rate * 50             # 50% weight on actionability
    )
    
    alert_quality_score.labels(alert=alert_name).set(score)
    
    return score
```

Grafana dashboard для Alert Quality:

```
Panel 1: Alert Quality Score по всем алертам 
Panel 2: False Positive Rate 
Panel 3: Time to Acknowledge (p50, p95) 
Panel 4: Time to Resolve (p50, p95) 
Panel 5: Alerts по категориям (actionable vs noise)
```


---

**Итоговый чеклист модуля 10:**
- ✅ Настроил Incident Management систему
- ✅ Создал Runbook templates
- ✅ Написал Post-Mortem template
- ✅ Настроил On-Call dashboard
- ✅ Реализовал Auto-Remediation
- ✅ Добавил Chaos Engineering
- ✅ Создал Alert Quality метрики

**Модуль 10 завершен!** 

# Мониторинг для DevOps: ФИНАЛЬНЫЙ ПРОЕКТ (60 минут)

## 🎯 Задача: Production-Ready Monitoring Stack для E-Commerce приложения

Разверни полноценную систему мониторинга для трехуровневого e-commerce приложения с метриками, логами, трейсингом, алертингом и incident management.

---

## 📋 Архитектура приложения

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │ Ingress │
                    │ (Nginx) │
                    └────┬────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼─────┐    ┌────▼─────┐    ┌────▼─────┐
   │Frontend  │    │   API    │    │  Admin   │
   │ (React)  │    │(Node.js) │    │ (React)  │
   └──────────┘    └────┬─────┘    └──────────┘
                        │
           ┌────────────┼────────────┐
           │            │            │
      ┌────▼─────┐ ┌───▼────┐  ┌───▼────────┐
      │ Product  │ │Payment │  │ Inventory  │
      │ Service  │ │Service │  │  Service   │
      │(Python)  │ │(Go)    │  │  (Java)    │
      └────┬─────┘ └───┬────┘  └─────┬──────┘
           │           │              │
           └───────────┼──────────────┘
                       │
              ┌────────┼────────┐
              │        │        │
         ┌────▼───┐ ┌──▼───┐ ┌─▼─────┐
         │PostgreSQL│ │Redis │ │RabbitMQ│
         │          │ │      │ │        │
         └──────────┘ └──────┘ └────────┘
```

---

## 📦 Что будет включено

### Мониторинг Stack:

- ✅ **Prometheus** - метрики
- ✅ **Grafana** - визуализация
- ✅ **Loki + Promtail** - логи
- ✅ **Jaeger** - трейсинг
- ✅ **Alertmanager** - алертинг
- ✅ **Blackbox Exporter** - synthetic monitoring
- ✅ **Node Exporter** - хост метрики
- ✅ **cAdvisor** - контейнер метрики
- ✅ **Incident Manager** - управление инцидентами

### Дашборды:

- 📊 Business Metrics Dashboard
- 📊 Application Performance Dashboard
- 📊 Infrastructure Dashboard
- 📊 SLO/Error Budget Dashboard
- 📊 On-Call Dashboard

### Алерты:

- 🚨 Критичные (страница на-call)
- ⚠️ Warning (создание тикета)
- 📝 Info (только в дашборде)

---

## 🏗️ Структура проекта

```
monitoring-final-project/
├── docker-compose.yml
├── .env
├── README.md
│
├── apps/
│   ├── frontend/
│   │   ├── Dockerfile
│   │   ├── src/
│   │   └── nginx.conf
│   ├── api/
│   │   ├── Dockerfile
│   │   ├── server.js
│   │   └── package.json
│   ├── product-service/
│   │   ├── Dockerfile
│   │   ├── app.py
│   │   └── requirements.txt
│   ├── payment-service/
│   │   ├── Dockerfile
│   │   └── main.go
│   └── inventory-service/
│       ├── Dockerfile
│       ├── src/
│       └── pom.xml
│
├── monitoring/
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   ├── alerts/
│   │   │   ├── application.yml
│   │   │   ├── infrastructure.yml
│   │   │   ├── business.yml
│   │   │   └── slo.yml
│   │   └── rules/
│   │       └── recording_rules.yml
│   │
│   ├── grafana/
│   │   ├── provisioning/
│   │   │   ├── datasources/
│   │   │   │   ├── prometheus.yml
│   │   │   │   ├── loki.yml
│   │   │   │   └── jaeger.yml
│   │   │   └── dashboards/
│   │   │       └── dashboard.yml
│   │   └── dashboards/
│   │       ├── business-metrics.json
│   │       ├── application-performance.json
│   │       ├── infrastructure.json
│   │       ├── slo-dashboard.json
│   │       └── oncall-dashboard.json
│   │
│   ├── loki/
│   │   └── loki-config.yml
│   │
│   ├── promtail/
│   │   └── promtail-config.yml
│   │
│   ├── alertmanager/
│   │   ├── alertmanager.yml
│   │   └── templates/
│   │       ├── email.tmpl
│   │       └── slack.tmpl
│   │
│   ├── blackbox/
│   │   └── blackbox.yml
│   │
│   └── incident-manager/
│       ├── Dockerfile
│       ├── app.py
│       └── requirements.txt
│
├── scripts/
│   ├── setup.sh
│   ├── load-test.sh
│   ├── chaos-test.sh
│   └── backup-dashboards.sh
│
└── docs/
    ├── RUNBOOKS.md
    ├── ARCHITECTURE.md
    ├── ALERTS.md
    └── POST_MORTEM_TEMPLATE.md
```

---

## 🚀 Шаг 1: Базовая инфраструктура (15 минут)

### 1.1 Создай docker-compose.yml

yaml

```yaml
version: '3.8'

networks:
  monitoring:
    driver: bridge
  app:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  postgres-data:
  redis-data:
  elasticsearch-data:

services:
  # ==================== DATABASES ====================
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret123
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret123
    networks:
      - app
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ==================== APPLICATION ====================
  frontend:
    build: ./apps/frontend
    container_name: frontend
    ports:
      - "3000:80"
    environment:
      - API_URL=http://api:4000
    networks:
      - app
      - monitoring
    labels:
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=9113"
    depends_on:
      - api

  api:
    build: ./apps/api
    container_name: api
    ports:
      - "4000:4000"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=ecommerce
      - DB_USER=admin
      - DB_PASSWORD=secret123
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - RABBITMQ_HOST=rabbitmq
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
    networks:
      - app
      - monitoring
    depends_on:
      - postgres
      - redis
      - rabbitmq
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  product-service:
    build: ./apps/product-service
    container_name: product-service
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
      - JAEGER_AGENT_HOST=jaeger
    networks:
      - app
      - monitoring
    depends_on:
      - postgres
      - redis

  payment-service:
    build: ./apps/payment-service
    container_name: payment-service
    ports:
      - "6000:6000"
    environment:
      - DB_HOST=postgres
      - JAEGER_AGENT_HOST=jaeger
    networks:
      - app
      - monitoring
    depends_on:
      - postgres

  inventory-service:
    build: ./apps/inventory-service
    container_name: inventory-service
    ports:
      - "7000:7000"
    environment:
      - DB_HOST=postgres
      - RABBITMQ_HOST=rabbitmq
      - JAEGER_AGENT_HOST=jaeger
    networks:
      - app
      - monitoring
    depends_on:
      - postgres
      - rabbitmq

  # ==================== MONITORING ====================
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
    networks:
      - monitoring
    depends_on:
      - prometheus
      - loki
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./monitoring/loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./monitoring/promtail/promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring
    depends_on:
      - loki
    restart: unless-stopped

  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "5775:5775/udp"
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
      - "16686:16686"
      - "14268:14268"
      - "14250:14250"
      - "9411:9411"
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - monitoring
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring
    restart: unless-stopped

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    ports:
      - "9115:9115"
    volumes:
      - ./monitoring/blackbox/blackbox.yml:/etc/blackbox_exporter/config.yml
    command:
      - '--config.file=/etc/blackbox_exporter/config.yml'
    networks:
      - monitoring
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    pid: host
    volumes:
      - '/:/host:ro,rslave'
    networks:
      - monitoring
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    networks:
      - monitoring
    restart: unless-stopped

  incident-manager:
    build: ./monitoring/incident-manager
    container_name: incident-manager
    ports:
      - "5001:5001"
    volumes:
      - ./monitoring/incident-manager/incidents.db:/app/incidents.db
    environment:
      - SLACK_WEBHOOK_URL=${SLACK_WEBHOOK_URL}
    networks:
      - monitoring
    restart: unless-stopped

  # ==================== LOAD GENERATOR ====================
  load-generator:
    image: williamyeh/hey:latest
    container_name: load-generator
    command: >
      sh -c "while true; do
        hey -z 60s -c 10 -q 5 http://api:4000/api/products;
        hey -z 60s -c 5 -q 2 http://api:4000/api/orders;
        sleep 60;
      done"
    networks:
      - app
    depends_on:
      - api
```

### 1.2 Создай .env файл

bash

```bash
# .env
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
PAGERDUTY_KEY=your-pagerduty-key

# Database
POSTGRES_PASSWORD=secret123
REDIS_PASSWORD=

# Grafana
GF_SECURITY_ADMIN_PASSWORD=admin123

# Application
NODE_ENV=production
```

---

## 🔧 Шаг 2: Приложения с инструментацией (20 минут)

### 2.1 API Service (Node.js)

`apps/api/server.js`:

javascript

```javascript
const express = require('express');
const promClient = require('prom-client');
const opentelemetry = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const winston = require('winston');
const { Pool } = require('pg');
const Redis = require('ioredis');

// ========== TRACING SETUP ==========
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'api-service',
  }),
});

const exporter = new JaegerExporter({
  endpoint: `http://${process.env.JAEGER_AGENT_HOST}:14268/api/traces`,
});

provider.addSpanProcessor(
  new opentelemetry.sdk.trace.node.BatchSpanProcessor(exporter)
);

provider.register();
const tracer = opentelemetry.trace.getTracer('api-service');

// ========== LOGGING SETUP ==========
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: '/var/log/api.log' })
  ]
});

// ========== METRICS SETUP ==========
const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const businessMetrics = {
  orders: new promClient.Counter({
    name: 'orders_total',
    help: 'Total number of orders',
    labelNames: ['status']
  }),
  revenue: new promClient.Counter({
    name: 'revenue_total',
    help: 'Total revenue',
    labelNames: ['currency']
  }),
  cartValue: new promClient.Histogram({
    name: 'cart_value_dollars',
    help: 'Shopping cart value in dollars',
    buckets: [10, 50, 100, 200, 500, 1000]
  })
};

register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);
register.registerMetric(businessMetrics.orders);
register.registerMetric(businessMetrics.revenue);
register.registerMetric(businessMetrics.cartValue);

// ========== DATABASE & CACHE ==========
const db = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20
});

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT
});

// ========== EXPRESS APP ==========
const app = express();
app.use(express.json());

// Middleware: Request tracking
app.use((req, res, next) => {
  const start = Date.now();
  const span = tracer.startSpan(`${req.method} ${req.path}`);
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestDuration.labels(req.method, req.path, res.statusCode).observe(duration);
    httpRequestTotal.labels(req.method, req.path, res.statusCode).inc();
    
    logger.info('HTTP Request', {
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration: duration,
      traceId: span.spanContext().traceId
    });
    
    span.end();
  });
  
  next();
});

// ========== ENDPOINTS ==========

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Get products
app.get('/api/products', async (req, res) => {
  const span = tracer.startSpan('get_products');
  
  try {
    // Check cache
    const cached = await redis.get('products:all');
    if (cached) {
      span.setAttribute('cache', 'hit');
      span.end();
      return res.json(JSON.parse(cached));
    }
    
    span.setAttribute('cache', 'miss');
    
    // Query database
    const result = await db.query('SELECT * FROM products LIMIT 100');
    const products = result.rows;
    
    // Cache for 5 minutes
    await redis.setex('products:all', 300, JSON.stringify(products));
    
    span.end();
    res.json(products);
    
  } catch (error) {
    span.recordException(error);
    span.end();
    logger.error('Error fetching products', { error: error.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create order
app.post('/api/orders', async (req, res) => {
  const span = tracer.startSpan('create_order');
  const { userId, items, total } = req.body;
  
  try {
    // Validate
    if (!userId || !items || !total) {
      return res.status(400).json({ error: 'Missing required fields' });
    }
    
    // Create order in database
    const result = await db.query(
      'INSERT INTO orders (user_id, total, status, created_at) VALUES ($1, $2, $3, NOW()) RETURNING *',
      [userId, total, 'pending']
    );
    
    const order = result.rows[0];
    
    // Business metrics
    businessMetrics.orders.labels('created').inc();
    businessMetrics.revenue.labels('USD').inc(total);
    businessMetrics.cartValue.observe(total);
    
    // Simulate payment processing (call payment service)
    // ... 
    
    span.setAttribute('order.id', order.id);
    span.setAttribute('order.total', total);
    span.end();
    
    logger.info('Order created', {
      orderId: order.id,
      userId: userId,
      total: total
    });
    
    res.status(201).json(order);
    
  } catch (error) {
    businessMetrics.orders.labels('failed').inc();
    span.recordException(error);
    span.end();
    logger.error('Error creating order', { error: error.message });
    res.status(500).json({ error: 'Order creation failed' });
  }
});

// Simulate slow endpoint
app.get('/api/slow', async (req, res) => {
  await new Promise(resolve => setTimeout(resolve, 3000));
  res.json({ message: 'This was slow' });
});

// Simulate error endpoint
app.get('/api/error', (req, res) => {
  if (Math.random() < 0.3) {
    logger.error('Random error occurred');
    return res.status(500).json({ error: 'Something went wrong' });
  }
  res.json({ message: 'Success' });
});

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => {
  logger.info(`API Server started on port ${PORT}`);
  console.log(`🚀 API Server running on http://localhost:${PORT}`);
  console.log(`📊 Metrics available at http://localhost:${PORT}/metrics`);
});
```

`apps/api/Dockerfile`:

dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy app
COPY . .

EXPOSE 4000

CMD ["node", "server.js"]
```

`apps/api/package.json`:

json

```json
{
  "name": "ecommerce-api",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^15.0.0",
    "@opentelemetry/api": "^1.7.0",
    "@opentelemetry/sdk-trace-node": "^1.18.0",
    "@opentelemetry/exporter-jaeger": "^1.18.0",
    "@opentelemetry/resources": "^1.18.0",
    "@opentelemetry/semantic-conventions": "^1.18.0",
    "winston": "^3.11.0",
    "pg": "^8.11.3",
    "ioredis": "^5.3.2"
  }
}
```

### 2.2 Product Service (Python/Flask)

`apps/product-service/app.py`:

python

```python
from flask import Flask, jsonify, request
from prometheus_flask_exporter import PrometheusMetrics
from elasticapm.contrib.flask import ElasticAPM
import logging
import psycopg2
from redis import Redis
import time
import random

app = Flask(__name__)

# ========== METRICS ==========
metrics = PrometheusMetrics(app)
metrics.info('app_info', 'Product Service', version='1.0.0')

# ========== LOGGING ==========
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# ========== DATABASE & CACHE ==========
def get_db():
    return psycopg2.connect(
        host='postgres',
        database='ecommerce',
        user='admin',
        password='secret123'
    )

redis_client = Redis(host='redis', port=6379, decode_responses=True)

# ========== ENDPOINTS ==========

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/api/products')
@metrics.summary('products_list_duration_seconds', 'Time to list products')
def list_products():
    try:
        # Check cache
        cached = redis_client.get('products:all')
        if cached:
            logger.info('Products served from cache')
            return jsonify({'products': eval(cached), 'source': 'cache'})
        
        # Query database
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute('SELECT id, name, price, stock FROM products LIMIT 100')
        
        products = [
            {'id': row[0], 'name': row[1], 'price': float(row[2]), 'stock': row[3]}
            for row in cursor.fetchall()
        ]
        
        cursor.close()
        conn.close()
        
        # Cache for 5 minutes
        redis_client.setex('products:all', 300, str(products))
        
        logger.info(f'Listed {len(products)} products from database')
        return jsonify({'products': products, 'source': 'database'})
        
    except Exception as e:
        logger.error(f'Error listing products: {e}')
        return jsonify({'error': str(e)}), 500

@app.route('/api/products/<int:product_id>')
def get_product(product_id):
    try:
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute(
            'SELECT id, name, price, stock, description FROM products WHERE id = %s',
            (product_id,)
        )
        
        row = cursor.fetchone()
        if not row:
            return jsonify({'error': 'Product not found'}), 404
        
        product = {
            'id': row[0],
            'name': row[1],
            'price': float(row[2]),
            'stock': row[3],
            'description': row[4]
        }
        
        cursor.close()
        conn.close()
        
        return jsonify(product)
        
    except Exception as e:
        logger.error(f'Error getting product {product_id}: {e}')
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

---

## 📊 Шаг 3: Prometheus конфигурация (10 минут)

`monitoring/prometheus/prometheus.yml`:

yaml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'ecommerce-prod'
    environment: 'production'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Load rules
rule_files:
  - '/etc/prometheus/alerts/*.yml'
  - '/etc/prometheus/rules/*.yml'

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Application services
  - job_name: 'api-service'
    static_configs:
      - targets: ['api:4000']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'api-service'

  - job_name: 'product-service'
    static_configs:
      - targets: ['product-service:5000']

	- job_name: 'payment-service' static_configs:
	    - targets: ['payment-service:6000']
	- job_name: 'inventory-service' static_configs:
	    - targets: ['inventory-service:7000']
	
	# Infrastructure
	
	- job_name: 'node-exporter' static_configs:
	    - targets: ['node-exporter:9100']
	- job_name: 'cadvisor' static_configs:
	    - targets: ['cadvisor:8080']
	
	# Databases
	
	- job_name: 'postgres' static_configs:
	    - targets: ['postgres:5432']
	- job_name: 'redis' static_configs:
	    - targets: ['redis:6379']

# Blackbox exporter for synthetic monitoring

- job_name: 'blackbox-http' metrics_path: /probe params: module: [http_2xx] static_configs:
    - targets:
        - [http://frontend:80](http://frontend:80)
        - [http://api:4000/health](http://api:4000/health)
        - [http://product-service:5000/health](http://product-service:5000/health) relabel_configs:
    - source_labels: [**address**] target_label: __param_target
    - source_labels: [__param_target] target_label: instance
    - target_label: **address** replacement: blackbox-exporter:9115

```

---

## 📢 Шаг 4: Алерты и правила (10 минут)

### 4.1 Application Alerts

`monitoring/prometheus/alerts/application.yml`:

yaml

```yaml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)
            /
            sum(rate(http_requests_total[5m])) by (job)
          ) * 100 > 5
        for: 5m
        labels:
          severity: critical
          component: application
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
          runbook: "https://wiki.company.com/runbooks/high-error-rate"
          dashboard: "http://grafana:3000/d/app-performance"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          ) > 2
        for: 10m
        labels:
          severity: warning
          component: application
        annotations:
          summary: "High latency on {{ $labels.job }}"
          description: "p95 latency is {{ $value }}s (threshold: 2s)"
          impact: "Users experiencing slow response times"

      # Service down
      - alert: ServiceDown
        expr: up{job=~".*-service"} == 0
        for: 2m
        labels:
          severity: critical
          component: application
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.job }} has been unreachable for 2 minutes"
          action: "Check service logs and restart if needed"

      # Low throughput
      - alert: LowThroughput
        expr: |
          sum(rate(http_requests_total[5m])) by (job) < 1
        for: 10m
        labels:
          severity: warning
          component: application
        annotations:
          summary: "Low throughput on {{ $labels.job }}"
          description: "Request rate is {{ $value }} req/s (expected >10)"
          impact: "Possible service degradation or traffic loss"

      # Database connection pool exhausted
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          (
            sum(pg_stat_activity_count) by (instance)
            /
            pg_settings_max_connections
          ) * 100 > 90
        for: 5m
        labels:
          severity: critical
          component: database
        annotations:
          summary: "Database connection pool exhausted"
          description: "{{ $value }}% of connections in use"
          action: "Scale application or increase max_connections"

      # Redis memory high
      - alert: RedisMemoryHigh
        expr: |
          (
            redis_memory_used_bytes
            /
            redis_memory_max_bytes
          ) * 100 > 90
        for: 5m
        labels:
          severity: warning
          component: cache
        annotations:
          summary: "Redis memory usage high"
          description: "Redis using {{ $value }}% of max memory"
```

### 4.2 Business Metrics Alerts

`monitoring/prometheus/alerts/business.yml`:

yaml

```yaml
groups:
  - name: business_alerts
    interval: 1m
    rules:
      # Order creation failure rate high
      - alert: HighOrderFailureRate
        expr: |
          (
            sum(rate(orders_total{status="failed"}[5m]))
            /
            sum(rate(orders_total[5m]))
          ) * 100 > 10
        for: 5m
        labels:
          severity: critical
          component: business
          team: payments
        annotations:
          summary: "High order failure rate"
          description: "{{ $value | humanizePercentage }} of orders failing"
          impact: "Revenue loss, customer dissatisfaction"
          action: "Check payment gateway and database"

      # Revenue drop
      - alert: RevenueDrop
        expr: |
          sum(rate(revenue_total[1h])) < 1000
        for: 30m
        labels:
          severity: warning
          component: business
          team: product
        annotations:
          summary: "Revenue drop detected"
          description: "Revenue is ${{ $value }}/hour (expected >$1000)"
          impact: "Potential business issue"

      # Cart abandonment rate high
      - alert: HighCartAbandonmentRate
        expr: |
          (
            1 - (
              sum(rate(orders_total{status="completed"}[1h]))
              /
              sum(rate(cart_created_total[1h]))
            )
          ) * 100 > 80
        for: 30m
        labels:
          severity: warning
          component: business
          team: product
        annotations:
          summary: "High cart abandonment rate"
          description: "{{ $value }}% cart abandonment (threshold: 80%)"
          impact: "Lost sales opportunity"

      # No orders in last 10 minutes
      - alert: NoOrdersReceived
        expr: |
          sum(increase(orders_total[10m])) == 0
        for: 10m
        labels:
          severity: critical
          component: business
        annotations:
          summary: "No orders received"
          description: "Zero orders in last 10 minutes"
          impact: "Complete checkout flow failure"
          action: "Investigate immediately - possible outage"
```

### 4.3 Infrastructure Alerts

`monitoring/prometheus/alerts/infrastructure.yml`:

yaml

```yaml
groups:
  - name: infrastructure_alerts
    interval: 30s
    rules:
      # Instance down
      - alert: InstanceDown
        expr: up == 0
        for: 3m
        labels:
          severity: critical
          component: infrastructure
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }} on {{ $labels.instance }} is unreachable"

      # High CPU
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 15m
        labels:
          severity: warning
          component: infrastructure
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}%"

      # High memory
      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 10m
        labels:
          severity: critical
          component: infrastructure
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}%"
          action: "Check for memory leaks or scale vertically"

      # Disk space
      - alert: DiskSpaceLow
        expr: |
          (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100 > 85
        for: 10m
        labels:
          severity: warning
          component: infrastructure
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk usage is {{ $value }}%"

      # Container restarts
      - alert: ContainerRestarting
        expr: |
          rate(container_last_seen[5m]) > 0
        for: 5m
        labels:
          severity: warning
          component: container
        annotations:
          summary: "Container {{ $labels.name }} restarting"
          description: "Container has restarted in the last 5 minutes"
```

### 4.4 SLO Alerts

`monitoring/prometheus/alerts/slo.yml`:

yaml

```yaml
groups:
  - name: slo_alerts
    interval: 1m
    rules:
      # Availability SLO (99.9%)
      - alert: AvailabilitySLOViolation
        expr: |
          (
            sum(rate(http_requests_total{status_code!~"5.."}[30d]))
            /
            sum(rate(http_requests_total[30d]))
          ) < 0.999
        for: 5m
        labels:
          severity: critical
          component: slo
        annotations:
          summary: "Availability SLO violation"
          description: "Availability is {{ $value | humanizePercentage }} (target: 99.9%)"
          impact: "SLO breach - error budget exhausted"

      # Latency SLO (p95 < 500ms)
      - alert: LatencySLOViolation
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 15m
        labels:
          severity: warning
          component: slo
        annotations:
          summary: "Latency SLO violation"
          description: "p95 latency is {{ $value }}s (target: <500ms)"

      # Error budget burn rate
      - alert: ErrorBudgetBurnRateHigh
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status_code!~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            )
          ) > 0.01
        for: 10m
        labels:
          severity: critical
          component: slo
        annotations:
          summary: "High error budget burn rate"
          description: "Burning error budget at {{ $value | humanizePercentage }}/hour"
          impact: "Monthly error budget will be exhausted soon"
          action: "Stop releases and focus on reliability"
```

### 4.5 Recording Rules

`monitoring/prometheus/rules/recording_rules.yml`:

yaml

```yaml
groups:
  - name: application_recording_rules
    interval: 30s
    rules:
      # Request rate by service
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      # Error rate by service
      - record: job:http_requests_errors:rate5m
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)

      # Error percentage
      - record: job:http_requests_errors:percentage
        expr: |
          (
            job:http_requests_errors:rate5m
            /
            job:http_requests:rate5m
          ) * 100

      # Average latency
      - record: job:http_request_duration:avg
        expr: |
          sum(rate(http_request_duration_seconds_sum[5m])) by (job)
          /
          sum(rate(http_request_duration_seconds_count[5m])) by (job)

      # p95 latency
      - record: job:http_request_duration:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )

      # p99 latency
      - record: job:http_request_duration:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )

  - name: business_recording_rules
    interval: 1m
    rules:
      # Orders per minute
      - record: business:orders:rate1m
        expr: sum(rate(orders_total[1m]))

      # Revenue per hour
      - record: business:revenue:rate1h
        expr: sum(rate(revenue_total[1h]))

      # Average cart value
      - record: business:cart_value:avg
        expr: |
          sum(rate(cart_value_dollars_sum[5m]))
          /
          sum(rate(cart_value_dollars_count[5m]))

      # Conversion rate
      - record: business:conversion_rate:percentage
        expr: |
          (
            sum(rate(orders_total{status="completed"}[1h]))
            /
            sum(rate(page_views_total{page="product"}[1h]))
          ) * 100
```

---

## 🎨 Шаг 5: Grafana Dashboards (10 минут)

### 5.1 Datasource Provisioning

`monitoring/grafana/provisioning/datasources/datasources.yml`:

yaml

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: "15s"

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true

  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger:16686
    editable: true
```

### 5.2 Dashboard Provisioning

`monitoring/grafana/provisioning/dashboards/dashboard.yml`:

yaml

```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

### 5.3 Business Metrics Dashboard

`monitoring/grafana/dashboards/business-metrics.json`:

json

```json
{
  "dashboard": {
    "title": "Business Metrics",
    "tags": ["business", "ecommerce"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Revenue (Hourly)",
        "type": "graph",
        "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum(rate(revenue_total[1h]))",
            "legendFormat": "Revenue/hour",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "currencyUSD"
          }
        }
      },
      {
        "id": 2,
        "title": "Orders per Minute",
        "type": "stat",
        "gridPos": {"x": 12, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(rate(orders_total[1m]))",
            "refId": "A"
          }
        ],
        "options": {
          "colorMode": "background",
          "graphMode": "area"
        },
        "fieldConfig": {
          "defaults": {
            "unit": "reqpm"
          }
        }
      },
      {
        "id": 3,
        "title": "Conversion Rate",
        "type": "gauge",
        "gridPos": {"x": 18, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "business:conversion_rate:percentage",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"value": 0, "color": "red"},
                {"value": 2, "color": "yellow"},
                {"value": 5, "color": "green"}
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "Average Cart Value",
        "type": "stat",
        "gridPos": {"x": 12, "y": 4, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "business:cart_value:avg",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "currencyUSD"
          }
        }
      },
      {
        "id": 5,
        "title": "Order Status Distribution",
        "type": "piechart",
        "gridPos": {"x": 18, "y": 4, "w": 6, "h": 8},
        "targets": [
          {
            "expr": "sum by(status) (rate(orders_total[5m]))",
            "legendFormat": "{{status}}",
            "refId": "A"
          }
        ]
      },
      {
        "id": 6,
        "title": "Failed Orders",
        "type": "graph",
        "gridPos": {"x": 0, "y": 8, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum(rate(orders_total{status=\"failed\"}[5m]))",
            "legendFormat": "Failed orders",
            "refId": "A"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [0.5],
                "type": "gt"
              },
              "operator": {
                "type": "and"
              },
              "query": {
                "params": ["A", "5m", "now"]
              },
              "reducer": {
                "params": [],
                "type": "avg"
              },
              "type": "query"
            }
          ]
        }
      },
      {
        "id": 7,
        "title": "Top Products by Revenue",
        "type": "table",
        "gridPos": {"x": 0, "y": 16, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "topk(10, sum by(product_name) (rate(product_revenue_total[1h])))",
            "format": "table",
            "refId": "A"
          }
        ]
      }
    ],
    "templating": {
      "list": [
        {
          "name": "timeRange",
          "type": "interval",
          "query": "1h,6h,12h,24h,7d,30d"
        }
      ]
    }
  }
}
```

### 5.4 Application Performance Dashboard

`monitoring/grafana/dashboards/application-performance.json`:

json

```json
{
  "dashboard": {
    "title": "Application Performance",
    "tags": ["application", "performance"],
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum by(job) (rate(http_requests_total[5m]))",
            "legendFormat": "{{job}}",
            "refId": "A"
          }
        ]
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "graph",
        "gridPos": {"x": 12, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum by(job) (rate(http_requests_total{status_code=~\"5..\"}[5m]))",
            "legendFormat": "{{job}} errors",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {"mode": "palette-classic"},
            "custom": {
              "axisLabel": "Errors/sec",
              "fillOpacity": 10
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Response Time (p50, p95, p99)",
        "type": "graph",
        "gridPos": {"x": 0, "y": 8, "w": 24, "h": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))",
            "legendFormat": "p50 - {{job}}",
            "refId": "A"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))",
            "legendFormat": "p95 - {{job}}",
            "refId": "B"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))",
            "legendFormat": "p99 - {{job}}",
            "refId": "C"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s"
          }
        }
      },
      {
        "id": 4,
        "title": "Database Connections",
        "type": "graph",
        "gridPos": {"x": 0, "y": 16, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "pg_stat_activity_count",
            "legendFormat": "Active connections",
            "refId": "A"
          },
          {
            "expr": "pg_settings_max_connections",
            "legendFormat": "Max connections",
            "refId": "B"
          }
        ]
      },
      {
        "id": 5,
        "title": "Redis Memory Usage",
        "type": "graph",
        "gridPos": {"x": 12, "y": 16, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "redis_memory_used_bytes / 1024 / 1024",
            "legendFormat": "Used (MB)",
            "refId": "A"
          }
        ]
      }
    ]
  }
}
```

### 5.5 SLO Dashboard

`monitoring/grafana/dashboards/slo-dashboard.json`:

json

```json
{
  "dashboard": {
    "title": "SLO Dashboard",
    "tags": ["slo", "reliability"],
    "panels": [
      {
        "id": 1,
        "title": "Availability SLO (99.9% target)",
        "type": "gauge",
        "gridPos": {"x": 0, "y": 0, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "(sum(rate(http_requests_total{status_code!~\"5..\"}[30d])) / sum(rate(http_requests_total[30d]))) * 100",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 99,
            "max": 100,
            "thresholds": {
              "steps": [
                {"value": 99, "color": "red"},
                {"value": 99.5, "color": "yellow"},
                {"value": 99.9, "color": "green"}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Error Budget Remaining",
        "type": "stat",
        "gridPos": {"x": 8, "y": 0, "w": 8, "h": 4},
        "targets": [
          {
            "expr": "((0.999 - (1 - (sum(rate(http_requests_total{status_code!~\"5..\"}[30d])) / sum(rate(http_requests_total[30d]))))) / 0.001) * 100",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"value": 0, "color": "red"},
                {"value": 25, "color": "yellow"},
                {"value": 50, "color": "green"}
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Error Budget Burn Rate",
        "type": "graph",
        "gridPos": {"x": 16, "y": 0, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "(1 - (sum(rate(http_requests_total{status_code!~\"5..\"}[1h])) / sum(rate(http_requests_total[1h])))) * 100",
            "legendFormat": "Hourly burn rate",
            "refId": "A"
          }
        ]
      },
      {
        "id": 4,
        "title": "Latency SLO (p95 < 500ms)",
        "type": "graph",
        "gridPos": {"x": 0, "y": 8, "w": 24, "h": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) * 1000",
            "legendFormat": "p95 latency",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "ms"
          }
        },
        "options": {
          "thresholds": [
            {
              "value": 500,
              "color": "red",
              "op": "gt"
            }
          ]
        }
      }
    ]
  }
}
```

---

## 🔥 Шаг 6: Финальная сборка и тестирование (15 минут)

### 6.1 Init Database Script

`init-db.sql`:

sql

```sql
-- Create tables
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);

-- Insert sample data
INSERT INTO products (name, description, price, stock) VALUES
('Laptop Pro', 'High-performance laptop', 1299.99, 50),
('Wireless Mouse', 'Ergonomic wireless mouse', 29.99, 200),
('USB-C Cable', 'Fast charging cable', 19.99, 500),
('External SSD', '1TB portable storage', 149.99, 100),
('Mechanical Keyboard', 'RGB gaming keyboard', 89.99, 75),
('Monitor 27"', '4K UHD display', 449.99, 30),
('Webcam HD', '1080p streaming webcam', 79.99, 120),
('Headphones', 'Noise-cancelling headphones', 199.99, 60),
('Desk Lamp', 'LED adjustable lamp', 39.99, 150),
('Standing Desk', 'Electric height-adjustable', 599.99, 20);

-- Create indexes
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_products_price ON products(price);
```

### 6.2 Setup Script

`scripts/setup.sh`:

bash

```bash
#!/bin/bash

echo "🚀 Setting up E-Commerce Monitoring Stack"
echo "=========================================="

# Colors
GREEN='\033[0.32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

# Check Docker
if ! command -v docker &> /dev/null; then
    echo -e "${RED}❌ Docker not found. Please install Docker first.${NC}"
    exit 1
fi

# Check Docker Compose
if ! command -v docker-compose &> /dev/null; then
    echo -e "${RED}❌ Docker Compose not found. Please install Docker Compose first.${NC}"
    exit 1
fi

# Create directories
echo -e "${YELLOW}📁 Creating directories...${NC}"
mkdir -p monitoring/prometheus/alerts
mkdir -p monitoring/prometheus/rules
mkdir -p monitoring/grafana/dashboards
mkdir -p monitoring/loki
mkdir -p monitoring/promtail
mkdir -p monitoring/alertmanager
mkdir -p apps/{frontend,api,product-service,payment-service,inventory-service}
mkdir -p scripts
mkdir -p docs

# Build images
echo -e "${YELLOW}🔨 Building Docker images...${NC}"
docker-compose build

# Start infrastructure services first
echo -e "${YELLOW}🗄️  Starting databases...${NC}"
docker-compose up -d postgres redis rabbitmq
sleep 10

# Start monitoring stack
echo -e "${YELLOW}📊 Starting monitoring stack...${NC}"
docker-compose up -d prometheus grafana loki promtail jaeger alertmanager blackbox-exporter node-exporter cadvisor incident-manager

# Wait for services to be healthy
echo -e "${YELLOW}⏳ Waiting for services to be ready...${NC}"
sleep 30

# Start application services
echo -e "${YELLOW}🚀 Starting application services...${NC}"
docker-compose up -d frontend api product-service payment-service inventory-service

# Start load generator
echo -e "${YELLOW}📈 Starting load generator...${NC}"
docker-compose up -d load-generator

# Health checks
echo ""
echo -e "${GREEN}✅ Stack is up! Running health checks...${NC}"
echo ""

services=(
    "Prometheus:http://localhost:9090/-/healthy"
    "Grafana:http://localhost:3001/api/health"
    "Alertmanager:http://localhost:9093/-/healthy"
    "API:http://localhost:4000/health"
    "Jaeger:http://localhost:16686"
)

for service in "${services[@]}"; do
    name="${service%%:*}"
    url="${service##*:}"
    
    if curl -sf "$url" > /dev/null 2>&1; then
        echo -e "  ${GREEN}✓${NC} $name is healthy"
    else
        echo -e "  ${RED}✗${NC} $name is not responding"
    fi
done

echo ""
echo -e "${GREEN}========================================${NC}"
echo -e "${GREEN}🎉 Setup complete!${NC}"
echo ""
echo -e "${YELLOW}Access your services:${NC}"
echo "  📊 Grafana:        http://localhost:3001 (admin/admin123)"
echo "  📈 Prometheus:     http://localhost:9090"
echo "  🔍 Jaeger:         http://localhost:16686"
echo "  🚨 Alertmanager:   http://localhost:9093"
echo "  🌐 Frontend:       http://localhost:3000"
echo "  🔧 API:            http://localhost:4000"
echo "  📦 Product Service: http://localhost:5000"
echo "  💳 Payment Service: http://localhost:6000"
echo "  📊 Incident Manager: http://localhost:5001"
echo ""
echo -e "${YELLOW}Useful commands:${NC}"
echo "  docker-compose logs -f [service]  - View logs"
echo "  docker-compose ps                 - Check status"
echo "  docker-compose down               - Stop all services"
echo "  docker-compose restart [service]  - Restart a service"
echo ""
echo -e "${YELLOW}Next steps:${NC}"
echo "  1. Import dashboards in Grafana"
echo "  2. Configure Slack webhook in .env"
echo "  3. Run ./scripts/load-test.sh to generate traffic"
echo "  4. Run ./scripts/chaos-test.sh to test alerting"
echo ""
```

Сделай исполняемым:

bash

```bash
chmod +x scripts/setup.sh
```

### 6.3 Load Test Script

`scripts/load-test.sh`:

bash

```bash
#!/bin/bash

echo "🔥 Starting Load Test"
echo "===================="

# Colors
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Test scenarios
scenarios=(
    "Normal Load"
    "High Load"
    "Spike Test"
    "Stress Test"
)

# Function to run load test
run_test() {
    local name=$1
    local duration=$2
    local concurrency=$3
    local rate=$4
    
    echo -e "\n${YELLOW}Running: $name${NC}"
    echo "  Duration: ${duration}s"
    echo "  Concurrency: $concurrency"
    echo "  Rate: $rate req/s"
    
    hey -z ${duration}s -c $concurrency -q $rate \
        -m GET \
        -H "User-Agent: LoadTest" \
        http://localhost:4000/api/products
    
    echo -e "${GREEN}✓ $name completed${NC}"
    sleep 10
}

echo "Select test scenario:"
echo "1) Normal Load (60s, 10 concurrent, 10 req/s)"
echo "2) High Load (120s, 50 concurrent, 50 req/s)"
echo "3) Spike Test (30s burst, 100 concurrent, 100 req/s)"
echo "4) Stress Test (300s, 200 concurrent, 200 req/s)"
echo "5) All scenarios"
read -p "Choice [1-5]: " choice

case $choice in
    1)
        run_test "Normal Load" 60 10 10
        ;;
    2)
        run_test "High Load" 120 50 50
        ;;
    3)
        run_test "Spike Test" 30 100 100
        ;;
    4)
        run_test "Stress Test" 300 200 200
        ;;
    5)
        run_test "Normal Load" 60 10 10
        run_test "High Load" 120 50 50
        run_test "Spike Test" 30 100 100
        ;;
    *)
        echo "Invalid choice"
        exit 1
        ;;
esac

echo ""
echo -e "${GREEN}Load test completed!${NC}"
echo ""
echo "Check results in:"
echo "  📊 Grafana: http://localhost:3001/d/app-performance"
echo "  📈 Prometheus: http://localhost:9090/graph"
```

bash

```bash
chmod +x scripts/load-test.sh
```

### 6.4 Chaos Test Script

`scripts/chaos-test.sh`:

bash

```bash
#!/bin/bash

echo "🐵 Chaos Engineering Tests"
echo "=========================="

# Colors
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

experiments=(
    "Kill API Service"
    "Kill Database"
    "Fill Disk Space"
    "Network Latency"
    "CPU Spike"
    "Memory Leak"
)

# Menu
echo "Select chaos experiment:"
echo "1) Kill API Service"
echo "2) Kill Database"
echo "3) Fill Disk Space"
echo "4) Add Network Latency"
echo "5) CPU Spike"
echo "6) Memory Leak Simulation"
echo "7) Run All Experiments (with 5min intervals)"
read -p "Choice [1-7]: " choice

kill_service() {
    local service=$1
    echo -e "\n${RED}💀 Killing $service${NC}"
    docker-compose kill $service
    echo "Waiting 2 minutes to observe..."
    sleep 120
    echo -e "${GREEN}✓ Restarting $service${NC}"
    docker-compose up -d $service
}

kill_database() {
    echo -e "\n${RED}💀 Killing PostgreSQL${NC}"
    docker-compose kill postgres
    echo "Waiting 2 minutes to observe..."
    sleep 120
    echo -e "${GREEN}✓ Restarting database${NC}"
    docker-compose up -d postgres
    sleep 30  # Wait for DB to be ready
}

fill_disk() {
    echo -e "\n${RED}💾 Filling disk space (1GB file)${NC}"
    docker exec api sh -c "dd if=/dev/zero of=/tmp/bigfile bs=1M count=1000"
    echo "Waiting 2 minutes to observe..."
    sleep 120
    echo -e "${GREEN}✓ Cleaning up${NC}"
    docker exec api rm /tmp/bigfile
}

add_latency() {
    echo -e "\n${RED}🐌 Adding 500ms network latency${NC}"
    docker exec api sh -c "apt-get update && apt-get install -y iproute2 && tc qdisc add dev eth0 root netem delay 500ms" 2>/dev/null
    echo "Waiting 2 minutes to observe..."
    sleep 120
    echo -e "${GREEN}✓ Removing latency${NC}"
    docker exec api sh -c "tc qdisc del dev eth0 root" 2>/dev/null
}

cpu_spike() {
    echo -e "\n${RED}🔥 Creating CPU spike (4 cores, 2min)${NC}"
    docker exec -d api sh -c "yes > /dev/null & yes > /dev/null & yes > /dev/null & yes > /dev/null"
    echo "Waiting 2 minutes to observe..."
    sleep 120
    echo -e "${GREEN}✓ Stopping CPU spike${NC}"
    docker exec api pkill yes
}

memory_leak() {
    echo -e "\n${RED}💧 Simulating memory leak${NC}"
    docker exec -d api node -e "
        const arr = [];
        setInterval(() => {
            for(let i = 0; i < 100000; i++) {
                arr.push(new Array(1000).fill('x'));
            }
        }, 100);
        setTimeout(() => process.exit(0), 120000);
    "
    echo "Waiting 2 minutes to observe..."
    sleep 120
    echo -e "${GREEN}✓ Restarting service${NC}"
    docker-compose restart api
}

case $choice in
    1) kill_service "api" ;;
    2) kill_database ;;
    3) fill_disk ;;
    4) add_latency ;;
    5) cpu_spike ;;
    6) memory_leak ;;
    7)
        echo -e "${YELLOW}Running all experiments with 5min intervals${NC}"
        kill_service "api"
        sleep 300
        kill_database
        sleep 300
        cpu_spike
        sleep 300
        memory_leak
        ;;
    *)
        echo "Invalid choice"
        exit 1
        ;;
esac

echo ""
echo -e "${GREEN}Chaos experiment completed!${NC}"
echo ""
echo "Check monitoring:"
echo "  🚨 Alerts: http://localhost:9093"
echo "  📊 Dashboards: http://localhost:3001"
echo "  📝 Incidents: http://localhost:5001/incidents"
```

bash

```bash
chmod +x scripts/chaos-test.sh
```

### 6.5 Backup Script

`scripts/backup-dashboards.sh`:

bash

```bash
#!/bin/bash

echo "💾 Backing up Grafana dashboards"
echo "================================"

BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Get all dashboards
dashboards=$(curl -s -H "Authorization: Bearer admin:admin123" \
    http://localhost:3001/api/search?type=dash-db | \
    jq -r '.[].uid')

for uid in $dashboards; do
    dashboard=$(curl -s -H "Authorization: Bearer admin:admin123" \
        "http://localhost:3001/api/dashboards/uid/$uid")
    
    title=$(echo $dashboard | jq -r '.meta.slug')
    echo "  ✓ Backing up: $title"
    
    echo $dashboard | jq '.dashboard' > "$BACKUP_DIR/${title}.json"
done

echo ""
echo "✅ Backup completed: $BACKUP_DIR"
```

bash

```bash
chmod +x scripts/backup-dashboards.sh
```

---

## 📚 Шаг 7: Документация

### 7.1 Runbooks

`docs/RUNBOOKS.md`:

markdown

````markdown
# Runbooks - E-Commerce Platform

## Table of Contents
1. [High Error Rate](#high-error-rate)
2. [High Latency](#high-latency)
3. [Database Connection Issues](#database-connection-issues)
4. [Service Down](#service-down)
5. [Order Creation Failures](#order-creation-failures)

---

## High Error Rate

**Alert:** `HighErrorRate`  
**Severity:** Critical  
**SLO Impact:** High

### Symptoms
- Alert fires when error rate >5% for 5 minutes
- Users report 500 errors
- Decreased order completion rate

### Investigation Steps

1. **Check Error Dashboard**
```
   http://localhost:3001/d/app-performance
```

2. **Query Recent Errors**
```bash
   # In Grafana Explore (Loki)
   {job="api-service"} |= "ERROR"
```

3. **Check Service Health**
```bash
   docker-compose ps
   curl http://localhost:4000/health
```

4. **Identify Error Pattern**
   - Database errors → Go to [Database Issues](#database-connection-issues)
   - Timeout errors → Go to [High Latency](#high-latency)
   - 5xx on specific endpoint → Check that service

### Resolution

**Quick Fix:**
```bash
# Restart affected service
docker-compose restart api

# If database related
docker-compose restart postgres
```

**If errors persist:**
1. Check recent deployments
2. Rollback if needed:
```bash
   docker-compose down api
   docker-compose up -d api
```
3. Scale horizontally:
```bash
   docker-compose up -d --scale api=3
```

### Prevention
- [ ] Add circuit breakers
- [ ] Implement retry logic with exponential backoff
- [ ] Add health checks
- [ ] Review error handling in code

---

## High Latency

**Alert:** `HighLatency`  
**Severity:** Warning  
**SLO Impact:** Medium

### Symptoms
- p95 latency >2s for 10 minutes
- Slow page loads
- User complaints about performance

### Investigation

1. **Check Latency Dashboard**
```
   http://localhost:3001/d/app-performance
   Panel: Response Time (p50, p95, p99)
```

2. **Identify Slow Endpoint**
```promql
   topk(5, 
     histogram_quantile(0.95, 
       sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
     )
   )
```

3. **Check Dependencies**
   - Database query time
   - External API calls
   - Cache hit rate

4. **Review Traces in Jaeger**
```
   http://localhost:16686
   Search by: Service=api-service, Tags=error:true
```

### Resolution

**Immediate Actions:**
```bash
# Check database performance
docker exec postgres psql -U admin -d ecommerce -c "
  SELECT query, mean_exec_time, calls 
  FROM pg_stat_statements 
  ORDER BY mean_exec_time DESC 
  LIMIT 10;
"

# Clear cache if stale
docker exec redis redis-cli FLUSHDB

# Add more instances
docker-compose up -d --scale api=5
```

**Long-term:**
- Optimize slow queries (add indexes)
- Implement caching strategy
- Add query timeouts
- Consider async processing

### Prevention
- [ ] Regular performance testing
- [ ] Database query optimization
- [ ] Implement APM profiling
- [ ] Set up load testing in CI/CD

---

## Database Connection Issues

**Alert:** `DatabaseConnectionPoolExhausted`  
**Severity:** Critical  
**SLO Impact:** High

### Symptoms
- "Connection pool exhausted" errors
- Timeouts connecting to database
- Applications unable to process requests

### Investigation

1. **Check Connection Pool**
```promql
   # Current connections
   pg_stat_activity_count
   
   # Max connections
   pg_settings_max_connections
   
   # Usage percentage
   (pg_stat_activity_count / pg_settings_max_connections) * 100
```

2. **Identify Connection Leaks**
```bash
   docker exec postgres psql -U admin -d ecommerce -c "
     SELECT pid, usename, application_name, state, 
            now() - state_change as duration
     FROM pg_stat_activity
     WHERE state = 'idle in transaction'
     ORDER BY duration DESC;
   "
```

3. **Check for Long-Running Queries**
```sql
   SELECT pid, now() - query_start AS duration, query
   FROM pg_stat_activity
   WHERE state != 'idle'
   ORDER BY duration DESC
   LIMIT 10;
```

### Resolution

**Immediate Actions:**
```bash
# Kill idle connections
docker exec postgres psql -U admin -d ecommerce -c "
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'idle in transaction'
  AND now() - state_change > interval '5 minutes';
"

# Increase max_connections temporarily
docker exec postgres psql -U admin -d ecommerce -c "
  ALTER SYSTEM SET max_connections = 1000;
  SELECT pg_reload_conf();
"

# Restart application to reset pools
docker-compose restart api product-service payment-service
```

**Long-term Fix:**
```javascript
// Configure connection pool properly
const pool = new Pool({
  max: 20,              // Maximum connections per instance
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
})
```

### Prevention
- [ ] Implement connection pool monitoring
- [ ] Add connection leak detection
- [ ] Set proper pool sizes
- [ ] Add circuit breakers for database

---

## Service Down

**Alert:** `ServiceDown`  
**Severity:** Critical  
**SLO Impact:** Critical

### Symptoms
- Service returns no response
- Health check fails
- Users cannot access features

### Investigation

1. **Check Service Status**
```bash
   docker-compose ps api
   docker logs api --tail 100
```

2. **Check Resource Usage**
```bash
   docker stats api
```

3. **Check Dependencies**
```bash
   # Database
   docker-compose ps postgres
   
   # Redis
   docker-compose ps redis
```

### Resolution

**Quick Recovery:**
```bash
# Restart service
docker-compose restart api

# If still failing, rebuild
docker-compose up -d --force-recreate api

# Check logs
docker-compose logs -f api
```

**If container keeps crashing:**
```bash
# Check for OOMKilled
docker inspect api | grep OOMKilled

# Increase memory limit
# Edit docker-compose.yml
services:
  api:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G

# Restart with new limits
docker-compose up -d api
```

### Prevention
- [ ] Implement health checks
- [ ] Add liveness/readiness probes
- [ ] Set resource limits and requests
- [ ] Implement auto-restart policies

---

## Order Creation Failures

**Alert:** `HighOrderFailureRate`  
**Severity:** Critical  
**SLO Impact:** Business Critical

### Symptoms
- Orders failing at checkout
- "Payment processing error" messages
- Revenue drop

### Investigation

1. **Check Order Metrics**
```promql
   rate(orders_total{status="failed"}[5m])
   rate(orders_total{status="completed"}[5m])
```

2. **Check Payment Service**
```bash
   curl http://localhost:6000/health
   docker logs payment-service --tail 50
```

3. **Check Database**
```bash
   # Check for locks
   docker exec postgres psql -U admin -d ecommerce -c "
     SELECT * FROM pg_locks 
     WHERE NOT granted;
   "
```

4. **Review Recent Orders**
```sql
   SELECT id, user_id, total, status, created_at
   FROM orders
   WHERE created_at > NOW() - INTERVAL '1 hour'
   ORDER BY created_at DESC
   LIMIT 50;
```

### Resolution

**Immediate Actions:**
```bash
# Check payment gateway status
curl https://api.payment-provider.com/status

# Restart payment service
docker-compose restart payment-service

# If database locks
docker exec postgres psql -U admin -d ecommerce -c "
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE wait_event_type = 'Lock';
"
```

**Enable Failover:**
```javascript
// Implement retry logic
async function createOrder(orderData) {
  const maxRetries = 3;
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await processOrder(orderData);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(1000 * Math.pow(2, i)); // Exponential backoff
    }
  }
}
```

### Prevention
- [ ] Implement order queue with retry
- [ ] Add payment gateway health checks
- [ ] Implement idempotency for order creation
- [ ] Add order status monitoring

---

## Emergency Contacts

**On-Call Engineers:**
- Primary: @oncall-primary
- Secondary: @oncall-secondary

**Escalation:**
- Engineering Manager: @eng-manager
- CTO: @cto

**External Contacts:**
- Payment Gateway Support: +1-XXX-XXX-XXXX
- Cloud Provider Support: portal.cloud.com/support
- Database Support: @dba-team
````

### 7.2 Architecture Documentation

`docs/ARCHITECTURE.md`:

markdown

````markdown
# E-Commerce Platform - Architecture

## Overview

This is a microservices-based e-commerce platform with comprehensive monitoring.

## Service Architecture
```
┌──────────────┐
│   Frontend   │ (React, Port 3000)
└──────┬───────┘
       │
┌──────▼───────┐
│  API Gateway │ (Node.js, Port 4000)
└──────┬───────┘
       │
       ├─────────────────┬─────────────────┬
       │                 │                 │
┌──────▼────────┐ ┌─────▼──────┐ ┌───────▼─────────┐
│Product Service│ │Payment Svc │ │Inventory Service│
│(Python:5000)  │ │(Go:6000)   │ │(Java:7000)      │
└──────┬────────┘ └─────┬──────┘ └───────┬─────────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
              ┌─────────┼─────────┐
              │         │         │
         ┌────▼───┐ ┌──▼───┐ ┌───▼────┐
         │Postgres│ │Redis │ │RabbitMQ│
         └────────┘ └──────┘ └────────┘
```

## Monitoring Stack
```
┌────────────────────────────────────────┐
│          Monitoring Layer              │
├────────────────────────────────────────┤
│                                        │
│  ┌──────────┐  ┌──────────┐          │
│  │Prometheus│  │  Loki    │          │
│  │(Metrics) │  │ (Logs)   │          │
│  └────┬─────┘  └────┬─────┘          │
│       │             │                 │
│       └──────┬──────┘                 │
│              │                        │
│         ┌────▼─────┐                 │
│         │ Grafana  │ (Visualization) │
│         └──────────┘                 │
│                                        │
│  ┌──────────┐  ┌─────────────┐       │
│  │  Jaeger  │  │ Alertmanager│       │
│  │(Tracing) │  │  (Alerts)   │       │
│  └──────────┘  └─────────────┘       │
└────────────────────────────────────────┘
```

## Data Flow

### Order Creation Flow
```
1. User clicks "Place Order" (Frontend)
   ↓
2. POST /api/orders (API Gateway)
   ↓
3. Validate cart items (Product Service)
   ↓
4. Check inventory (Inventory Service)
   ↓
5. Process payment (Payment Service)
   ↓
6. Create order record (Database)
   ↓
7. Send confirmation (RabbitMQ → Email Service)
   ↓
8. Return order ID to user
```

### Monitoring Data Flow
```
Application
   │
   ├─→ Metrics → Prometheus → Grafana
   ├─→ Logs → Promtail → Loki → Grafana
   └─→ Traces → Jaeger → Grafana
```

## Technology Stack

**Frontend:**
- React 18
- Nginx (reverse proxy)

**Backend Services:**
- API: Node.js + Express
- Product Service: Python + Flask
- Payment Service: Go
- Inventory Service: Java + Spring Boot

**Databases:**
- PostgreSQL 15 (primary data store)
- Redis 7 (caching)
- RabbitMQ 3 (message queue)

**Monitoring:**
- Prometheus 2.x (metrics)
- Grafana 10.x (visualization)
- Loki 2.x (logs)
- Jaeger 1.x (tracing)
- Alertmanager 0.x (alerting)

## Deployment

**Container Orchestration:**
- Docker Compose (development)
- Kubernetes (production - not included in this demo)

**Resource Allocation:**
```yaml
API Service:
  CPU: 0.5 cores
  Memory: 512MB
  Replicas: 3

Product Service:
  CPU: 0.25 cores
  Memory: 256MB
  Replicas: 2

Database:
  CPU: 1 core
  Memory: 2GB
  Replicas: 1 (with standby in prod)
```

## Scaling Strategy

**Horizontal Scaling:**
- API Gateway: 3-10 instances
- Product Service: 2-5 instances
- Payment Service: 2-5 instances
- Inventory Service: 2-5 instances

**Vertical Scaling:**
- Database: Upgrade to larger instance when needed
- Redis: Increase memory allocation

**Auto-scaling triggers:**
- CPU > 70% for 5 minutes
- Request rate > 1000 req/s
- Queue depth > 100 messages

## Security

**Authentication:**
- JWT tokens for API authentication
- OAuth2 for third-party integrations

**Network Security:**
- All services in private network
- Only API Gateway exposed publicly
- TLS for all external communications

**Data Security:**
- Encrypted database connections
- Secrets managed via environment variables
- No sensitive data in logs

## Disaster Recovery

**Backup Strategy:**
- Database: Daily backups, 30-day retention
- Configuration: Version controlled in Git
- Dashboards: Automated backup via script

**Recovery Time Objectives:**
- RTO: 1 hour
- RPO: 24 hours

**Failover Procedures:**
1. Primary database fails → Promote standby
2. Service crashes → Auto-restart via Docker
3. Complete datacenter failure → Failover to DR region

## Performance Targets

**SLOs:**
- Availability: 99.9% uptime
- Latency: p95 < 500ms
- Error Rate: < 0.1%

**Capacity:**
- 10,000 requests/minute
- 100,000 daily active users
- 5,000 orders/day

## Monitoring Dashboards

1. **Business Metrics**
   - Revenue tracking
   - Order volume
   - Conversion rates

2. **Application Performance**
   - Request rates
   - Error rates
   - Latency percentiles

3. **Infrastructure**
   - CPU/Memory usage
   - Disk space
   - Network traffic

4. **SLO Dashboard**
   - Availability %
   - Error budget
   - SLO compliance

## Alerting

**Critical Alerts (Page on-call):**
- Service down
- Error rate > 5%
- Order creation failures
- Database connection issues

**Warning Alerts (Create ticket):**
- High latency
- High resource usage
- Low throughput

## Further Reading

- [Runbooks](./RUNBOOKS.md)
- [Alert Definitions](./ALERTS.md)
- [Post-Mortem Template](./POST_MORTEM_TEMPLATE.md)
````

### 7.3 Post-Mortem Template

`docs/POST_MORTEM_TEMPLATE.md`:

markdown

```markdown
# Post-Mortem Template

**Copy this template for each incident**

---

## Incident: [Title]

**Date:** YYYY-MM-DD  
**Authors:** @username1, @username2  
**Status:** Draft | Review | Published  
**Severity:** SEV-1 | SEV-2 | SEV-3  

## Executive Summary

[2-3 sentence summary of what happened, impact, and resolution]

## Impact

**Timeline:**
- **Started:** YYYY-MM-DD HH:MM UTC
- **Detected:** YYYY-MM-DD HH:MM UTC
- **Resolved:** YYYY-MM-DD HH:MM UTC
- **Duration:** X hours Y minutes

**Affected Services:**
- [ ] Frontend
- [ ] API Gateway
- [ ] Product Service
- [ ] Payment Service
- [ ] Inventory Service
- [ ] Database
- [ ] Other: ___________

**User Impact:**
- Users affected: X,XXX (~XX%)
- Failed requests: X,XXX
- Revenue impact: $X,XXX

**SLO Impact:**
- Availability: XX.XX% (target: 99.9%)
- Error budget consumed: XX% of monthly budget
- Latency p95: XXXms (target: <500ms)

## Timeline (All times UTC)

**Detection Phase:**
- HH:MM - [Event description]
- HH:MM - [Alert fired / User report received]
- HH:MM - [On-call engineer acknowledged]

**Investigation Phase:**
- HH:MM - [Investigation step 1]
- HH:MM - [Discovery 1]
- HH:MM - [Investigation step 2]
- HH:MM - [Root cause identified]

**Mitigation Phase:**
- HH:MM - [Attempted fix 1]
- HH:MM - [Result of fix 1]
- HH:MM - [Attempted fix 2]
- HH:MM - [Partial recovery]

**Resolution Phase:**
- HH:MM - [Final fix applied]
- HH:MM - [Full recovery confirmed]
- HH:MM - [Incident closed]

## Root Cause

[Detailed explanation of what caused the incident]

**Technical Details:**
- Primary cause: [Description]
- Contributing factors:
  1. [Factor 1]
  2. [Factor 2]
  3. [Factor 3]

**Why did our safeguards fail?**
- [Explanation of why existing monitoring/alerting/controls didn't prevent this]

## Detection

**How was this detected?**
- [ ] Automated alert
- [ ] User report
- [ ] Internal discovery
- [ ] Other: ___________

**Alert details:**
- Alert name: [AlertName]
- Fired at: HH:MM UTC
- Time to acknowledge: X minutes

**Could we have detected this earlier?**
[Discussion of earlier warning signs we missed]

## What Went Well ✅

- [Thing that worked well #1]
- [Thing that worked well #2]
- [Thing that worked well #3]

## What Didn't Go Well ❌

- [Thing that didn't work #1]
- [Thing that didn't work #2]
- [Thing that didn't work #3]

## Where We Got Lucky 🍀

- [Lucky break #1 that prevented worse outcome]
- [Lucky break #2]

## Action Items

### Immediate (Complete within 1 week)

| Action | Owner | Priority | Due Date | Status | PR/Ticket |
|--------|-------|----------|----------|--------|-----------|
| [Action 1] | @owner | P0 | YYYY-MM-DD | ☐ | [Link] |
| [Action 2] | @owner | P0 | YYYY-MM-DD | ☐ | [Link] |

### Short-term (Complete within 1 month)

| Action | Owner | Priority | Due Date | Status | PR/Ticket |
|--------|-------|----------|----------|--------|-----------|
| [Action 1] | @owner | P1 | YYYY-MM-DD | ☐ | [Link] |
| [Action 2] | @owner | P1 | YYYY-MM-DD | ☐ | [Link] |

### Long-term (Complete within 1 quarter)

| Action | Owner | Priority | Due Date | Status | PR/Ticket |
|--------|-------|----------|----------|--------|-----------|
| [Action 1] | @owner | P2 | YYYY-MM-DD | ☐ | [Link] |
| [Action 2] | @owner | P2 | YYYY-MM-DD | ☐ | [Link] |

## Lessons Learned

### Technical
1. [Lesson 1]
2. [Lesson 2]
3. [Lesson 3]

### Process
1. [Lesson 1]
2. [Lesson 2]

### People/Communication
1. [Lesson 1]
2. [Lesson 2]

## Supporting Information

### Dashboards
- [Link to relevant dashboard]
- [Link to relevant dashboard]

### Logs
```

[Relevant log snippets]

```

### Metrics
![Metric graph](link-to-image)

### Related Incidents
- [INC-XXXX](link) - Similar incident from [date]

### Customer Communication
- Status page updates: [timestamps and messages]
- Support tickets created: XXX
- Average resolution time: XX minutes

## Review

**Review Meeting:**
- Date: YYYY-MM-DD
		HH:MM UTC

- Attendees: [List]
- Recording: [Link]

**Approvals:**

- [ ]  Engineering Lead
- [ ]  Product Manager
- [ ]  CTO (if SEV-1)

---

**Template version:** 1.0  
**Last updated:** 2025-01-15

````

---

## 🎓 Шаг 8: Тестирование и валидация

### 8.1 Полный запуск системы
```bash
# 1. Запусти setup script
./scripts/setup.sh

# 2. Проверь что все сервисы работают
docker-compose ps

# Ожидаемый вывод:
# NAME                  STATUS              PORTS
# prometheus            Up 2 minutes        0.0.0.0:9090->9090/tcp
# grafana               Up 2 minutes        0.0.0.0:3001->3000/tcp
# api                   Up 2 minutes        0.0.0.0:4000->4000/tcp
# ...

# 3. Проверь логи
docker-compose logs -f api | head -20
docker-compose logs -f prometheus | head -20

# 4. Проверь метрики
curl http://localhost:4000/metrics | grep http_requests_total
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

### 8.2 Тест мониторинга
```bash
# Сгенерируй нормальный трафик
./scripts/load-test.sh
# Выбери: 1 (Normal Load)

# Открой Grafana и проверь:
# 1. Business Metrics Dashboard
#    - Revenue растет
#    - Orders создаются
#    - Conversion rate > 0

# 2. Application Performance Dashboard
#    - Request rate > 0
#    - Error rate < 1%
#    - Latency p95 < 500ms

# 3. Infrastructure Dashboard
#    - CPU < 50%
#    - Memory < 70%
#    - Disk space OK
```

### 8.3 Тест алертинга
```bash
# Запусти chaos test
./scripts/chaos-test.sh
# Выбери: 1 (Kill API Service)

# Проверь что произошло:

# 1. Alert должен сработать через ~2 минуты
curl http://localhost:9093/api/v2/alerts | jq '.[] | {name: .labels.alertname, state: .status.state}'

# Ожидаемый output:
# {
#   "name": "ServiceDown",
#   "state": "firing"
# }

# 2. Инцидент должен создаться
curl http://localhost:5001/incidents | jq '.[0]'

# 3. После 2 минут сервис восстановится
# Alert должен перейти в resolved

# 4. Проверь дашборд инцидентов
open http://localhost:3001/d/oncall-dashboard
```

### 8.4 Тест логирования
```bash
# Проверь что логи собираются
# В Grafana:
# 1. Открой Explore
# 2. Выбери Loki datasource
# 3. Query: {job="api-service"}

# Должны видеть логи вида:
# {
#   "timestamp": "2025-01-15T10:30:00Z",
#   "level": "INFO",
#   "method": "GET",
#   "path": "/api/products",
#   "status": 200,
#   "duration": 0.045
# }

# Фильтры:
# {job="api-service"} |= "ERROR"
# {job="api-service"} | json | status >= 500
```

### 8.5 Тест трейсинга
```bash
# Сделай несколько запросов
for i in {1..10}; do
  curl http://localhost:4000/api/products
  sleep 1
done

# Открой Jaeger
open http://localhost:16686

# 1. Service: api-service
# 2. Operation: GET /api/products
# 3. Find Traces

# Должны видеть spans:
# - get_products (api)
# - cache_check (redis)
# - db_query (postgres)
# - cache_set (redis)

# Кликни на трейс чтобы увидеть водопад
```

---

## 🏆 Шаг 9: Финальная проверка

### Checklist
````
Инфраструктура:
[✓] Все сервисы запущены
[✓] Health checks проходят
[✓] База данных наполнена данными
[✓] Логи пишутся корректно
Метрики:
[✓] Prometheus собирает метрики
[✓] Business метрики работают
[✓] Application метрики работают
[✓] Infrastructure метрики работают
[✓] Recording rules применены
Визуализация:
[✓] Grafana подключена к Prometheus
[✓] Grafana подключена к Loki
[✓] Grafana подключена к Jaeger
[✓] Все дашборды импортированы
[✓] Дашборды показывают данные
Логирование:
[✓] Loki собирает логи
[✓] Promtail работает
[✓] Логи видны в Grafana
[✓] Structured logging работает
Трейсинг:
[✓] Jaeger собирает трейсы
[✓] Трейсы видны в UI
[✓] Spans правильно связаны
[✓] Контекст передается между сервисами
Алертинг:
[✓] Alertmanager работает
[✓] Правила алертов загружены
[✓] Тестовый алерт срабатывает
[✓] Алерты приходят в Incident Manager
[✓] Email/Slack уведомления работают (если настроены)
Incident Management:
[✓] Incident Manager работает
[✓] Инциденты создаются при алертах
[✓] API /incidents работает
[✓] Статистика собирается
Синтетический мониторинг:
[✓] Blackbox exporter работает
[✓] HTTP проверки проходят
[✓] Метрики uptime собираются
Тестирование:
[✓] Load test пройден
[✓] Chaos test пройден
[✓] Метрики корректные при нагрузке
[✓] Алерты срабатывают при проблемах
Документация:
[✓] README.md написан
[✓] RUNBOOKS.md заполнен
[✓] ARCHITECTURE.md описана
[✓] POST_MORTEM_TEMPLATE.md создан
````

### Итоговая валидация
```bash
# Запусти полный тест
./scripts/validate-stack.sh
```

`scripts/validate-stack.sh`:
```bash
#!/bin/bash

echo "🔍 Validating Monitoring Stack"
echo "=============================="

FAILED=0

check_service() {
    local name=$1
    local url=$2
    
    if curl -sf "$url" > /dev/null 2>&1; then
        echo "✅ $name is healthy"
    else
        echo "❌ $name is not responding"
        ((FAILED++))
    fi
}

check_metrics() {
    local query=$1
    local expected=$2
    
    result=$(curl -s "http://localhost:9090/api/v1/query?query=$query" | jq -r '.data.result[0].value[1]')
    
    if [ "$result" != "null" ] && [ "$result" != "" ]; then
        echo "✅ Metric query successful: $query"
    else
        echo "❌ Metric query failed: $query"
        ((FAILED++))
    fi
}

echo ""
echo "Checking Services..."
check_service "Prometheus" "http://localhost:9090/-/healthy"
check_service "Grafana" "http://localhost:3001/api/health"
check_service "Alertmanager" "http://localhost:9093/-/healthy"
check_service "Loki" "http://localhost:3100/ready"
check_service "Jaeger" "http://localhost:16686"
check_service "API" "http://localhost:4000/health"
check_service "Incident Manager" "http://localhost:5001/incidents"

echo ""
echo "Checking Metrics..."
check_metrics "up{job='prometheus'}" "1"
check_metrics "up{job='api-service'}" "1"
check_metrics "http_requests_total" "0"

echo ""
echo "Checking Dashboards..."
dashboards=$(curl -s "http://admin:admin123@localhost:3001/api/search?type=dash-db" | jq '. | length')
if [ "$dashboards" -gt 0 ]; then
    echo "✅ $dashboards dashboards found"
else
    echo "❌ No dashboards found"
    ((FAILED++))
fi

echo ""
echo "Checking Alerts..."
alerts=$(curl -s "http://localhost:9090/api/v1/rules" | jq '[.data.groups[].rules[] | select(.type=="alerting")] | length')
if [ "$alerts" -gt 0 ]; then
    echo "✅ $alerts alert rules loaded"
else
    echo "❌ No alert rules found"
    ((FAILED++))
fi

echo ""
echo "=============================="
if [ $FAILED -eq 0 ]; then
    echo "🎉 All checks passed!"
    exit 0
else
    echo "❌ $FAILED checks failed"
    exit 1
fi
```
```bash
chmod +x scripts/validate-stack.sh
```

---

## 🎯 Финальное задание

Теперь, когда стек полностью развернут, выполни эти задачи:

### Задача 1: Создай свой custom dashboard
````

1. Открой Grafana
2. Create → Dashboard
3. Добавь панели:
    - Total Revenue (Last 24h)
    - Active Users
    - Popular Products
    - Payment Success Rate
4. Сохрани как "My Custom Dashboard"
5. Экспортируй JSON

````

### Задача 2: Создай свой алерт
```

1. Создай файл monitoring/prometheus/alerts/custom.yml
2. Добавь алерт:
    - Имя: LowRevenue
    - Условие: Revenue < $500/hour
    - For: 30 минут
    - Severity: warning
3. Перезагрузи Prometheus
4. Проверь что алерт появился в UI

```

### Задача 3: Симулируй incident и напиши post-mortem
```

1. Запусти chaos test (kill database)
2. Дождись срабатывания алертов
3. Исследуй проблему через дашборды
4. Восстанови систему
5. Напиши post-mortem по шаблону
6. Заполни все секции

```

### Задача 4: Оптимизация
```

Найди и исправь:

1. Медленный запрос в базе данных
2. Endpoint с высокой латентностью
3. Сервис с утечкой памяти
4. Неиспользуемый индекс в БД

```

---

## 📝 Заключение

Поздравляю! Ты создал **production-ready monitoring stack** для современного микросервисного приложения.

### Что ты освоил:

✅ **Полный monitoring stack:**
- Prometheus для метрик
- Grafana для визуализации
- Loki для логов
- Jaeger для трейсинга
- Alertmanager для алертов

✅ **Application instrumentation:**
- Custom метрики
- Structured logging
- Distributed tracing
- Business metrics

✅ **Alerting & Incident Management:**
- Правила алертов
- Severity levels
- Runbooks
- Post-mortems

✅ **Best Practices:**
- SLO monitoring
- Error budgets
- On-call workflows
- Chaos engineering

✅ **Documentation:**
- Runbooks
- Architecture docs
- Post-mortem templates

### Следующие шаги:

1. **Адаптируй для своего проекта:**
   - Замени demo-приложения на свои
   - Настрой свои метрики
   - Создай свои дашборды

2. **Добавь интеграции:**
   - Slack для уведомлений
   - PagerDuty для on-call
   - JIRA для тикетов

3. **Масштабируй:**
   - Kubernetes deployment
   - High-availability setup
   - Multi-region monitoring

4. **Углубись:**
   - Advanced PromQL
   - Custom exporters
   - Grafana plugins
   - Machine learning для anomaly detection

### Полезные ресурсы:

**Официальная документация:**
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- Loki: https://grafana.com/docs/loki/
- Jaeger: https://www.jaegertracing.io/docs/

**Книги:**
- "Site Reliability Engineering" - Google
- "The Art of Monitoring" - James Turnbull
- "Distributed Systems Observability" - Cindy Sridharan

**Community:**
- CNCF Slack: cloud-native.slack.com
- Prometheus Users: https://groups.google.com/g/prometheus-users
- Grafana Community: https://community.grafana.com/

---

## 🎓 Сертификация (опционально)

После завершения курса, ты можешь пройти эти сертификации:

- **Prometheus Certified Associate (PCA)**
- **Certified Kubernetes Administrator (CKA)**
- **Google Cloud Professional Cloud DevOps Engineer**
- **AWS Certified DevOps Engineer**

---

## 💬 Feedback

Курс помог? Есть предложения?
Напиши на: [ваш email для обратной связи]

---

**Версия курса:** 1.0  
**Последнее обновление:** 2025-01-15  
**Автор:** [Ваше имя]  
**Лицензия:** MIT

---

# 🎊 Поздравляем с завершением курса!

Ты теперь владеешь навыками **Production-Grade Monitoring** и готов применить их в реальных проектах.

**Помни:**
> "You can't improve what you don't measure" - Peter Drucker

**Удачи в твоем DevOps путешествии!** 🚀
