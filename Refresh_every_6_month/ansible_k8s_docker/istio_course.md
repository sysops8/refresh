# Istio Service Mesh: Практический курс для DevOps

**Цель:** Освоить Istio Service Mesh за 3-4 часа практики и внедрить в production.

**Формат:** Каждый раздел состоит из:
1. **Краткой теории (Напоминалка)**: Самое главное о компонентах и концепциях
2. **Практического задания**: Реальная задача, которую нужно решить
3. **Бонусного задания (для роста)**: Продвинутые техники и паттерны

**Предварительные требования:**
- Kubernetes кластер (minikube, kind, k3s или облачный)
- kubectl установлен и настроен
- Базовые знания Kubernetes (Pods, Services, Deployments)
- Helm 3.x установлен
- 4+ CPU и 8GB+ RAM для кластера

---

## Модуль 1: Введение в Service Mesh и установка Istio (30 минут)

### 🎯 Напоминалка

**Что такое Service Mesh:**
```
Service Mesh - это инфраструктурный слой для управления 
коммуникацией между микросервисами:

├── Traffic Management    # Маршрутизация, балансировка
├── Security             # mTLS, авторизация
├── Observability        # Метрики, трейсинг, логи
└── Resilience          # Retry, timeout, circuit breaker
```

**Архитектура Istio:**
```
Control Plane (istiod):
├── Pilot           # Управление трафиком, service discovery
├── Citadel         # Управление сертификатами, mTLS
└── Galley          # Конфигурация и валидация

Data Plane:
└── Envoy Proxy     # Sidecar прокси в каждом Pod'е
    ├── Перехват трафика (iptables)
    ├── Применение политик
    └── Сбор телеметрии
```

**Ключевые концепции:**
```yaml
VirtualService     # Правила маршрутизации (L7)
DestinationRule    # Политики балансировки, подмножества
Gateway           # Входящий/исходящий трафик
ServiceEntry      # Внешние сервисы
Sidecar          # Конфигурация прокси
PeerAuthentication # mTLS настройки
AuthorizationPolicy # RBAC для сервисов
```

**Когда использовать Service Mesh:**
✅ Микросервисная архитектура (10+ сервисов)
✅ Нужен mTLS между сервисами
✅ Сложная маршрутизация (canary, A/B)
✅ Централизованная observability
✅ Rate limiting, circuit breakers

❌ Монолит или 2-3 сервиса
❌ Простая архитектура
❌ Ограниченные ресурсы

### 💻 Задание

**1. Установка Istio:**

```bash
# Скачать istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0  # или актуальная версия
export PATH=$PWD/bin:$PATH

# Проверка
istioctl version

# Установка с профилем demo (для обучения)
istioctl install --set profile=demo -y

# Для production используй:
# istioctl install --set profile=default -y

# Проверка установки
kubectl get pods -n istio-system
kubectl get svc -n istio-system

# Должны быть запущены:
# - istiod (control plane)
# - istio-ingressgateway (входящий трафик)
# - istio-egressgateway (исходящий трафик)
```

**2. Создай namespace с автоматическим sidecar injection:**

```bash
# Создать namespace
kubectl create namespace istio-demo

# Включить автоматический injection
kubectl label namespace istio-demo istio-injection=enabled

# Проверка
kubectl get namespace -L istio-injection
```

**3. Разверни тестовое приложение (Bookinfo):**

```bash
# Разверни sample приложение
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo

# Проверка - у каждого Pod должно быть 2 контейнера (app + envoy)
kubectl get pods -n istio-demo

# Проверка, что приложение работает
kubectl exec -n istio-demo "$(kubectl get pod -n istio-demo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

**4. Создай Gateway для внешнего доступа:**

```bash
# Применить Gateway и VirtualService
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo

# Получить EXTERNAL-IP
kubectl get svc istio-ingressgateway -n istio-system

# Для minikube
export INGRESS_HOST=$(minikube ip)
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

# Тестирование
curl -s "http://${GATEWAY_URL}/productpage" | grep -o "<title>.*</title>"

# Или открой в браузере
echo "http://${GATEWAY_URL}/productpage"
```

**5. Установка Kiali, Prometheus, Grafana, Jaeger:**

```bash
# Установить addons для observability
kubectl apply -f samples/addons

# Проверка
kubectl get pods -n istio-system

# Доступ к Kiali (Service Mesh Dashboard)
istioctl dashboard kiali

# Доступ к Grafana
istioctl dashboard grafana

# Доступ к Jaeger (distributed tracing)
istioctl dashboard jaeger

# Или через port-forward
kubectl port-forward -n istio-system svc/kiali 20001:20001
kubectl port-forward -n istio-system svc/grafana 3000:3000
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686
```

### 🚀 Бонус (новое)

**1. Установка с кастомным профилем:**

```bash
# Создать custom profile
cat <<EOF > custom-profile.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  meshConfig:
    accessLogFile: /dev/stdout
    enableTracing: true
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        service:
          type: LoadBalancer
EOF

# Установить с custom profile
istioctl install -f custom-profile.yaml -y
```

**2. Проверка конфигурации:**

```bash
# Анализ конфигурации Istio
istioctl analyze -n istio-demo

# Проверка Envoy конфигурации в Pod
istioctl proxy-config cluster <pod-name> -n istio-demo
istioctl proxy-config listener <pod-name> -n istio-demo
istioctl proxy-config route <pod-name> -n istio-demo

# Проверка синхронизации
istioctl proxy-status
```

**3. Использование Operator для управления:**

```bash
# Установка Istio Operator
istioctl operator init

# Создание IstioOperator CR
kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-control-plane
spec:
  profile: demo
EOF

# Проверка
kubectl get iop -n istio-system
```

---

## Модуль 2: Traffic Management (40 минут)

### 🎯 Напоминалка

**VirtualService** - правила маршрутизации L7:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews                    # К какому сервису применяется
  http:
  - match:                     # Условия
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2             # Версия сервиса
  - route:                     # Дефолтный роут
    - destination:
        host: reviews
        subset: v1
```

**DestinationRule** - политики для destination:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:               # Применяется ко всем subsets
    loadBalancer:
      simple: RANDOM
  subsets:                     # Определение версий
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:             # Переопределяет для v2
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

**Gateway** - управление входящим трафиком:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway      # Использовать istio-ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "bookinfo.example.com"   # Или "*" для всех
```

**Типы маршрутизации:**
```yaml
# Weight-based (процентное распределение)
route:
- destination:
    host: reviews
    subset: v1
  weight: 80
- destination:
    host: reviews
    subset: v2
  weight: 20

# Header-based (по заголовкам)
match:
- headers:
    user-agent:
      regex: ".*Chrome.*"

# URI-based (по пути)
match:
- uri:
    prefix: "/api/v1"
- uri:
    exact: "/login"

# Query params
match:
- queryParams:
    version:
      exact: "v2"
```

**Traffic Policy:**
```yaml
trafficPolicy:
  connectionPool:
    tcp:
      maxConnections: 100
    http:
      http1MaxPendingRequests: 1
      http2MaxRequests: 100
      maxRequestsPerConnection: 2
  outlierDetection:
    consecutiveErrors: 5
    interval: 30s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
  loadBalancer:
    simple: LEAST_REQUEST      # ROUND_ROBIN, RANDOM, PASSTHROUGH
```

### 💻 Задание

**Задача:** Настроить Canary deployment с постепенным переходом на новую версию.

**1. Создай DestinationRule с subsets:**

```yaml
# destination-rule-reviews.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: istio-demo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

```bash
kubectl apply -f destination-rule-reviews.yaml
```

**2. Направь весь трафик на v1:**

```yaml
# virtual-service-reviews-v1.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
```

```bash
kubectl apply -f virtual-service-reviews-v1.yaml

# Проверка - все запросы идут на v1 (без звезд)
for i in {1..10}; do curl -s "http://${GATEWAY_URL}/productpage" | grep -o "glyphicon-star"; done
```

**3. Canary deployment - 10% на v2:**

```yaml
# virtual-service-reviews-canary-10.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

```bash
kubectl apply -f virtual-service-reviews-canary-10.yaml

# Проверка в Kiali
istioctl dashboard kiali
# Открой Graph -> Display -> Traffic Distribution
```

**4. Увеличь трафик на v2 до 50%:**

```yaml
# virtual-service-reviews-canary-50.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v2
      weight: 50
```

**5. Полный переход на v2:**

```yaml
# virtual-service-reviews-v2.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
      weight: 100
```

**6. Header-based routing:**

```yaml
# virtual-service-reviews-header.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

```bash
kubectl apply -f virtual-service-reviews-header.yaml

# Проверка - логин как jason покажет v2 (черные звезды)
# http://${GATEWAY_URL}/productpage
# Логин: jason, без пароля
```

### 🚀 Бонус (новое)

**1. Request timeout и retry:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
```

**2. Circuit Breaker:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
```

**3. Fault Injection (для тестирования):**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 7s
    route:
    - destination:
        host: reviews
        subset: v2
  - fault:
      abort:
        percentage:
          value: 10
        httpStatus: 500
    route:
    - destination:
        host: reviews
        subset: v1
```

**4. Traffic Mirroring (Shadow traffic):**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
    mirror:
      host: reviews
      subset: v2
    mirrorPercentage:
      value: 100
```

---

## Модуль 3: Security - mTLS и авторизация (40 минут)

### 🎯 Напоминалка

**Mutual TLS (mTLS):**
```
Без Service Mesh:
Client → [HTTP] → Server
❌ Незашифрованный трафик
❌ Нет аутентификации сервиса

С Istio mTLS:
Client → [Envoy mTLS] → [Envoy mTLS] → Server
✅ Автоматическое шифрование
✅ Автоматическая ротация сертификатов
✅ Взаимная аутентификация
```

**PeerAuthentication** - управление mTLS:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-demo
spec:
  mtls:
    mode: STRICT           # STRICT, PERMISSIVE, DISABLE
```

**Режимы mTLS:**
```yaml
STRICT       # Только mTLS (production)
PERMISSIVE   # mTLS и plain text (миграция)
DISABLE      # Отключить mTLS
```

**AuthorizationPolicy** - RBAC для сервисов:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW              # ALLOW, DENY, AUDIT, CUSTOM
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/reviews/*"]
```

**RequestAuthentication** - JWT валидация:
```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
spec:
  selector:
    matchLabels:
      app: productpage
  jwtRules:
  - issuer: "https://accounts.google.com"
    jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
```

### 💻 Задание

**Задача:** Настроить mTLS и политики авторизации между сервисами.

**1. Проверь текущий статус mTLS:**

```bash
# Проверка режима mTLS
kubectl get peerauthentication -n istio-demo

# Если нет настроек, по умолчанию PERMISSIVE

# Проверка в Kiali
istioctl dashboard kiali
# Settings -> Display -> Security (покажет замочки на соединениях)
```

**2. Включи STRICT mTLS для всего namespace:**

```yaml
# peer-auth-strict.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-demo
spec:
  mtls:
    mode: STRICT
```

```bash
kubectl apply -f peer-auth-strict.yaml

# Проверка
kubectl get peerauthentication -n istio-demo

# Проверка в Kiali - все соединения должны показывать замочки
```

**3. Проверь, что без mTLS нельзя подключиться:**

```bash
# Создай Pod без sidecar
kubectl run test --image=curlimages/curl -n istio-demo --rm -it -- sh

# Внутри Pod попробуй подключиться
curl http://productpage:9080/productpage

# Должна быть ошибка, т.к. требуется mTLS
exit
```

**4. Создай deny-all политику (default deny):**

```yaml
# authz-deny-all.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: istio-demo
spec:
  {}  # Пустая политика = запретить всё
```

```bash
kubectl apply -f authz-deny-all.yaml

# Проверка - приложение должно перестать работать
curl "http://${GATEWAY_URL}/productpage"
# Должен вернуть RBAC: access denied
```

**5. Разрешай трафик от productpage к reviews:**

```yaml
# authz-allow-productpage-reviews.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage-reviews
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
```

**6. Разрешай трафик от productpage к details:**

```yaml
# authz-allow-productpage-details.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage-details
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/bookinfo-productpage"]
```

**7. Разрешай трафик от reviews к ratings:**

```yaml
# authz-allow-reviews-ratings.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-reviews-ratings
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/bookinfo-reviews"]
```

**8. Разрешай входящий трафик на productpage:**

```yaml
# authz-allow-ingress-productpage.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-productpage
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
```

```bash
# Применить все политики
kubectl apply -f authz-allow-productpage-reviews.yaml
kubectl apply -f authz-allow-productpage-details.yaml
kubectl apply -f authz-allow-reviews-ratings.yaml
kubectl apply -f authz-allow-ingress-productpage.yaml

# Проверка - приложение должно работать
curl "http://${GATEWAY_URL}/productpage"
```

### 🚀 Бонус (новое)

**1. HTTP методы и paths:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-only-get
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
        paths: ["/reviews/*"]
```

**2. IP-based access control:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-specific-ips
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        ipBlocks: ["10.0.0.0/8", "192.168.0.0/16"]
```

**3. Custom conditions:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-with-header
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - when:
    - key: request.headers[x-custom-token]
      values: ["secret-token-123"]
```

**4. Namespace-level mTLS:**

```yaml
# Global mesh-wide STRICT mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

**5. AUDIT mode (логирование без блокировки):**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: audit-reviews
spec:
  selector:
    matchLabels:
      app: reviews
  action: AUDIT
  rules:
  - from:
    - source:
        notPrincipals: ["cluster.local/ns/istio-demo/sa/bookinfo-productpage"]
```

---

## Модуль 4: Observability - Метрики, трейсинг, логи (40 минут)

### 🎯 Напоминалка

**Три столпа Observability:**
```
Metrics     → Что происходит? (Prometheus/Grafana)
Traces      → Где проблема? (Jaeger/Zipkin)
Logs        → Почему? (Fluentd/ELK)
```

**Istio автоматически собирает:**
```yaml
Metrics (Prometheus):
├── Request count
├── Request duration
├── Request size
├── Response size
└── Response codes

Traces (Jaeger):
├── Span для каждого запроса
├── Связи между сервисами
└── Latency breakdown

Access Logs (Envoy):
└── Детальные логи каждого запроса
```

**Стандартные метрики Istio:**
```
istio_requests_total              # Общее количество запросов
istio_request_duration_seconds    # Длительность запросов
istio_request_bytes              # Размер запросов
istio_response_bytes             # Размер ответов
istio_tcp_connections_opened     # TCP соединения
```

**Grafana дашборды:**
```
Mesh Dashboard         # Общая картина mesh
Service Dashboard      # Метрики сервиса
Workload Dashboard     # Метрики workload
Performance Dashboard  # Производительность
Control Plane Dashboard # Состояние control plane
```

**Distributed Tracing концепции:**
```
Trace    # Полный путь запроса через систему
Span     # Единица работы в сервисе
Context  # Передается через headers (x-b3-*)
```

### 💻 Задание

**Задача:** Настроить полный стек observability и найти проблемы производительности.

**1. Включи access logging:**

```yaml
# configmap-istio.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
```

```bash
# Или через istioctl
istioctl install --set meshConfig.accessLogFile=/dev/stdout -y

# Проверка логов
kubectl logs -l app=productpage -n istio-demo -c istio-proxy --tail=10
```

**2. Генерация нагрузки для метрик:**

```bash
# Установка fortio для load testing
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml -n istio-demo

# Генерация трафика
export FORTIO_POD=$(kubectl get pods -n istio-demo -l app=fortio -o 'jsonpath={.items[0].metadata.name}')

kubectl exec "$FORTIO_POD" -n istio-demo -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -t 30s -loglevel Warning http://productpage:9080/productpage

# Или через curl loop
for i in {1..100}; do 
  curl -s "http://${GATEWAY_URL}/productpage" > /dev/null
  echo "Request $i"
  sleep 0.5
done
```

**3. Анализ в Grafana:**

```bash
# Открыть Grafana
istioctl dashboard grafana

# Дашборды для просмотра:
# 1. Istio Mesh Dashboard - общая картина
# 2. Istio Service Dashboard - метрики конкретного сервиса
# 3. Istio Workload Dashboard - метрики workload
# 4. Istio Performance Dashboard - latency, throughput

# Ключевые метрики:
# - Request rate (req/s)
# - Success rate (%)
# - P50, P90, P99 latency
# - Error rate
```

**4. Анализ в Kiali:**

```bash
# Открыть Kiali
istioctl dashboard kiali

# Функции Kiali:
# - Graph: визуализация трафика между сервисами
# - Applications/Workloads/Services: детали каждого компонента
# - Istio Config: валидация конфигурации
# - Distributed Tracing: интеграция с Jaeger

# Настройки Graph:
# Display -> Traffic Distribution (показать вес трафика)
# Display -> Security (показать mTLS)
# Display -> Response Time (показать latency)
```

**5. Distributed Tracing с Jaeger:**

```bash
# Открыть Jaeger
istioctl dashboard jaeger

# Поиск traces:
# Service: productpage.istio-demo
# Operation: all
# Lookback: 1h
# Limit: 20

# Анализ trace:
# - Общая длительность запроса
# - Breakdown по сервисам
# - Где самая большая задержка
# - Errors в цепочке
```

**6. Создай custom метрики:**

```yaml
# telemetry-metrics.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: istio-demo
spec:
  metrics:
  - providers:
    - name: prometheus
    dimensions:
      request_host: request.host
      destination_port: destination.port
      request_protocol: request.protocol
    overrides:
    - match:
        metric: REQUEST_COUNT
      tagOverrides:
        response_code:
          value: "response.code"
```

```bash
kubectl apply -f telemetry-metrics.yaml

# Проверка в Prometheus
istioctl dashboard prometheus
# Query: istio_requests_total{destination_service="reviews.istio-demo.svc.cluster.local"}
```

**7. Настрой алерты в Prometheus:**

```yaml
# prometheus-alert-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alert-rules
  namespace: istio-system
data:
  alert.rules: |
    groups:
    - name: istio
      interval: 10s
      rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
          /
          sum(rate(istio_requests_total[5m]))
          > 0.05
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le)
          ) > 1000
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P99 latency is {{ $value }}ms"
```

### 🚀 Бонус (новое)

**1. Sampling rate для трейсинг:**

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-control-plane
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100.0  # 100% для dev, 1-10% для prod
        zipkin:
          address: jaeger-collector.istio-system:9411
```

**2. Настройка OpenTelemetry:**

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: otel-tracing
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: opentelemetry
    randomSamplingPercentage: 100.0
    customTags:
      environment:
        literal:
          value: "production"
      version:
        header:
          name: "x-app-version"
```

**3. Access Log Format:**

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: access-logging
  namespace: istio-demo
spec:
  accessLogging:
  - providers:
    - name: envoy
    filter:
      expression: response.code >= 400
```

**4. Grafana дашборд для custom метрик:**

```json
{
  "dashboard": {
    "title": "Custom Service Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"reviews.istio-demo.svc.cluster.local\"}[5m])) by (destination_version)"
          }
        ]
      }
    ]
  }
}
```

---

## Модуль 5: Multi-Cluster и Advanced Patterns (45 минут)

### 🎯 Напоминалка

**Multi-Cluster топологии:**
```
Single Network:
├── Все кластеры в одной сети
├── Прямая связь между Pod'ами
└── Проще настроить

Multi Network:
├── Кластеры в разных сетях
├── Связь через Gateways
└── Более защищенно
```

**Multi-Primary:**
```
Cluster 1                  Cluster 2
├── Control Plane         ├── Control Plane
├── Data Plane            ├── Data Plane
└── Services              └── Services
     ↕ Service Discovery ↕
```

**Primary-Remote:**
```
Primary Cluster           Remote Cluster
├── Control Plane         ├── Data Plane only
├── Data Plane            └── Services
└── Services
     ↓ Config sync
```

**Egress Gateway** - контроль исходящего трафика:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: egress-gateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: tls
      protocol: TLS
    hosts:
    - "*.external.com"
    tls:
      mode: PASSTHROUGH
```

**ServiceEntry** - регистрация внешних сервисов:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - api.external.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

### 💻 Задание

**Задача:** Настроить доступ к внешним сервисам и egress контроль.

**1. Проверь текущий режим outbound:**

```bash
# По умолчанию Istio в ALLOW_ANY режиме
# (позволяет доступ к внешним сервисам)

# Проверка
kubectl get configmap istio -n istio-system -o yaml | grep -A 1 outboundTrafficPolicy

# Тест доступа к внешнему API
kubectl exec "$FORTIO_POD" -n istio-demo -c fortio -- /usr/bin/fortio curl https://httpbin.org/headers
```

**2. Переключи в REGISTRY_ONLY режим:**

```bash
# Только зарегистрированные сервисы
istioctl install --set profile=demo \
  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y

# Проверка - доступ должен быть заблокирован
kubectl exec "$FORTIO_POD" -n istio-demo -c fortio -- /usr/bin/fortio curl https://httpbin.org/headers
# Должна быть ошибка
```

**3. Зарегистрируй внешний сервис:**

```yaml
# service-entry-httpbin.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: httpbin-external
  namespace: istio-demo
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

```bash
kubectl apply -f service-entry-httpbin.yaml

# Проверка - теперь должно работать
kubectl exec "$FORTIO_POD" -n istio-demo -c fortio -- /usr/bin/fortio curl http://httpbin.org/headers
```

**4. Настрой Egress Gateway для контроля:**

```yaml
# egress-gateway-httpbin.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: egress-httpbin
  namespace: istio-demo
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.org
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-httpbin-through-egress
  namespace: istio-demo
spec:
  hosts:
  - httpbin.org
  gateways:
  - egress-httpbin
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - egress-httpbin
      port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
      weight: 100
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egressgateway-for-httpbin
  namespace: istio-demo
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: httpbin
```

```bash
kubectl apply -f egress-gateway-httpbin.yaml

# Проверка - трафик идет через egress gateway
kubectl logs -l istio=egressgateway -n istio-system -f
```

**5. TLS origination на Egress Gateway:**

```yaml
# egress-tls-origination.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: google-ext
  namespace: istio-demo
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: egress-google
  namespace: istio-demo
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - www.google.com
    tls:
      mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-google-through-egress
  namespace: istio-demo
spec:
  hosts:
  - www.google.com
  gateways:
  - egress-google
  - mesh
  tls:
  - match:
    - gateways:
      - mesh
      port: 443
      sniHosts:
      - www.google.com
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 443
  - match:
    - gateways:
      - egress-google
      port: 443
      sniHosts:
      - www.google.com
    route:
    - destination:
        host: www.google.com
        port:
          number: 443
      weight: 100
```

### 🚀 Бонус (новое)

**1. Locality Load Balancing:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-locality
spec:
  host: reviews.istio-demo.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
        - from: "us-central1/us-central1-a/*"
          to:
            "us-central1/us-central1-a/*": 80
            "us-central1/us-central1-b/*": 20
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**2. Multi-Cluster Service Discovery:**

```bash
# Установка на втором кластере
istioctl install --set profile=demo \
  --set values.global.multiCluster.clusterName=cluster2

# Включение endpoint discovery
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: istio-remote-secret-cluster2
  namespace: istio-system
  annotations:
    networking.istio.io/cluster: cluster2
type: Opaque
stringData:
  cluster2: |
    $(cat kubeconfig-cluster2)
EOF
```

**3. Sidecar Resource Optimization:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: istio-demo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  workloadSelector:
    labels:
      app: productpage
```

**4. Wasm Extensions:**

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: custom-header
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: productpage
  url: oci://docker.io/myrepo/custom-header:v1.0.0
  phase: AUTHN
  pluginConfig:
    header_name: "x-custom-header"
    header_value: "my-value"
```

---

## Модуль 6: Production Best Practices (40 минут)

### 🎯 Напоминалка

**Resource Requirements:**
```yaml
Pilot (istiod):
  CPU: 500m-2000m
  Memory: 2Gi-8Gi
  
Ingress Gateway:
  CPU: 100m-2000m
  Memory: 128Mi-1Gi
  
Egress Gateway:
  CPU: 100m-1000m
  Memory: 128Mi-512Mi

Sidecar (на каждый Pod):
  CPU: 10m-2000m
  Memory: 40Mi-1Gi
```

**High Availability:**
```yaml
Control Plane:
├── 2+ replicas istiod
├── HPA для масштабирования
└── PodDisruptionBudget

Gateways:
├── 2+ replicas
├── Anti-affinity для распределения
├── HPA
└── PodDisruptionBudget
```

**Security Checklist:**
```
✅ STRICT mTLS везде
✅ REGISTRY_ONLY для egress
✅ AuthorizationPolicy default deny
✅ Минимальные RBAC права
✅ Regular certificate rotation
✅ Security scanning образов
✅ Network policies
```

**Performance Tuning:**
```yaml
# Reduce sidecar resource footprint
--set values.global.proxy.resources.requests.cpu=10m
--set values.global.proxy.resources.requests.memory=40Mi

# Reduce config push frequency
--set pilot.env.PILOT_PUSH_THROTTLE=100

# Limit concurrent connections
--set values.global.proxy.concurrency=2
```

### 💻 Задание

**Задача:** Подготовить production-ready конфигурацию Istio.

**1. Production IstioOperator:**

```yaml
# istio-production.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-production
spec:
  profile: default
  
  # Mesh config
  meshConfig:
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 1.0  # 1% sampling
        zipkin:
          address: jaeger-collector.istio-system:9411
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
  
  # Components
  components:
    # Control Plane
    pilot:
      k8s:
        replicaCount: 2
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
          metrics:
          - type: Resource
            resource:
              name: cpu
              targetAverageUtilization: 80
        podDisruptionBudget:
          minAvailable: 1
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: istiod
                topologyKey: kubernetes.io/hostname
    
    # Ingress Gateway
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        replicaCount: 2
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1Gi
        hpaSpec:
          minReplicas: 2
          maxReplicas: 10
          metrics:
          - type: Resource
            resource:
              name: cpu
              targetAverageUtilization: 80
        podDisruptionBudget:
          minAvailable: 1
        service:
          type: LoadBalancer
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          - name: http2
            port: 80
            targetPort: 8080
          - name: https
            port: 443
            targetPort: 8443
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: istio-ingressgateway
                topologyKey: kubernetes.io/hostname
    
    # Egress Gateway
    egressGateways:
    - name: istio-egressgateway
      enabled: true
      k8s:
        replicaCount: 2
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
        podDisruptionBudget:
          minAvailable: 1
  
  # Global settings
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
          limits:
            cpu: 2000m
            memory: 1Gi
        logLevel: warning
        componentLogLevel: "misc:error"
      
      # mTLS
      mtls:
        enabled: true
        auto: true
```

```bash
istioctl install -f istio-production.yaml -y
```

**2. Namespace-level конфигурация:**

```yaml
# namespace-production.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
    environment: production
---
# Default STRICT mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# Default deny all
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}
---
# Sidecar optimization
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: production
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```

**3. Monitoring и alerting:**

```yaml
# service-monitor-istio.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: istio-system
spec:
  jobLabel: istio
  selector:
    matchExpressions:
    - key: istio
      operator: In
      values:
      - pilot
  endpoints:
  - port: http-monitoring
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-stats-monitor
  namespace: istio-system
spec:
  selector:
    matchExpressions:
    - key: istio-prometheus-ignore
      operator: DoesNotExist
  podMetricsEndpoints:
  - path: /stats/prometheus
    interval: 15s
```

**4. Upgrade strategy:**

```bash
# Canary upgrade
# 1. Установить новую версию control plane
istioctl install --set revision=1-20-0 -f istio-production.yaml -y

# 2. Проверка
kubectl get pods -n istio-system -L istio.io/rev

# 3. Постепенный rollout на namespace
kubectl label namespace production istio.io/rev=1-20-0 --overwrite
kubectl label namespace production istio-injection-

# 4. Restart workloads
kubectl rollout restart deployment -n production

# 5. Мониторинг в течение дней/недель

# 6. Удалить старый control plane
istioctl uninstall --revision=1-19-0 -y
```

### 🚀 Бонус (новое)

**1. Custom Metrics для HPA:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: reviews-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: reviews-v1
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: istio_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

**2. Disaster Recovery:**

```bash
# Backup Istio configuration
kubectl get istiooperator -n istio-system -o yaml > istio-operator-backup.yaml
kubectl get virtualservice,destinationrule,gateway,serviceentry -A -o yaml > istio-config-backup.yaml

# Backup secrets (certificates)
kubectl get secrets -n istio-system -l istio.io/config=true -o yaml > istio-secrets-backup.yaml
```

**3. Cost Optimization:**

```yaml
# Reduce sidecar footprint для non-critical workloads
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-sidecar-injector
  namespace: istio-system
data:
  values: |
    global:
      proxy:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

**4. Advanced Circuit Breaking:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-circuit-breaker
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
```

---

## Финальный проект (60 минут)

### Задача: Развернуть production-ready микросервисное приложение с Istio

**Архитектура:**
```
Internet
    ↓
Istio Ingress Gateway (TLS)
    ↓
Frontend (React) → [mTLS] → Backend API (Node.js)
                              ↓ [mTLS]
                         Database Proxy
                              ↓
                         PostgreSQL
    ↓ [Egress Gateway]
External Payment API
```

**Требования:**

1. **Traffic Management:**
   - Canary deployment для Backend (90/10)
   - Header-based routing для beta users
   - Retry и timeout политики
   - Circuit breaker

2. **Security:**
   - STRICT mTLS между всеми сервисами
   - AuthorizationPolicy с default deny
   - Egress control для Payment API
   - TLS termination на Gateway

3. **Observability:**
   - Distributed tracing (100% sampling для dev)
   - Custom метрики для business KPIs
   - Access logging в JSON
   - Grafana дашборды

4. **Resilience:**
   - HPA для всех компонентов
   - PodDisruptionBudget
   - Health checks
   - Fault injection для тестирования

5. **Production готовность:**
   - 2+ replicas для критичных сервисов
   - Resource requests/limits
   - Anti-affinity rules
   - Backup стратегия

**Структура проекта:**

```
istio-production-app/
├── istio/
│   ├── installation/
│   │   └── istio-operator.yaml
│   ├── gateway/
│   │   ├── gateway.yaml
│   │   ├── virtual-service.yaml
│   │   └── tls-secret.yaml
│   ├── traffic/
│   │   ├── destination-rules.yaml
│   │   ├── virtual-services-canary.yaml
│   │   └── service-entries.yaml
│   ├── security/
│   │   ├── peer-authentication.yaml
│   │   ├── authorization-policies.yaml
│   │   └── request-authentication.yaml
│   └── observability/
│       ├── telemetry.yaml
│       └── service-monitors.yaml
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   ├── backend/
│   │   ├── deployment-v1.yaml
│   │   ├── deployment-v2.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   └── database/
│       ├── statefulset.yaml
│       └── service.yaml
├── scripts/
│   ├── install.sh
│   ├── deploy-app.sh
│   ├── canary-promote.sh
│   └── rollback.sh
└── README.md
```

**Начни с установки:**

```bash
# 1. Установить Istio
kubectl apply -f istio/installation/istio-operator.yaml

# 2. Создать namespace
kubectl create namespace production
kubectl label namespace production istio-injection=enabled

# 3. Применить security baseline
kubectl apply -f istio/security/

# 4. Развернуть приложения
kubectl apply -f apps/

# 5. Настроить traffic management
kubectl apply -f istio/traffic/

# 6. Настроить Gateway
kubectl apply -f istio/gateway/

# 7. Настроить observability
kubectl apply -f istio/observability/
```

**Проверочный чеклист:**

- [ ] Istio установлен и istiod запущен
- [ ] Все Pod'ы имеют sidecar (2/2 containers)
- [ ] mTLS STRICT включен
- [ ] AuthorizationPolicy работает
- [ ] Gateway принимает трафик
- [ ] Canary routing 90/10 работает
- [ ] Metrics собираются в Prometheus
- [ ] Traces видны в Jaeger
- [ ] Grafana дашборды показывают данные
- [ ] Kiali показывает topology
- [ ] HPA масштабирует при нагрузке
- [ ] Circuit breaker срабатывает при ошибках
- [ ] Egress Gateway контролирует внешний трафик

---

## Справочная секция

### Istio CLI команды

```bash
# Установка
istioctl install --set profile=demo
istioctl install --set profile=default
istioctl install -f istio-operator.yaml

# Проверка
istioctl version
istioctl proxy-status
istioctl analyze -n istio-demo

# Debugging
istioctl proxy-config cluster <pod> -n <namespace>
istioctl proxy-config listener <pod> -n <namespace>
istioctl proxy-config route <pod> -n <namespace>
istioctl proxy-config endpoint <pod> -n <namespace>
istioctl proxy-config bootstrap <pod> -n <namespace>
istioctl proxy-config secret <pod> -n <namespace>

# Logs
istioctl proxy-config log <pod> --level debug
istioctl proxy-config log <pod> --level warning

# Dashboards
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
istioctl dashboard prometheus
istioctl dashboard envoy <pod>.<namespace>

# Inject/Uninject
istioctl kube-inject -f deployment.yaml | kubectl apply -f -
kubectl label namespace <namespace> istio-injection=enabled
kubectl label namespace <namespace> istio-injection-

# Upgrade
istioctl upgrade
istioctl manifest generate > manifest.yaml

# Validation
istioctl validate -f manifest.yaml
istioctl analyze -n <namespace>

# Experimental
istioctl experimental describe pod <pod> -n <namespace>
istioctl experimental wait --for=distribution Gateway <gateway>
```

### Troubleshooting Guide

**Pod не получает sidecar:**
```bash
# Проверка namespace label
kubectl get namespace <namespace> -L istio-injection

# Проверка webhook
kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml

# Проверка istiod logs
kubectl logs -n istio-system -l app=istiod

# Manual injection
istioctl kube-inject -f deployment.yaml | kubectl apply -f -
```

**Traffic не идет через Istio:**
```bash
# Проверка sidecar status
istioctl proxy-status

# Проверка Envoy config
istioctl proxy-config listener <pod> -n <namespace>
istioctl proxy-config route <pod> -n <namespace>

# Проверка Virtual Service
kubectl describe virtualservice <name> -n <namespace>

# Проверка в Kiali
istioctl dashboard kiali
```

**mTLS проблемы:**
```bash
# Проверка PeerAuthentication
kubectl get peerauthentication -A

# Проверка certificates
istioctl proxy-config secret <pod> -n <namespace>

# Проверка в Kiali (должны быть замочки)
istioctl dashboard kiali

# Test connectivity
kubectl exec <pod> -n <namespace> -c istio-proxy -- \
  curl -v http://service:port
```

**Gateway не работает:**
```bash
# Проверка Gateway
kubectl get gateway -A
kubectl describe gateway <name> -n <namespace>

# Проверка ingress gateway Pod
kubectl get pods -n istio-system -l istio=ingressgateway
kubectl logs -n istio-system -l istio=ingressgateway

# Получить EXTERNAL-IP
kubectl get svc istio-ingressgateway -n istio-system

# Проверка Virtual Service
kubectl describe virtualservice <name> -n <namespace>

# Test
curl -v http://<GATEWAY-IP>/path
```

**High memory/CPU usage:**
```bash
# Проверка ресурсов
kubectl top pods -n istio-system
kubectl top pods -n <namespace>

# Reduce sidecar resources
kubectl patch deployment <name> -n <namespace> -p '
{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "sidecar.istio.io/proxyCPU": "10m",
          "sidecar.istio.io/proxyMemory": "128Mi"
        }
      }
    }
  }
}'

# Optimize control plane
istioctl install --set values.pilot.resources.requests.memory=512Mi
```

**Envoy не синхронизируется:**
```bash
# Проверка sync status
istioctl proxy-status

# Если NOT SYNCED, проверить istiod
kubectl logs -n istio-system -l app=istiod

# Force refresh
kubectl delete pod <pod> -n <namespace>

# Check version mismatch
kubectl get pods -n istio-system -o jsonpath='{.items[*].spec.containers[*].image}'
```

### Полезные ресурсы

**Официальная документация:**
- https://istio.io/latest/docs/
- https://istio.io/latest/docs/ops/best-practices/
- https://istio.io/latest/docs/reference/config/

**Community:**
- Istio Slack: https://slack.istio.io/
- GitHub Issues: https://github.com/istio/istio/issues
- Discuss: https://discuss.istio.io/

**Обучение:**
- Istio Basics: https://academy.tetrate.io/
- Istio Fundamentals: https://www.katacoda.com/istio
- Solo.io workshops: https://www.solo.io/workshops/

**Полезные инструменты:**
- Kiali - Service mesh visualization
- Jaeger - Distributed tracing
- Prometheus - Metrics
- Grafana - Dashboards
- Fortio - Load testing

### YAML шаблоны

**Минимальный VirtualService:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service
```

**Минимальный DestinationRule:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

**Минимальный Gateway:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

**Минимальный ServiceEntry:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - api.external.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

### Best Practices чеклист

**Development:**
- ✅ Используй profile: demo для обучения
- ✅ Включи все addons (Kiali, Grafana, Jaeger)
- ✅ 100% sampling для traces
- ✅ Access logging включен
- ✅ ALLOW_ANY для egress (проще разработка)

**Staging:**
- ✅ Используй profile: default
- ✅ PERMISSIVE mTLS (миграция)
- ✅ 10% sampling для traces
- ✅ REGISTRY_ONLY для egress
- ✅ Тестируй AuthorizationPolicy
- ✅ Load testing с реальными нагрузками

**Production:**
- ✅ Используй profile: default или custom
- ✅ STRICT mTLS везде
- ✅ 1-5% sampling для traces
- ✅ REGISTRY_ONLY для egress
- ✅ Default deny AuthorizationPolicy
- ✅ 2+ replicas для control plane
- ✅ 2+ replicas для gateways
- ✅ HPA для всех компонентов
- ✅ PodDisruptionBudget
- ✅ Resource requests/limits оптимизированы
- ✅ Monitoring и alerting настроены
- ✅ Backup конфигурации
- ✅ Canary upgrade стратегия
- ✅ Disaster recovery план

**Security:**
- ✅ STRICT mTLS
- ✅ REGISTRY_ONLY egress
- ✅ Default deny Authorization
- ✅ Least privilege RBAC
- ✅ Regular security audits
- ✅ Certificate rotation настроена
- ✅ Network policies
- ✅ Image scanning

**Performance:**
- ✅ Sidecar resources оптимизированы
- ✅ Connection pooling настроен
- ✅ Circuit breakers для resilience
- ✅ Retry с exponential backoff
- ✅ Timeouts настроены адекватно
- ✅ Locality load balancing
- ✅ Compression включен где нужно

### Метрики для мониторинга

**Control Plane:**
```
pilot_xds_pushes           # Config pushes
pilot_conflict_outbound    # Conflicting configs
pilot_total_xds_rejects    # Rejected configs
galley_validation_passed   # Validation success
citadel_server_csr_count   # Certificate requests
```

**Data Plane:**
```
istio_requests_total                    # Request count
istio_request_duration_milliseconds    # Latency
istio_request_bytes                    # Request size
istio_response_bytes                   # Response size
istio_tcp_connections_opened           # TCP connections
envoy_cluster_upstream_cx_active       # Active connections
envoy_cluster_upstream_cx_overflow     # Connection overflow
```

**Custom metrics query examples:**
```promql
# Request rate per service
sum(rate(istio_requests_total[5m])) by (destination_service_name)

# Error rate
sum(rate(istio_requests_total{response_code=~"5.."}[5m])) 
/ 
sum(rate(istio_requests_total[5m]))

# P99 latency
histogram_quantile(0.99, 
  sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le, destination_service_name)
)

# Success rate
sum(rate(istio_requests_total{response_code!~"5.."}[5m])) by (destination_service_name)
/ 
sum(rate(istio_requests_total[5m])) by (destination_service_name)
```

### Migration Strategy

**От без Service Mesh к Istio:**

**Phase 1: Preparation (1-2 недели)**
```bash
# 1. Audit текущей архитектуры
# 2. Identify критичные сервисы
# 3. План миграции (canary per namespace)
# 4. Setup observability stack
# 5. Training команды
```

**Phase 2: Pilot (2-4 недели)**
```bash
# 1. Установить Istio с PERMISSIVE mTLS
istioctl install --set profile=default \
  --set meshConfig.mtls.mode=PERMISSIVE

# 2. Выбрать non-critical namespace для pilot
kubectl label namespace dev istio-injection=enabled

# 3. Rollout постепенно
kubectl rollout restart deployment -n dev

# 4. Мониторинг метрик, errors, latency
# 5. Collect feedback
```

**Phase 3: Gradual Rollout (1-3 месяца)**
```bash
# По одному namespace за раз:
# 1. Enable injection
# 2. Restart workloads
# 3. Monitor 1-2 недели
# 4. Fix issues
# 5. Move to next namespace

# Priority order:
# dev → staging → non-critical prod → critical prod
```

**Phase 4: Strict mTLS (1-2 недели)**
```bash
# После миграции всех namespace
# 1. Включить STRICT mTLS постепенно
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

# 2. Monitor для проблем
# 3. Fix external integrations
```

**Phase 5: Authorization & Advanced Features (ongoing)**
```bash
# 1. Authorization policies
# 2. Advanced traffic management
# 3. Egress control
# 4. Multi-cluster если нужно
```

### Cost Optimization

**Reduce Control Plane costs:**
```yaml
# Minimal pilot resources
spec:
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
```

**Reduce Data Plane costs:**
```yaml
# Aggressive sidecar limits
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
          limits:
            cpu: 100m
            memory: 128Mi
```

**Selective injection:**
```yaml
# Только для Pod'ов, которым нужен mesh
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "true"  # или "false"
```

**Namespace-level control:**
```bash
# Не все namespace нужен Istio
kubectl label namespace monitoring istio-injection=disabled
kubectl label namespace kube-system istio-injection=disabled
```

---

## Заключение

Поздравляю! Ты прошел курс по Istio Service Mesh.

**Следующие шаги:**
1. Практикуй регулярно - разверни Istio в своем проекте
2. Изучай смежные технологии: Envoy, eBPF, Cilium
3. Следи за release notes - Istio быстро развивается
4. Получи сертификацию Istio от Tetrate или Solo.io
5. Делись знаниями - помогай новичкам в community

**Помни:**
- Service Mesh - это инструмент, не silver bullet
- Начинай с простого, усложняй постепенно
- Observability критична для debugging
- Security должна быть приоритетом
- Performance testing обязателен
- Documentation спасает команду

**Когда НЕ использовать Istio:**
- ❌ Простая архитектура (1-3 сервиса)
- ❌ Монолит без планов на микросервисы
- ❌ Очень ограниченные ресурсы
- ❌ Команда не готова к complexity
- ❌ Нет времени на proper setup

**Когда использовать Istio:**
- ✅ 10+ микросервисов
- ✅ Нужен централизованный контроль трафика
- ✅ Требуется mTLS между сервисами
- ✅ Сложные deployment стратегии (canary, A/B)
- ✅ Multi-cluster архитектура
- ✅ Compliance требования
- ✅ Команда готова инвестировать в обучение

**Альтернативы Istio:**
- **Linkerd** - легче, проще, меньше функций
- **Consul Connect** - если уже используете Consul
- **AWS App Mesh** - для AWS native решений
- **Cilium Service Mesh** - eBPF-based, высокая производительность
- **Kuma** - от Kong, CNCF проект

Проходи этот курс каждые 6-12 месяцев, чтобы оставаться в форме. Istio быстро развивается, и каждый раз ты будешь узнавать что-то новое!

Happy Service Meshing! 🕸️🚀

---

**Версия курса:** 1.0  
**Istio версия:** 1.20.x  
**Дата:** Декабрь 2024  
**Автор:** DevOps Community

**Changelog:**
- v1.0 - Initial release (Istio 1.20)

**TODO для следующих версий:**
- [ ] Ambient Mesh (sidecar-less mode)
- [ ] WebAssembly extensions
- [ ] Istio + GitOps (ArgoCD/Flux)
- [ ] Service Mesh Interface (SMI)
- [ ] Cost analysis tools
- [ ] Production war stories