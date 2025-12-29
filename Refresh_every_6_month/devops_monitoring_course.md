# Продолжение курса: Модуль 4 - Логирование

```bash
# Традиционные логи
tail -f /var/log/syslog
tail -f /var/log/nginx/access.log
grep "ERROR" /var/log/app.log
grep -i "error" /var/log/* | tail -20

# Анализ логов
awk '{print $1}' access.log | sort | uniq -c | sort -rn  # Top IP
awk '{print $9}' access.log | sort | uniq -c | sort -rn  # HTTP codes
```

**LogQL (Loki Query Language):**
```logql
# Stream selector
{job="varlogs"}
{job="varlogs", level="error"}

# Line filter
{job="varlogs"} |= "error"
{job="varlogs"} |~ "error|fail"
{job="varlogs"} != "debug"

# JSON parsing
{job="app"} | json | level="error"

# Rate
rate({job="varlogs"}[5m])

# Count
count_over_time({job="varlogs"}[5m])
```

### 💻 Задание

Настрой централизованное логирование с Loki:

1. **Добавь Loki stack в docker-compose.yml**:
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
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped
    depends_on:
      - loki

volumes:
  loki-data:
```

2. **Создай loki-config.yml**:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
```

3. **Создай promtail-config.yml**:
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # System logs
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

  # Docker containers
  - job_name: containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
```

4. **Запусти Loki stack**:
```bash
docker-compose up -d loki promtail

# Проверка
curl http://localhost:3100/ready
docker logs promtail
```

5. **Добавь Loki в Grafana**:
- Configuration → Data Sources → Add data source
- Выбери Loki
- URL: http://loki:3100
- Save & Test

6. **Создай Log Dashboard в Grafana**:
- Create → Dashboard → Add new panel
- Data source: Loki
- Query: `{job="varlogs"}`
- Visualization: Logs

**Попробуй LogQL запросы:**
```logql
# Все логи
{job="varlogs"}

# Только ошибки
{job="varlogs"} |= "error"

# Docker логи конкретного контейнера
{container="prometheus"}

# Rate логов
rate({job="varlogs"}[5m])

# Количество логов
count_over_time({job="varlogs"}[1h])

# Топ IP адресов
topk(10, sum by (ip) (rate({job="nginx"}[5m])))
```

### 🚀 Бонус (новое)

**Создай приложение с structured logging**:

**app-with-logs.py**:
```python
import logging
import json
import time
from datetime import datetime
from flask import Flask, request
import random

# Structured JSON logging
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "service": "demo-app",
        }
        if hasattr(record, 'request_id'):
            log_obj['request_id'] = record.request_id
        if hasattr(record, 'user_id'):
            log_obj['user_id'] = record.user_id
        if hasattr(record, 'duration_ms'):
            log_obj['duration_ms'] = record.duration_ms
        return json.dumps(log_obj)

# Setup logging
logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

app = Flask(__name__)

@app.route('/')
def home():
    request_id = f"req-{random.randint(1000, 9999)}"
    start = time.time()
    
    logger.info("Request received", extra={
        'request_id': request_id,
        'path': request.path
    })
    
    # Simulate work
    time.sleep(random.uniform(0.1, 0.5))
    
    # Random error
    if random.random() < 0.1:
        logger.error("Internal error occurred", extra={
            'request_id': request_id,
            'error': "Database connection failed"
        })
        return "Error", 500
    
    duration = (time.time() - start) * 1000
    logger.info("Request completed", extra={
        'request_id': request_id,
        'duration_ms': duration
    })
    
    return "OK", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**Dockerfile**:
```dockerfile
FROM python:3.9-slim
WORKDIR /app
RUN pip install flask
COPY app-with-logs.py .
CMD ["python", "app-with-logs.py"]
```

**Добавь в docker-compose.yml**:
```yaml
  demo-app:
    build: ./demo-app
    container_name: demo-app
    ports:
      - "5000:5000"
    restart: unless-stopped
```

**Генерируй трафик**:
```bash
# Install hey (HTTP load generator)
# или используй curl в цикле
while true; do curl http://localhost:5000; sleep 0.5; done
```

**Анализируй логи в Grafana**:
```logql
# Error rate
sum(rate({container="demo-app"} |= "ERROR" [5m]))

# Average duration
avg_over_time({container="demo-app"} | json | unwrap duration_ms [5m])

# Request count by level
sum by (level) (count_over_time({container="demo-app"} [5m]))
```

---

## Модуль 5: Алертинг и On-Call (25 минут)

### 🎯 Напоминалка

**Alertmanager архитектура:**
```
┌────────────┐
│ Prometheus │──┐
└────────────┘  │
                │  ┌──────────────┐    ┌──────────┐
┌────────────┐  ├─►│ Alertmanager │───►│ Receivers│
│ Grafana    │──┘  └──────────────┘    └──────────┘
└────────────┘           │                    │
                         │              Email, Slack
                         │              PagerDuty
                         ▼              Webhook
                    Grouping
                    Throttling
                    Silencing
```

**Alert states:**
```
Inactive  - Условие не выполнено
Pending   - Условие выполнено, но меньше "for:"
Firing    - Условие выполнено дольше "for:"
```

**Alert best practices:**
```
1. Alert на симптомы, не на причины
2. Каждый alert должен быть actionable
3. Включай контекст в annotations
4. Используй severity levels
5. Документируй runbook
6. Тестируй alerts
7. Избегай alert fatigue
```

**Severity levels:**
```
critical  - Нужно немедленное действие (pager)
warning   - Нужно действие в рабочее время
info      - Для информации, действие не требуется
```

**Good vs Bad alerts:**
```
✅ GOOD:
- "API error rate > 5% for 5 minutes"
- "Disk will be full in 4 hours"
- "Payment processing failed"

❌ BAD:
- "CPU usage > 80%"  (не actionable)
- "Log contains ERROR"  (слишком шумно)
- "Service restarted"  (если это норма)
```

**Alertmanager routing:**
```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match:
        severity: warning
      receiver: 'slack'
```

### 💻 Задание

Настрой полноценный Alertmanager:

1. **Добавь Alertmanager в docker-compose.yml**:
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
```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    # Critical alerts
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true
    
    # Warning alerts
    - match:
        severity: warning
      receiver: 'warning-alerts'
      continue: true

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:5001/webhook'
        send_resolved: true

  - name: 'critical-alerts'
    slack_configs:
      - channel: '#alerts-critical'
        title: '🚨 Critical Alert'
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}"
        send_resolved: true

  - name: 'warning-alerts'
    slack_configs:
      - channel: '#alerts-warning'
        title: '⚠️ Warning Alert'
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}"
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

3. **Обнови prometheus.yml**:
```yaml
# Добавь в секцию alerting
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

4. **Создай более продвинутые alerts** (обнови `alerts.yml`):
```yaml
groups:
  - name: slo_alerts
    rules:
    # SLO: 99.9% availability
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
        ) > 0.001
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }} (threshold: 0.1%)"
        runbook: "https://runbook.example.com/high-error-rate"

    # SLO: p99 latency < 500ms
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99,
          rate(http_request_duration_seconds_bucket[5m])
        ) > 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "p99 latency is {{ $value }}s (threshold: 0.5s)"

  - name: capacity_alerts
    rules:
    # Predict disk full
    - alert: DiskWillFillIn4Hours
      expr: |
        predict_linear(
          node_filesystem_avail_bytes{mountpoint="/"}[1h], 
          4 * 3600
        ) < 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Disk will be full soon"
        description: "Disk on {{ $labels.instance }} will be full in ~4 hours"
        action: "Clean up logs or expand disk"

    # Memory pressure
    - alert: HighMemoryPressure
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.95
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory pressure"
        description: "Memory usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

  - name: service_alerts
    rules:
    # Service down
    - alert: ServiceDown
      expr: up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Service {{ $labels.job }} is down"
        description: "{{ $labels.instance }} has been down for more than 1 minute"
        action: "Check service status and logs"

    # Container restarting
    - alert: ContainerRestarting
      expr: |
        rate(container_last_seen{name!=""}[5m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Container {{ $labels.name }} is restarting"
        description: "Container has restarted {{ $value }} times in the last 5 minutes"
```

5. **Запусти и протестируй**:
```bash
docker-compose up -d alertmanager

# Reload Prometheus config
curl -X POST http://localhost:9090/-/reload

# Проверь Alertmanager UI
http://localhost:9093

# Проверь alerts в Prometheus
http://localhost:9090/alerts
```

6. **Создай webhook receiver для тестирования** (`webhook_receiver.py`):
```python
from flask import Flask, request
import json

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json
    print("=" * 50)
    print("Alert received:")
    print(json.dumps(data, indent=2))
    print("=" * 50)
    
    for alert in data.get('alerts', []):
        status = alert['status']
        labels = alert['labels']
        annotations = alert['annotations']
        
        print(f"\nStatus: {status}")
        print(f"Alert: {labels.get('alertname')}")
        print(f"Severity: {labels.get('severity')}")
        print(f"Summary: {annotations.get('summary')}")
        print(f"Description: {annotations.get('description')}")
    
    return "OK", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
```

### 🚀 Бонус (новое)

**Настрой Silences и Inhibit rules**:

**Создай silence через API**:
```bash
# Silence alert на 2 часа
curl -XPOST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "HighCPUUsage",
        "isRegex": false
      }
    ],
    "startsAt": "2025-01-15T10:00:00Z",
    "endsAt": "2025-01-15T12:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance window"
  }'
```

**Продвинутые inhibit rules**:
```yaml
inhibit_rules:
  # Если сервис down, не алертить про его метрики
  - source_match:
      alertname: 'ServiceDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']
  
  # Если critical alert firing, подавить warning
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

**Настрой PagerDuty integration**:
```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
```

---

## Модуль 6: Синтетический мониторинг и Uptime (20 минут)

### 🎯 Напоминалка

**Synthetic monitoring** - проактивная проверка доступности и производительности:
```
Real User Monitoring (RUM)  - Реальные пользователи
Synthetic Monitoring        - Роботы/боты проверяют
```

**Типы проверок:**
```
- HTTP/HTTPS endpoints
- SSL certificate expiration
- DNS resolution
- TCP port availability
- API response validation
- Multi-step transactions
- Browser scenarios (Selenium)
```

**Blackbox Exporter** - для проверки endpoints:
```
Protocols:
- HTTP/HTTPS
- TCP
- ICMP (ping)
- DNS
- GRPC
```

### 💻 Задание

Настрой синтетический мониторинг:

1. **Добавь Blackbox Exporter в docker-compose.yml**:
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
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"test": "data"}'

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"

  dns_example:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"

  ssl_expire:
    prober: http
    timeout: 5s
    http:
      method: GET
      tls_config:
        insecure_skip_verify: false
```

3. **Добавь в prometheus.yml**:
```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://google.com
        - https://github.com
        - http://localhost:3000  # Grafana
        - http://localhost:9090  # Prometheus
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox-ssl'
    metrics_path: /probe
    params:
      module: [ssl_expire]
    static_configs:
      - targets:
        - https://google.com
        - https://github.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

4. **Создай uptime alerts** (добавь в `alerts.yml`):
```yaml
groups:
  - name: uptime_alerts
    rules:
    # Endpoint down
    - alert: EndpointDown
      expr: probe_success == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Endpoint {{ $labels.instance }} is down"
        description: "Endpoint has been down for more than 2 minutes"

    # SSL certificate expires soon
    - alert: SSLCertExpiringSoon
      expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "SSL certificate expiring soon"
        description: "SSL certificate for {{ $labels.instance }} expires in {{ $value }} days"

    # Slow response
    - alert: SlowResponse
      expr: probe_duration_seconds > 5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Slow response from {{ $labels.instance }}"
        description: "Response time is {{ $value }}s (threshold: 5s)"

    # HTTP status code error
    - alert: HTTPStatusError
      expr: probe_http_status_code >= 400
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "HTTP error on {{ $labels.instance }}"
        description: "HTTP status code is {{ $value }}"
```

5. **Запусти и проверь**:
```bash
docker-compose up -d blackbox-exporter

# Проверь metrics
curl "http://localhost:9115/probe?target=https://google.com&module=http_2xx"

# Reload Prometheus
curl -X POST http://localhost:9090/-/reload
```

6. **Создай Uptime Dashboard в Grafana**:

**Panel 1: Service Availability**
```promql
avg_over_time(probe_success[24h]) * 100
```
Visualization: Stat
Unit: Percent (0-100)

**Panel 2: Response Time**
```promql
probe_duration_seconds
```
Visualization: Time series

**Panel 3: SSL Certificate Days Left**
```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400
```
Visualization: Gauge
Thresholds: Green >30, Yellow 15-30, Red <15

**Panel 4: Uptime Table**
```promql
probe_success
```
Visualization: Table
Transform: Add field from calculation → "Uptime" → `$probe_success * 100`

### 🚀 Бонус (новое)

**Создай multi-step transaction check** с использованием Selenium:

**synthetic_test.py**:
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from prometheus_client import Gauge, Counter, start_http_server
import time

# Metrics
transaction_duration = Gauge('transaction_duration_seconds', 'Transaction duration')
transaction_success = Counter('transaction_success_total', 'Successful transactions')
transaction_failure = Counter('transaction_failure_total', 'Failed transactions')

def run_transaction():
    """Simulate user transaction"""
    start = time.time()
    
    try:
        options = Options()
        options.add_argument('--headless')
        driver = webdriver.Chrome(options=options)
        
        # Step 1: Navigate
        driver.get('https://example.com')
        time.sleep(1)
        
        # Step 2: Find element
        element = driver.find_element(By.TAG_NAME, 'h1')
        assert 'Example' in element.text
        
        # Step 3: Click (if needed)
        # link = driver.find_element(By.LINK_TEXT, 'More information...')
        # link.click()
        
        driver.quit()
        
        duration = time.time() - start
        transaction_duration.set(duration)
        transaction_success.inc()
        print(f"✅ Transaction successful in {duration:.2f}s")
        
    except Exception as e:
        transaction_failure.inc()
        print(f"❌ Transaction failed: {e}")

if __name__ == '__main__':
    start_http_server(8001)
    print("Synthetic monitoring started on port 8001")
    
    while True:
        run_transaction()
        time.sleep(60)  # Run every minute
```

**Создай API endpoint checker**:
```python
# api_checker.py
import requests
import time
from prometheus_client import Histogram, Counter, Gauge, start_http_server

api_request_duration = Histogram('api_request_duration_seconds', 
                                 'API request duration',
                                 ['endpoint', 'method'])
api_request_total = Counter('api_request_total',
                           'Total API requests',
                           ['endpoint', 'method', 'status'])
api_request_errors = Counter('api_request_errors_total',
                            'Total API errors',
                            ['endpoint', 'error_type'])

endpoints = [
    {'url': 'https://api.github.com', 'method': 'GET'},
    {'url': 'https://jsonplaceholder.typicode.com/posts/1', 'method': 'GET'},
]

def check_endpoint(endpoint):
    url = endpoint['url']
    method = endpoint['method']
    
    try:
        start = time.time()
        
        if method == 'GET':
            response = requests.get(url, timeout=10)
        elif method == 'POST':
            response = requests.post(url, json={}, timeout=10)
        
        duration = time.time() - start
        
        api_request_duration.labels(endpoint=url, method=method).observe(duration)
        api_request_total.labels(endpoint=url, method=method, status=response.status_code).inc()
        
        print(f"✅ {method} {url}: {response.status_code} ({duration:.2f}s)")
        
    except requests.Timeout:
        api_request_errors.labels(endpoint=url, error_type='timeout').inc()
        print(f"⏱️ {method} {url}: Timeout")
    except requests.ConnectionError:
        api_request_errors.labels(endpoint=url, error_type='connection').inc()
        print(f"🔌 {method} {url}: Connection error")
    except Exception as e:
        api_request_errors.labels(endpoint=url, error_type='other').inc()
        print(f"❌ {method} {url}: {e}")

if __name__ == '__main__':
    start_http_server(8002)
    print("API checker started on port 8002")
    
    while True:
        for endpoint in endpoints:
            check_endpoint(endpoint)
        time.sleep(30)
```

---

## Модуль 7: Application Performance Monitoring (APM) (20 минут)

### 🎯 Напоминалка

**APM компоненты:**
```
- Distributed Tracing  - Трассировка запросов между сервисами
- Profiling           - Анализ производительности кода
- Error Tracking      - Отслеживание ошибок
- Transaction Monitoring - Мони