# Zabbix Refresh: Ежегодный/Полугодовой курс для DevOps

**Цель:** Освежить в памяти ключевые концепции Zabbix за 2-3 часа практики и узнать 1-2 новые продвинутые техники.

**Формат:** Каждый раздел состоит из:
1. **Краткой теории (Напоминалка)**: Самое главное, что вы могли забыть
2. **Практического задания**: Реальная задача, которую нужно решить
3. **Бонусного задания (для роста)**: Задача посложнее или с использованием новой фичи

**Предварительные требования:**
- Доступ к Zabbix серверу (6.0+) или возможность развернуть через Docker
- Базовые знания Linux
- SSH доступ к тестовым хостам

---

## Модуль 1: Архитектура и базовая настройка (20 минут)

### 🎯 Напоминалка

**Архитектура Zabbix:**
```
Zabbix Server
├── Database (MySQL/PostgreSQL)
├── Web Frontend (Apache/Nginx + PHP)
└── Zabbix Server Process
    ├── Pollers (сбор данных)
    ├── Trappers (прием данных)
    ├── Alerters (отправка уведомлений)
    ├── Timers (триггеры времени)
    └── Escalators (эскалация проблем)

Zabbix Agent (на мониторимых хостах)
├── Active checks (агент инициирует)
└── Passive checks (сервер опрашивает)

Zabbix Proxy (опционально)
└── Промежуточный сборщик для удаленных локаций
```

**Основные компоненты:**
- **Host** - устройство/сервер для мониторинга
- **Item** - метрика для сбора (CPU, memory, disk, etc.)
- **Trigger** - условие для генерации проблемы
- **Action** - что делать при срабатывании триггера
- **Template** - набор items/triggers для переиспользования
- **Host Group** - логическая группировка хостов

**Типы мониторинга:**
```
Zabbix Agent    # Активный/пассивный агент
SNMP            # Сетевое оборудование
IPMI            # Аппаратный мониторинг серверов
JMX             # Java приложения
HTTP Agent      # REST API мониторинг
Database        # Прямые запросы к БД
SSH/Telnet      # Выполнение команд
```

**Быстрая установка через Docker:**
```bash
# Docker Compose для Zabbix 6.4
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pwd
      POSTGRES_DB: zabbix
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - zabbix-net

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-6.4-latest
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pwd
      POSTGRES_DB: zabbix
      ZBX_ENABLE_SNMP_TRAPS: "true"
    depends_on:
      - postgres
    ports:
      - "10051:10051"
    volumes:
      - zabbix-server-data:/var/lib/zabbix
    networks:
      - zabbix-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-6.4-latest
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pwd
      POSTGRES_DB: zabbix
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: Asia/Almaty
    depends_on:
      - postgres
      - zabbix-server
    ports:
      - "8080:8080"
    networks:
      - zabbix-net

  zabbix-agent:
    image: zabbix/zabbix-agent2:alpine-6.4-latest
    environment:
      ZBX_HOSTNAME: "Zabbix server"
      ZBX_SERVER_HOST: zabbix-server
    privileged: true
    pid: "host"
    networks:
      - zabbix-net

volumes:
  postgres-data:
  zabbix-server-data:

networks:
  zabbix-net:
    driver: bridge
EOF

# Запуск
docker-compose up -d

# Проверка
docker-compose ps
```

**Доступ к веб-интерфейсу:**
```
URL: http://localhost:8080
Login: Admin
Password: zabbix
```

**Установка Zabbix Agent на Linux:**
```bash
# Ubuntu/Debian
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
apt update
apt install zabbix-agent2

# CentOS/RHEL
rpm -Uvh https://repo.zabbix.com/zabbix/6.4/rhel/8/x86_64/zabbix-release-6.4-1.el8.noarch.rpm
dnf clean all
dnf install zabbix-agent2

# Конфигурация
vi /etc/zabbix/zabbix_agent2.conf
# Server=<IP_ZABBIX_SERVER>
# ServerActive=<IP_ZABBIX_SERVER>
# Hostname=<HOSTNAME>

# Запуск
systemctl enable zabbix-agent2
systemctl start zabbix-agent2
systemctl status zabbix-agent2
```

**Основные файлы конфигурации:**
```bash
# Сервер
/etc/zabbix/zabbix_server.conf

# Агент
/etc/zabbix/zabbix_agent2.conf
/etc/zabbix/zabbix_agent2.d/  # Дополнительные конфиги

# Веб
/etc/zabbix/web/zabbix.conf.php

# Логи
/var/log/zabbix/zabbix_server.log
/var/log/zabbix/zabbix_agent2.log
```

**Проверка связи:**
```bash
# С сервера проверить агента
zabbix_get -s <agent_host> -k agent.ping

# Проверить версию агента
zabbix_get -s <agent_host> -k agent.version

# Тест конфигурации
zabbix_agent2 -t agent.hostname

# Просмотр доступных метрик
zabbix_agent2 -p
```

### 💻 Задание

Разверни тестовое окружение:

1. **Запусти Zabbix через Docker Compose**
   ```bash
   docker-compose up -d
   docker-compose ps
   docker-compose logs -f zabbix-server
   ```

2. **Войди в веб-интерфейс**
   - Открой http://localhost:8080
   - Логин: Admin, Пароль: zabbix
   - Смени пароль на безопасный

3. **Добавь первый хост**
   - Configuration → Hosts → Create host
   - Host name: TestServer
   - Templates: Linux by Zabbix agent
   - Agent: DNS name или IP адрес агента
   - Port: 10050

4. **Проверь работу мониторинга**
   - Monitoring → Latest data
   - Выбери TestServer
   - Проверь поступление данных (CPU, Memory, Disk)

5. **Создай простой триггер**
   - Configuration → Hosts → TestServer → Triggers
   - Name: High CPU usage
   - Expression: `avg(/TestServer/system.cpu.util,5m)>80`
   - Severity: Warning

### 🚀 Бонус (новое)

**Настрой Zabbix Agent 2 с плагинами:**

Zabbix Agent 2 поддерживает плагины для расширенного мониторинга:

```bash
# /etc/zabbix/zabbix_agent2.conf
Plugins.SystemRun.LogRemoteCommands=1

# Docker monitoring
Plugins.Docker.Endpoint=unix:///var/run/docker.sock

# Redis monitoring
Plugins.Redis.Uri=tcp://localhost:6379

# PostgreSQL monitoring
Plugins.PostgreSQL.Uri=tcp://localhost:5432

# Перезапуск
systemctl restart zabbix-agent2

# Проверка плагинов
zabbix_agent2 -t docker.info
zabbix_agent2 -t redis.ping
```

**Используй Zabbix API для автоматизации:**

```bash
# Получить authentication token
curl -X POST http://localhost:8080/api_jsonrpc.php \
  -H "Content-Type: application/json-rpc" \
  -d '{
    "jsonrpc": "2.0",
    "method": "user.login",
    "params": {
      "username": "Admin",
      "password": "zabbix"
    },
    "id": 1
  }'

# Получить список хостов
curl -X POST http://localhost:8080/api_jsonrpc.php \
  -H "Content-Type: application/json-rpc" \
  -d '{
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
      "output": ["hostid", "host"]
    },
    "auth": "<YOUR_AUTH_TOKEN>",
    "id": 1
  }'
```

---

## Модуль 2: Items и сбор данных (25 минут)

### 🎯 Напоминалка

**Типы Items:**
```
Zabbix agent       # Сбор через агента
Simple check       # Ping, порты без агента
SNMP agent         # SNMP мониторинг
Zabbix internal    # Внутренние метрики Zabbix
Zabbix trapper     # Прием данных от внешних источников
External check     # Выполнение скриптов на сервере
Database monitor   # SQL запросы
HTTP agent         # REST API, веб-мониторинг
Calculated         # Вычисляемые на основе других items
Dependent          # Зависимые от других items
```

**Основные ключи агента:**
```bash
# Система
system.cpu.util[,idle]           # CPU утилизация
system.cpu.load[percpu,avg1]     # Load average
vm.memory.size[available]        # Доступная память
system.uptime                    # Время работы

# Диски
vfs.fs.size[/,free]              # Свободное место
vfs.fs.size[/,pfree]             # % свободного места
vfs.fs.inode[/,pfree]            # % свободных inode
vfs.file.exists[/path/file]      # Проверка существования

# Сеть
net.if.in[eth0]                  # Входящий трафик
net.if.out[eth0]                 # Исходящий трафик
net.tcp.listen[80]               # Проверка порта
net.tcp.service[http]            # Проверка сервиса

# Процессы
proc.num[nginx]                  # Количество процессов
proc.cpu.util[nginx]             # CPU процесса
proc.mem[nginx]                  # Память процесса

# Логи
log[/var/log/app.log,ERROR]      # Мониторинг логов
logrt["/var/log/app*.log",ERROR] # С regex pattern

# Пользовательские
system.run[command]              # Выполнение команды
```

**Update interval (интервал обновления):**
```
1s-23h59m59s     # Фиксированный интервал
1m;wd1-5,h9-18   # По расписанию (пн-пт, 9-18)
1h/1-7,00:00-24:00  # Гибкое расписание
```

**Value type (тип значения):**
```
Numeric (float)     # Числовое с плавающей точкой
Numeric (unsigned)  # Целое положительное
Character           # Строка (до 255 символов)
Log                 # Лог файлы с timestamp
Text                # Текст (до 64KB)
```

**Preprocessing (предобработка):**
```
Regular expression      # Извлечение через regex
JSONPath               # Парсинг JSON
XML XPath              # Парсинг XML
Custom multiplier      # Умножение на коэффициент
Change per second      # Delta в секунду
Discard unchanged      # Игнорировать дубли
Prometheus pattern     # Парсинг Prometheus метрик
JavaScript             # Кастомная обработка
```

**HTTP Agent item:**
```yaml
URL: https://api.example.com/metrics
Query fields:
  - name: token
    value: {$API_TOKEN}
Headers:
  - Authorization: Bearer {$API_TOKEN}
Request type: GET
Timeout: 3s
Follow redirects: Yes
Verify peer: Yes
Verify host: Yes

Preprocessing:
  - JSONPath: $.data.cpu_usage
  - Custom multiplier: 1
```

**Dependent items (зависимые):**
```
Master item: system.run[ps aux]
↓
Dependent item 1: Preprocessing → JSONPath → $.processes[?(@.name=="nginx")].count
Dependent item 2: Preprocessing → JSONPath → $.processes[?(@.name=="mysql")].cpu
Dependent item 3: Preprocessing → JSONPath → $.processes[?(@.name=="redis")].memory
```

**Calculated items:**
```
# Процент использованной памяти
100*(last(//vm.memory.size[total])-last(//vm.memory.size[available]))/last(//vm.memory.size[total])

# Суммарный трафик
last(//net.if.in[eth0])+last(//net.if.out[eth0])

# Средний CPU за 5 минут
avg(//system.cpu.util,5m)
```

### 💻 Задание

Настрой расширенный сбор метрик:

1. **Создай кастомный UserParameter**
   
   На хосте с агентом:
   ```bash
   # Создай скрипт
   cat > /usr/local/bin/check_service.sh <<'EOF'
   #!/bin/bash
   SERVICE=$1
   if systemctl is-active --quiet $SERVICE; then
     echo 1
   else
     echo 0
   fi
   EOF
   
   chmod +x /usr/local/bin/check_service.sh
   
   # Добавь UserParameter
   cat > /etc/zabbix/zabbix_agent2.d/custom.conf <<EOF
   UserParameter=custom.service.status[*],/usr/local/bin/check_service.sh $1
   UserParameter=custom.disk.usage,df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
   EOF
   
   # Перезапусти агент
   systemctl restart zabbix-agent2
   
   # Тест
   zabbix_agent2 -t custom.service.status[nginx]
   ```

2. **Добавь item в Zabbix**
   - Configuration → Hosts → TestServer → Items → Create item
   - Name: Nginx service status
   - Type: Zabbix agent
   - Key: `custom.service.status[nginx]`
   - Type of information: Numeric (unsigned)
   - Update interval: 30s

3. **Настрой HTTP Agent для API мониторинга**
   
   Используем публичное API для теста:
   - Create item
   - Name: GitHub API Status
   - Type: HTTP agent
   - URL: `https://api.github.com/status`
   - Headers: `User-Agent: Zabbix`
   - Preprocessing:
     - JSONPath: `$.status`
     - Discard unchanged

4. **Создай Dependent items**
   
   Master item (один запрос к API):
   - Name: Docker Stats JSON
   - Type: Zabbix agent
   - Key: `system.run[docker stats --no-stream --format json]`
   
   Dependent items:
   - CPU Usage: Preprocessing → JSONPath → `$[0].CPUPerc`
   - Memory Usage: Preprocessing → JSONPath → `$[0].MemPerc`

5. **Настрой мониторинг логов**
   - Create item
   - Name: Application errors
   - Type: Zabbix agent (active)
   - Key: `log[/var/log/app/error.log,ERROR,,,skip]`
   - Type of information: Log
   - Update interval: 10s

### 🚀 Бонус (новое)

**Используй LLD (Low-Level Discovery) для автоматического обнаружения:**

1. **Создай Discovery Rule для дисков**
   ```bash
   # UserParameter для discovery
   cat > /etc/zabbix/zabbix_agent2.d/disk_discovery.conf <<'EOF'
   UserParameter=custom.disk.discovery,/usr/local/bin/disk_discovery.sh
   EOF
   
   # Скрипт discovery
   cat > /usr/local/bin/disk_discovery.sh <<'EOF'
   #!/bin/bash
   echo -n '{"data":['
   first=1
   df -h | awk 'NR>1 {print $1, $6}' | while read device mountpoint; do
     if [ $first -eq 1 ]; then
       first=0
     else
       echo -n ','
     fi
     echo -n '{"{#FSNAME}":"'$mountpoint'","{#FSDEVICE}":"'$device'"}'
   done
   echo ']}'
   EOF
   
   chmod +x /usr/local/bin/disk_discovery.sh
   systemctl restart zabbix-agent2
   ```

2. **В Zabbix создай Discovery Rule**
   - Configuration → Hosts → TestServer → Discovery rules
   - Name: Filesystem discovery
   - Type: Zabbix agent
   - Key: `custom.disk.discovery`
   - Update interval: 1h

3. **Создай Item prototype**
   - Name: `Free disk space on {#FSNAME}`
   - Key: `vfs.fs.size[{#FSNAME},free]`
   - Type: Numeric (unsigned)
   - Units: B

**Настрой Web Scenarios для мониторинга веб-приложений:**

```
Configuration → Hosts → TestServer → Web → Create web scenario

Name: Website availability
Application: Web monitoring
Agent: Mozilla/5.0
Update interval: 1m

Steps:
  Step 1: Homepage
    URL: https://example.com
    Required status codes: 200
    Timeout: 15s
    
  Step 2: Login page
    URL: https://example.com/login
    Required string: "Username"
    
  Step 3: API endpoint
    URL: https://api.example.com/health
    JSONPath: $.status = "healthy"
```

---

## Модуль 3: Triggers и проблемы (30 минут)

### 🎯 Напоминалка

**Severity (серьезность проблем):**
```
Not classified   # Не классифицировано
Information      # Информация
Warning          # Предупреждение
Average          # Средняя
High             # Высокая
Disaster         # Критическая
```

**Основные функции триггеров:**
```
last()           # Последнее значение
avg()            # Среднее за период
min()/max()      # Минимум/максимум
sum()            # Сумма
count()          # Количество
change()         # Изменение значения
diff()           # Разница между значениями
abschange()      # Абсолютное изменение
nodata()         # Нет данных
date()/time()    # Дата и время
now()            # Текущее время (Unix timestamp)
```

**Примеры триггеров:**
```bash
# CPU выше 80% последние 5 минут
avg(/Host/system.cpu.util,5m)>80

# Свободно менее 10% дискового пространства
last(/Host/vfs.fs.size[/,pfree])<10

# Нет данных более 3 минут
nodata(/Host/agent.ping,3m)=1

# Процесс не запущен
last(/Host/proc.num[nginx])<1

# Изменился размер файла
change(/Host/vfs.file.size[/etc/passwd])<>0

# HTTP код не 200
last(/Host/web.page.regexp[,,,200])=0

# Load average выше количества CPU
last(/Host/system.cpu.load[percpu,avg1])>last(/Host/system.cpu.num)

# Используется более 90% памяти
last(/Host/vm.memory.size[available])<last(/Host/vm.memory.size[total])*0.1

# Комбинированное условие (CPU и Memory)
avg(/Host/system.cpu.util,5m)>80 and last(/Host/vm.memory.size[pavailable])<20
```

**Expression с hysteresis (гистерезис):**
```bash
# Проблема: CPU > 80%
avg(/Host/system.cpu.util,5m)>80

# Восстановление: CPU < 70% (предотвращение флаппинга)
avg(/Host/system.cpu.util,5m)<70
```

**Trigger dependencies (зависимости):**
```
Trigger A: Host unavailable
↓ depends on
Trigger B: High CPU
Trigger C: High Memory

# Если A сработал, B и C не будут показывать проблемы
```

**Problem suppression (подавление проблем):**
```
Можно настроить подавление:
- По времени (например, во время обслуживания)
- По другим триггерам
- Вручную через Maintenance

Configuration → Maintenance → Create maintenance period
Name: Monthly maintenance
Active since: 2025-01-01 02:00
Period type: One time only
Duration: 4 hours
Hosts/Groups: Production servers
```

**Event tags и tag-based permissions:**
```yaml
Trigger expression: avg(/Host/system.cpu.util,5m)>80
Tags:
  - Application: Database
  - Environment: Production
  - Team: DBA

# Используется для:
# - Фильтрации в Dashboard
# - Routing уведомлений
# - RBAC доступа
```

### 💻 Задание

Создай систему триггеров для комплексного мониторинга:

1. **Базовый триггер с гистерезисом**
   
   Configuration → Hosts → TestServer → Triggers → Create trigger
   
   ```
   Name: High CPU usage on {HOST.NAME}
   Severity: Warning
   
   Expression:
   Problem: avg(/TestServer/system.cpu.util,5m)>80
   Recovery: avg(/TestServer/system.cpu.util,5m)<70
   
   Recovery expression: Expression
   PROBLEM event generation mode: Multiple
   OK event closes: All problems
   
   Tags:
   - component: cpu
   - severity: warning
   ```

2. **Триггер с множественными условиями**
   ```
   Name: Server overload
   Severity: High
   
   Expression:
   avg(/TestServer/system.cpu.util,5m)>90 and 
   last(/TestServer/vm.memory.size[pavailable])<10 and
   avg(/TestServer/system.cpu.load[percpu,avg5],5m)>2
   
   Description:
   CPU: {ITEM.LASTVALUE1}
   Memory available: {ITEM.LASTVALUE2}%
   Load average: {ITEM.LASTVALUE3}
   ```

3. **Триггер для мониторинга процесса**
   ```
   Name: Nginx is not running
   Severity: High
   
   Expression:
   last(/TestServer/proc.num[nginx])<1
   
   Manual close: Allowed
   
   Tags:
   - service: nginx
   - application: webserver
   ```

4. **Триггер с nodata**
   ```
   Name: No data from agent
   Severity: Average
   
   Expression:
   nodata(/TestServer/agent.ping,5m)=1
   
   Description:
   Agent not responding for 5 minutes
   Last seen: {ITEM.LASTCLOCK}
   ```

5. **Настрой Trigger dependencies**
   ```
   Parent trigger: Host {HOST.NAME} is unavailable
   Expression: max(/TestServer/agent.ping,#3)=0
   
   Child triggers (зависят от parent):
   - High CPU usage
   - High memory usage
   - Service down
   
   Configuration → Triggers → Dependencies → Add
   ```

6. **Создай Maintenance период**
   ```
   Configuration → Maintenance → Create
   
   Name: Weekend maintenance
   Type: With data collection
   Active since: This Saturday 02:00
   Maintenance period: Every week, Saturday, 02:00-06:00
   
   Hosts: TestServer
   
   Description: Regular weekend maintenance window
   ```

### 🚀 Бонус (новое)

**Используй новые возможности Trigger expressions (Zabbix 6.0+):**

1. **Функция `forecast()` для предсказания**
   ```
   Name: Disk will be full in 24h
   Severity: Warning
   
   Expression:
   timeleft(/TestServer/vfs.fs.size[/,free],1h,"linear")<24h
   
   # Предсказывает заполнение диска на основе тренда
   ```

2. **Использование макросов в триггерах**
   ```
   # Создай User macro
   Administration → General → Macros
   Macro: {$CPU.UTIL.MAX}
   Value: 80
   
   # В триггере
   avg(/TestServer/system.cpu.util,5m)>{$CPU.UTIL.MAX}
   
   # Можно переопределить на уровне хоста
   Configuration → Hosts → TestServer → Macros
   {$CPU.UTIL.MAX} = 90
   ```

3. **Event correlation (корреляция событий)**
   ```
   Configuration → Event correlation → Create
   
   Name: Flapping service detection
   
   Conditions:
   - New event tag Application equals: nginx
   - Event tag value equals: service_down
   - Type of event: Problem
   
   Operations:
   - Close old events
   - Tag: flapping
   ```

4. **Service monitoring с SLA**
   ```
   Services → Create service
   
   Name: Web Application
   Parent: None
   
   Problem tags:
   - application: webapp
   
   SLA: 99.9%
   
   Child services:
   - Frontend
   - API
   - Database
   
   # Автоматический расчет SLA на основе триггеров
   ```

---

## Модуль 4: Actions и уведомления (30 минут)

### 🎯 Напоминалка

**Типы Actions:**
```
Trigger actions        # При срабатывании триггеров
Discovery actions      # При обнаружении хостов
Auto-registration      # При авторегистрации агентов
Internal actions       # Внутренние события Zabbix
```

**Media types (каналы уведомлений):**
```
Email                  # SMTP почта
SMS                    # SMS шлюзы
Jabber                 # XMPP
Ez Texting             # SMS сервис
Script                 # Кастомные скрипты
Webhook                # HTTP вебхуки
```

**Популярные webhooks:**
```
Slack
MS Teams
Telegram
PagerDuty
OpsGenie
Mattermost
Discord
Jira
ServiceNow
```

**Структура Action:**
```yaml
Conditions:          # Когда выполнять
  - Trigger severity >= Warning
  - Host group = Production
  - Trigger value = PROBLEM

Operations:          # Что делать
  - Send message to: Admin group
  - Media type: Slack
  - Send to: #alerts
  - Message: 
      Subject: Problem: {EVENT.NAME}
      Body: |
        Problem started: {EVENT.TIME} {EVENT.DATE}
        Host: {HOST.NAME}
        Severity: {EVENT.SEVERITY}
        {TRIGGER.URL}

Recovery operations: # При восстановлении
  - Send message: Problem resolved
  
Update operations:   # При обновлении проблемы
  - Send message: Problem updated
```

**Escalation (эскалация):**
```yaml
Step 1 (0-60s):
  - Send to: Operator group
  - Via: Slack

Step 2 (60-600s):
  - Send to: Team Lead
  - Via: Email + SMS

Step 3 (600-3600s):
  - Send to: Manager
  - Via: Phone call
  - Execute: escalation_script.sh
```

**Макросы в сообщениях:**
```
{HOST.NAME}           # Имя хоста
{HOST.IP}             # IP хоста
{EVENT.NAME}          # Название события
{EVENT.SEVERITY}      # Серьезность
{EVENT.DATE}          # Дата
{EVENT.TIME}          # Время
{TRIGGER.STATUS}      # Статус триггера
{TRIGGER.SEVERITY}    # Серьезность триггера
{ITEM.VALUE}          # Значение item
{ITEM.LASTVALUE}      # Последнее значение
{EVENT.AGE}           # Возраст проблемы
{EVENT.ACK.STATUS}    # Статус подтверждения
{TRIGGER.URL}         # URL на триггер
{EVENT.TAGS}          # Теги события
{EVENT.OPDATA}        # Operational data
```

**Шаблоны сообщений:**
```
Subject: {EVENT.SEVERITY}: {EVENT.NAME}

Body:
Problem: {EVENT.NAME}
Host: {HOST.NAME} ({HOST.IP})
Severity: {EVENT.SEVERITY}
Started: {EVENT.DATE} {EVENT.TIME}
Duration: {EVENT.AGE}

Current values:
{ITEM.VALUE1}

Link: {TRIGGER.URL}

---
Zabbix Monitoring System
```

### 💻 Задание

Настрой систему уведомлений:

1. **Настрой Email уведомления**
   
   Administration → Media types → Email → Configure
   ```
   SMTP server: smtp.gmail.com
   SMTP server port: 587
   SMTP helo: zabbix.local
   SMTP email: your-email@gmail.com
   Connection security: STARTTLS
   Authentication: Username and password
   Username: your-email@gmail.com
   Password: <app-password>
   ```

2. **Создай Media для пользователя**
   
   Administration → Users → Admin → Media → Add
   ```
   Type: Email
   Send to: your-email@example.com
   When active: 1-7,00:00-24:00
   Use if severity: All
   Status: Enabled
   ```

3. **Настрой Slack webhook**
   
   Administration → Media types → Create media type
   ```
   Name: Slack
   Type: Webhook
   
   Parameters:
   - alert_message: {ALERT.MESSAGE}
   - alert_subject: {ALERT.SUBJECT}
   - event_source: {EVENT.SOURCE}
   - event_value: {EVENT.VALUE}
   - trigger_id: {TRIGGER.ID}
   - zabbix_url: http://your-zabbix-server
   
   Script:
   try {
     var params = JSON.parse(value);
     var req = new HttpRequest();
     req.addHeader('Content-Type: application/json');
     var webhook_url = '<YOUR_SLACK_WEBHOOK>';
     var payload = {
       channel: '#alerts',
       username: 'Zabbix',
       text: params.alert_subject,
       attachments: [{
         color: params.event_value == '1' ? 'danger' : 'good',
         text: params.alert_message
       }]
     };
     req.post(webhook_url, JSON.stringify(payload));
     return 'OK';
   } catch (error) {
     throw 'Slack notification failed: ' + error;
   }
   ```

4. **Создай Action для проблем**
   
   Configuration → Actions → Trigger actions → Create action
   ```
   Name: Notify on problems
   
   Conditions:
   - Trigger severity >= Warning
   - Host group equals: Production
   
   Operations:
   Step 1-2 (0-60s):
     Send message to: Zabbix administrators
     Via: Slack
     Custom message: Yes
     Subject: 🔴 Problem: {EVENT.NAME}
     Message:
       Host: {HOST.NAME}
       Severity: {EVENT.SEVERITY}
       Time: {EVENT.DATE} {EVENT.TIME}
       Details: {ITEM.VALUE1}
       Link: {TRIGGER.URL}
   
   Step 3-4 (60-300s):
     Send message to: Team Lead
     Via: Email
   
   Recovery operations:
     Send message to: Zabbix administrators
     Via: Slack
     Subject: ✅ Resolved: {EVENT.NAME}
     Message:
       Host: {HOST.NAME}
       Duration: {EVENT.AGE}
       Resolved: {EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}
   
   Update operations:
     Send message to: Zabbix administrators
     Via: Slack
     Custom message: Yes
     Subject: 🔄 Updated: {EVENT.NAME}
   ```

5. **Тест уведомлений**
   ```bash
   # Создай проблему искусственно
   # На агенте
   stress --cpu 8 --timeout 300s
   
   # Или через триггер
   Configuration → Hosts → TestServer → Triggers
   # Временно снизь порог CPU до 10%
   
   # Проверь
   Monitoring → Problems
   # Должно прийти уведомление
   
   # Восстанови порог
   ```

### 🚀 Бонус (новое)

**Настрой Telegram уведомления через Webhook:**

1. **Создай Telegram бота**
   ```
   - Найди @BotFather в Telegram
   - /newbot
   - Получи токен: 123456789:ABCdefGHIjklMNOpqrsTUVwxyz
   - Найди свой Chat ID: @userinfobot
   ```

2. **Создай Media Type для Telegram**
   ```javascript
   // Administration → Media types → Create
   Name: Telegram
   Type: Webhook
   
   Parameters:
   - bot_token: <YOUR_BOT_TOKEN>
   - chat_id: {ALERT.SENDTO}
   - message: {ALERT.MESSAGE}
   - parse_mode: HTML
   
   Script:
   try {
     var params = JSON.parse(value);
     var req = new HttpRequest();
     var url = 'https://api.telegram.org/bot' + params.bot_token + '/sendMessage';
     
     var message = params.message;
     // Экранирование HTML
     message = message.replace(/&/g, '&amp;')
                     .replace(/</g, '&lt;')
                     .replace(/>/g, '&gt;');
     
     var payload = {
       chat_id: params.chat_id,
       text: message,
       parse_mode: params.parse_mode,
       disable_web_page_preview: true
     };
     
     req.addHeader('Content-Type: application/json');
     var response = req.post(url, JSON.stringify(payload));
     
     if (req.getStatus() != 200) {
       throw 'Response code: ' + req.getStatus();
     }
     
     return 'OK';
   } catch (error) {
     throw 'Telegram notification failed: ' + error;
   }
   ```

**Настрой условную эскалацию:**

```yaml
Action: Smart escalation

Conditions:
- Trigger severity >= Average
- Problem is not suppressed

Operations:
Step 1 (0-300s):
  IF {EVENT.SEVERITY} >= High:
    - Send to: On-call engineer
    - Via: Telegram + SMS
  ELSE:
    - Send to: Team channel
    - Via: Slack

Step 2 (300-900s):
  IF {EVENT.ACK.STATUS} = No:
    - Send to: Team Lead
    - Via: Email + Phone
  
Step 3 (900-3600s):
  IF {EVENT.STATUS} = PROBLEM:
    - Execute: escalation_script.sh
    - Create: JIRA ticket
    - Send to: Manager
```

**Используй Problem tags для роутинга:**

```yaml
Action: Tag-based routing

Conditions:
- Trigger severity >= Warning

Operations:
IF {EVENT.TAGS} contains "team=database":
  - Send to: DBA team
  - Via: PagerDuty

IF {EVENT.TAGS} contains "team=network":
  - Send to: Network team
  - Via: OpsGenie

IF {EVENT.TAGS} contains "application=critical":
  - Send to: All teams
  - Via: Multiple channels
```

---

## Модуль 5: Templates и массовое управление (25 минут)

### 🎯 Напоминалка

**Структура Template:**
```
Template
├── Items               # Метрики для сбора
├── Triggers            # Условия проблем
├── Graphs              # Графики
├── Discovery rules     # Правила обнаружения
├── Web scenarios       # Веб-мониторинг
├── Macros              # Переменные шаблона
└── Linked templates    # Связанные шаблоны
```

**Встроенные templates:**
```
Linux by Zabbix agent
Windows by Zabbix agent
MySQL by Zabbix agent
PostgreSQL by Zabbix agent
Apache by HTTP
Nginx by HTTP
Docker by Zabbix agent 2
Kubernetes cluster by HTTP
VMware by HTTP
```


**Template linking (наследование):**
```
Template App Generic
↓ linked to
Template App Database
↓ linked to
Template App MySQL
↓ applied to
Host: mysql-prod-01

# Хост наследует все items/triggers/graphs
```

**Template macros (переменные):**
```
{$CPU.UTIL.MAX}          # Максимальная загрузка CPU
{$MEMORY.UTIL.MAX}       # Максимальная загрузка памяти
{$DISK.PFREE.MIN}        # Минимум свободного места
{$SERVICE.NAME}          # Имя сервиса
{$API.TOKEN}             # API токен
{$SNMP.COMMUNITY}        # SNMP community

# Можно переопределить на уровне:
# - Template → Host group → Host
```

**Export/Import templates:**
```bash
# Export через веб-интерфейс
Configuration → Templates → Select → Export
Format: YAML (предпочтительно) или XML

# Import
Configuration → Templates → Import
Source: файл YAML/XML
Rules:
  - Create new
  - Update existing
  - Delete missing
```

**Template groups:**
```
Templates/Applications      # Приложения
Templates/Databases        # БД
Templates/Network devices  # Сетевые устройства
Templates/Operating systems # ОС
Templates/Modules          # Модули
Templates/Virtualization   # Виртуализация
```

**Value mapping:**
```yaml
# Преобразование числовых значений в текст
Configuration → General → Value mapping

Name: Service state
Mappings:
  0 → Down
  1 → Up
  2 → Unknown

# Использование в items
Item: Service status
Type of information: Numeric (unsigned)
Show value: Service state
```

**Cloning и mass update:**
```bash
# Clone host
Configuration → Hosts → Select host → Clone
# Копирует всю конфигурацию

# Mass update
Configuration → Hosts → Select multiple → Mass update
Update:
  - Templates
  - Macros
  - Inventory
  - Tags
  - Groups
```

### 💻 Задание

Создай production-ready template:

1. **Создай кастомный Template**
   
   Configuration → Templates → Create template
   ```
   Template name: Template App Custom Service
   Groups: Templates/Applications
   
   Description:
   Template for custom application monitoring
   - Service availability
   - Response time
   - Error rate
   - Custom metrics
   ```

2. **Добавь Macros**
   ```
   {$SERVICE.NAME} = myapp
   {$SERVICE.PORT} = 8080
   {$SERVICE.URL} = http://localhost:8080/health
   {$RESPONSE.TIME.MAX} = 1000
   {$ERROR.RATE.MAX} = 5
   ```

3. **Создай Items**
   
   Item 1: Service status
   ```
   Name: Service {$SERVICE.NAME} status
   Type: Simple check
   Key: net.tcp.service[tcp,localhost,{$SERVICE.PORT}]
   Type of information: Numeric (unsigned)
   Value mapping: Service state (0=Down, 1=Up)
   Update interval: 30s
   Applications: Service monitoring
   ```
   
   Item 2: Service response time
   ```
   Name: Service {$SERVICE.NAME} response time
   Type: HTTP agent
   Key: service.response.time
   URL: {$SERVICE.URL}
   Type of information: Numeric (float)
   Units: ms
   Update interval: 1m
   
   Preprocessing:
   - Regular expression: \n → (empty) # Remove newlines
   - JSONPath: $.response_time
   ```
   
   Item 3: Service error rate
   ```
   Name: Service {$SERVICE.NAME} error rate
   Type: Zabbix agent
   Key: system.run[grep -c ERROR /var/log/{$SERVICE.NAME}/error.log]
   Type of information: Numeric (unsigned)
   Update interval: 5m
   ```

4. **Создай Triggers**
   
   Trigger 1: Service down
   ```
   Name: Service {$SERVICE.NAME} is down
   Severity: High
   Expression: last(/Template App Custom Service/net.tcp.service[tcp,localhost,{$SERVICE.PORT}])=0
   Recovery: last(/Template App Custom Service/net.tcp.service[tcp,localhost,{$SERVICE.PORT}])=1
   Manual close: Allowed
   
   Tags:
   - service: {$SERVICE.NAME}
   - component: availability
   ```
   
   Trigger 2: High response time
   ```
   Name: Service {$SERVICE.NAME} high response time
   Severity: Warning
   Expression: avg(/Template App Custom Service/service.response.time,5m)>{$RESPONSE.TIME.MAX}
   
   Description:
   Current: {ITEM.LASTVALUE}
   Threshold: {$RESPONSE.TIME.MAX}
   ```
   
   Trigger 3: High error rate
   ```
   Name: Service {$SERVICE.NAME} high error rate
   Severity: Average
   Expression: last(/Template App Custom Service/system.run[grep -c ERROR /var/log/{$SERVICE.NAME}/error.log])>{$ERROR.RATE.MAX}
   ```

5. **Создай Graph**
   ```
   Configuration → Templates → Template App Custom Service → Graphs → Create graph
   
   Name: Service {$SERVICE.NAME} performance
   Width: 900
   Height: 200
   Show legend: Yes
   
   Items:
   - service.response.time (Line, Green)
   - Y axis: Response time (ms)
   ```

6. **Добавь Discovery Rule**
   ```
   Name: Service endpoints discovery
   Type: HTTP agent
   Key: service.endpoints.discovery
   URL: {$SERVICE.URL}/endpoints
   Update interval: 1h
   
   Preprocessing:
   - JSONPath: $.endpoints
   
   Item prototypes:
   Name: Endpoint {#ENDPOINT} response time
   Key: endpoint[{#ENDPOINT},response.time]
   ```

7. **Export template**
   ```
   Configuration → Templates → Template App Custom Service → Export
   Format: YAML
   Сохрани: template_app_custom_service.yaml
   ```

8. **Примени template к хосту**
   ```
   Configuration → Hosts → TestServer → Templates → Select
   Link new templates: Template App Custom Service
   
   Macros (override):
   {$SERVICE.NAME} = nginx
   {$SERVICE.PORT} = 80
   ```

### 🚀 Бонус (новое)

**Создай Template с LLD для Docker containers:**

```bash
# UserParameter для Docker discovery
cat > /etc/zabbix/zabbix_agent2.d/docker_discovery.conf <<'EOF'
UserParameter=docker.containers.discovery,docker ps --format '{"data":[{{range .}}{{if .}}{{printf "{" }}{{printf "\"{#CONTAINER.NAME}\":\"%s\"," .Names}}{{printf "\"{#CONTAINER.ID}\":\"%s\"," .ID}}{{printf "\"{#CONTAINER.IMAGE}\":\"%s\"" .Image}}{{printf "},"}}{{end}}{{end}}]}' | sed 's/,]/]/'
UserParameter=docker.container.stats[*],docker stats --no-stream --format "{{json .}}" $1 | jq -r '.$2'
EOF

systemctl restart zabbix-agent2
```

**Template: Docker Containers**
```yaml
Discovery rule:
  Name: Docker containers discovery
  Key: docker.containers.discovery
  Update interval: 5m

Item prototypes:
  1. Container CPU usage
     Key: docker.container.stats[{#CONTAINER.ID},CPUPerc]
     Preprocessing: 
       - Regular expression: %$ → (empty)
       - Custom multiplier: 1
  
  2. Container Memory usage
     Key: docker.container.stats[{#CONTAINER.ID},MemPerc]
  
  3. Container status
     Key: docker.container.stats[{#CONTAINER.ID},Status]

Trigger prototypes:
  1. Container {#CONTAINER.NAME} high CPU
     Expression: avg(//docker.container.stats[{#CONTAINER.ID},CPUPerc],5m)>80
  
  2. Container {#CONTAINER.NAME} is down
     Expression: find(//docker.container.stats[{#CONTAINER.ID},Status],,"like","Up")=0

Graph prototypes:
  Name: Container {#CONTAINER.NAME} resources
  Items:
    - CPU usage
    - Memory usage
```

**Массовое применение templates через API:**

```python
#!/usr/bin/env python3
import requests
import json

ZABBIX_URL = "http://localhost:8080/api_jsonrpc.php"
ZABBIX_USER = "Admin"
ZABBIX_PASSWORD = "zabbix"

def get_auth_token():
    payload = {
        "jsonrpc": "2.0",
        "method": "user.login",
        "params": {
            "username": ZABBIX_USER,
            "password": ZABBIX_PASSWORD
        },
        "id": 1
    }
    response = requests.post(ZABBIX_URL, json=payload)
    return response.json()['result']

def get_template_id(auth, template_name):
    payload = {
        "jsonrpc": "2.0",
        "method": "template.get",
        "params": {
            "output": ["templateid"],
            "filter": {"host": [template_name]}
        },
        "auth": auth,
        "id": 2
    }
    response = requests.post(ZABBIX_URL, json=payload)
    return response.json()['result'][0]['templateid']

def get_hosts_by_group(auth, group_name):
    payload = {
        "jsonrpc": "2.0",
        "method": "host.get",
        "params": {
            "output": ["hostid", "host"],
            "groupids": group_name,
            "selectGroups": ["name"]
        },
        "auth": auth,
        "id": 3
    }
    response = requests.post(ZABBIX_URL, json=payload)
    return response.json()['result']

def link_template_to_hosts(auth, template_id, host_ids):
    for host_id in host_ids:
        payload = {
            "jsonrpc": "2.0",
            "method": "host.update",
            "params": {
                "hostid": host_id,
                "templates": [{"templateid": template_id}]
            },
            "auth": auth,
            "id": 4
        }
        response = requests.post(ZABBIX_URL, json=payload)
        print(f"Linked template to host {host_id}: {response.json()}")

# Использование
auth = get_auth_token()
template_id = get_template_id(auth, "Template App Custom Service")
hosts = get_hosts_by_group(auth, "Production servers")
host_ids = [h['hostid'] for h in hosts]
link_template_to_hosts(auth, template_id, host_ids)
```

**Template inheritance с override:**

```yaml
Base Template: Template Module ICMP Ping
Items:
  - ICMP ping
  - ICMP loss
  - ICMP response time

↓ linked to

Template OS Linux
Macros override:
  {$ICMP.LOSS.MAX} = 20  # Override from base 50
  {$ICMP.TIME.MAX} = 150 # Override from base 300

↓ linked to

Host: linux-server-01
Macros override:
  {$ICMP.LOSS.MAX} = 10  # Override from template 20
```

---

## Модуль 6: Dashboards и визуализация (30 минут)

### 🎯 Напоминалка

**Типы виджетов:**
```
Graph (classic)        # Классический график
Graph prototype        # График из LLD
Simple graph           # Простой график
Plain text             # Текст
URL                    # Внешний URL в iframe
Action log             # Лог действий
Data overview          # Обзор данных
Clock                  # Часы
Map                    # Карта сети
Problems               # Список проблем
Problems by severity   # Проблемы по серьезности
Host availability      # Доступность хостов
System information     # Информация о системе
Item history           # История item
Top hosts              # Топ хостов
Trigger overview       # Обзор триггеров
Web monitoring         # Веб-мониторинг
Gauge                  # Шкала/спидометр
Pie chart              # Круговая диаграмма
Bar chart              # Столбчатая диаграмма
Geomap                 # Географическая карта
Honeycomb              # Сотовая диаграмма
Item value             # Значение item
SLA report             # SLA отчет
```

**Dashboard pages:**
```yaml
Dashboard: Infrastructure Overview
├── Page 1: Overview
│   ├── Widget: Problems by severity
│   ├── Widget: System status
│   └── Widget: Top CPU usage
├── Page 2: Servers
│   ├── Widget: Linux servers
│   └── Widget: Windows servers
├── Page 3: Network
│   ├── Widget: Network map
│   └── Widget: Bandwidth
└── Page 4: Applications
    ├── Widget: Databases
    └── Widget: Web servers
```

**Widget configuration:**
```yaml
Widget: Graph
Settings:
  Data set 1:
    Host: TestServer
    Item: CPU utilization
    Time shift: 0
    Aggregation function: avg
    Aggregate interval: 1m
  
  Displaying options:
    History data selection: Auto
    Show legend: Yes
    Working time: Yes
    Percentile line: 95
  
  Time period:
    From: now-1h
    To: now
  
  Axes:
    Left Y: Auto
    Right Y: Auto
  
  Problems:
    Show problems: Yes
    Problem hosts: TestServer
```

**Time selector:**
```
Last 1 hour        # now-1h/now
Last 24 hours      # now-1d/now
Last 7 days        # now-7d/now
Last 30 days       # now-30d/now
Last year          # now-1y/now
Custom period      # 2025-01-01 00:00 to 2025-01-31 23:59
```

**Sharing dashboards:**
```
Monitoring → Dashboards → Properties
Sharing options:
  - Private (default)
  - Public (anyone with link)
  - Public with authentication

Dashboard URL:
http://zabbix/zabbix.php?action=dashboard.view&dashboardid=X
```

**Dashboard slideshows:**
```
Monitoring → Dashboards → Create slideshow

Name: Infrastructure monitoring
Slides:
  1. Overview (30s)
  2. Server health (30s)
  3. Network status (30s)
  4. Application metrics (30s)

# Автоматическая ротация для мониторинга на больших экранах
```

**URL widget с кастомными дашбордами:**
```yaml
Widget: URL
URL: https://grafana.example.com/dashboard/xyz?kiosk
Refresh interval: 30s
Dynamic item: No

# Интеграция внешних систем:
# - Grafana
# - Kibana
# - Custom apps
```

### 💻 Задание

Создай production dashboard:

1. **Создай новый Dashboard**
   ```
   Monitoring → Dashboards → Create dashboard
   
   Name: Production Infrastructure
   Owner: Admin
   Default page display period: Last 1 hour
   Start slideshow: No
   ```

2. **Page 1: System Overview**
   
   Widget 1: Problems by severity
   ```
   Type: Problems by severity
   Name: Current problems
   Refresh interval: 1m
   
   Show:
     - Problems
     - Recent problems: 1h
   
   Host groups: Production servers
   Exclude host groups: (none)
   Hosts: (all)
   Problem: (all)
   Severity: Not classified to Disaster
   Tags: (none)
   Show tags: 3
   Show timeline: Yes
   ```
   
   Widget 2: Top hosts by CPU
   ```
   Type: Top hosts
   Name: Top 10 CPU usage
   Refresh interval: 1m
   
   Host groups: Production servers
   Hosts: (all)
   Items:
     - system.cpu.util
   
   Order: Top N
   Order column: Last value
   Count: 10
   ```
   
   Widget 3: System information
   ```
   Type: System information
   Name: Zabbix server info
   Refresh interval: 5m
   
   Show:
     - System information
     - High availability cluster
     - Server
   ```

3. **Page 2: Server Monitoring**
   
   Widget 1: Host availability
   ```
   Type: Host availability
   Name: Server availability
   
   Host groups: Production servers
   Interface type: Agent
   
   Layout:
     - Horizontal
     - Show hosts in maintenance
   ```
   
   Widget 2: CPU Usage Graph
   ```
   Type: Graph
   Name: CPU utilization
   Refresh interval: 1m
   
   Data sets:
     Set 1:
       Host: {HOST.HOST}
       Item: system.cpu.util
       Color: Green
       Width: 2
   
   Time period: now-1h to now
   Show legend: Yes
   Show problems: Yes
   ```
   
   Widget 3: Memory Usage Graph
   ```
   Type: Graph
   Name: Memory utilization
   
   Data sets:
     Set 1: Memory used
     Set 2: Memory available
     Set 3: Memory cached
   
   Overrides:
     - Stacked area
     - Different colors
   ```

4. **Page 3: Application Metrics**
   
   Widget 1: Item value (Gauge)
   ```
   Type: Gauge
   Name: Service response time
   Refresh interval: 30s
   
   Item: service.response.time
   
   Thresholds:
     - 0-500ms: Green
     - 500-1000ms: Yellow
     - 1000+ms: Red
   
   Units: ms
   Needle: Show
   Scale: Auto
   ```
   
   Widget 2: Pie chart
   ```
   Type: Pie chart
   Name: Problems by severity
   
   Data set:
     Host groups: Production
     Item: Problem count
     Aggregation: sum
   
   Legend: Right side
   Show percentages: Yes
   ```

5. **Page 4: Custom Maps**
   
   Widget: Map
   ```
   Type: Map
   Name: Infrastructure map
   
   Map: Create new map
   Elements:
     - Hosts (auto-positioned)
     - Triggers (show problems)
     - Links (network connections)
   
   Background image: Network topology
   Size: 1200x800
   ```

6. **Настрой Auto-refresh**
   ```
   Dashboard settings:
   Default page display period: Last 1 hour
   Auto-refresh: Yes
   ```

7. **Share Dashboard**
   ```
   Properties → Sharing
   Type: Public link
   Copy URL: http://localhost:8080/zabbix.php?action=dashboard.view&dashboardid=X
   
   # Можно открыть на TV/monitor без авторизации
   ```

### 🚀 Бонус (новое)

**Создай интерактивный Geomap:**

```yaml
Widget: Geomap
Name: Global infrastructure

Settings:
  Default view:
    Latitude: 48.8566 (Paris)
    Longitude: 2.3522
    Zoom: 5
  
  Host groups: Production servers
  
  Mappings:
    - Tag: location=europe → Icon: Server (Blue)
    - Tag: location=us → Icon: Server (Green)
    - Tag: location=asia → Icon: Server (Yellow)
  
  Show problems: Yes
  Minimum severity: Average

# На хостах добавь tags:
Host: paris-web-01
Tags:
  - location: europe
  - geo_lat: 48.8566
  - geo_lon: 2.3522
```

**Honeycomb widget для visualize многих метрик:**

```yaml
Widget: Honeycomb
Name: Service health matrix

Data:
  Host groups: Production servers
  Items:
    - system.cpu.util
    - vm.memory.size[pavailable]
    - vfs.fs.size[/,pfree]
  
  Color palette:
    - 0-70%: Green
    - 70-85%: Yellow
    - 85-100%: Red

# Показывает состояние всех серверов в виде сот
```

**JavaScript widget для кастомной визуализации:**

```javascript
// Monitoring → Dashboards → Add widget → Item value

// Custom JavaScript
<script>
// Get item value
var value = widget.getFieldValue('item');
var threshold = 80;

// Custom HTML
var html = '<div style="padding: 20px; text-align: center;">';
html += '<h2>CPU Usage</h2>';
html += '<div style="font-size: 48px; font-weight: bold; color: ' + 
        (value > threshold ? 'red' : 'green') + ';">';
html += value + '%';
html += '</div>';

if (value > threshold) {
  html += '<div style="color: red; margin-top: 10px;">⚠️ High CPU usage!</div>';
}

html += '</div>';

document.getElementById(widget.uniqueid).innerHTML = html;

// Auto-refresh every 30 seconds
setTimeout(function() {
  widget.refresh();
}, 30000);
</script>
```

**SLA Report widget:**

```yaml
Widget: SLA report
Name: Service availability SLA

Settings:
  Service: Web Application
  SLA: 99.9%
  Reporting period: Last 30 days
  
  Display:
    - SLA
    - Uptime
    - Downtime
    - Problem list
  
  Excluded downtimes:
    - Maintenance windows
    - Planned outages

# Автоматический расчет на основе триггеров
```

**Dashboard templates для разных ролей:**

```yaml
Dashboard: Executive View (CEO/CTO)
Widgets:
  - Overall SLA
  - Critical problems only
  - Service availability
  - Monthly trends

Dashboard: Operations View (DevOps)
Widgets:
  - All problems
  - Server metrics
  - Application performance
  - Recent changes
  - Alert history

Dashboard: Developer View
Widgets:
  - Application logs
  - API response times
  - Error rates
  - Database performance
  - Deployment status

Dashboard: NOC View (24/7 monitoring)
Widgets:
  - Network map
  - All active problems
  - Host availability
  - Auto-rotating slideshow
```

**Export/Import dashboards:**

```bash
# Export через API
curl -X POST http://localhost:8080/api_jsonrpc.php \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "dashboard.export",
    "params": {
      "dashboardids": ["1"]
    },
    "auth": "<TOKEN>",
    "id": 1
  }' | jq -r '.result' > dashboard.json

# Import
Monitoring → Dashboards → Import
Select file: dashboard.json
Rules: Create new / Update existing
```

---
