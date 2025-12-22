# CCNA Мини-курс на основе Jeremy's Mega Lab
## Часть 7: Security Features (ACL, Port Security, DHCP Snooping, DAI)

---

## 📋 Теоретический минимум

### Ключевые концепции безопасности:

**1. Access Control Lists (ACL)**
- Фильтрация трафика на основе условий
- Standard ACL: только source IP (1-99, 1300-1999)
- Extended ACL: source/dest IP, protocol, port (100-199, 2000-2699)
- Named ACL: текстовое имя вместо номера

**2. Port Security**
- Ограничение MAC адресов на порту
- Защита от MAC flooding attacks
- Violation modes: shutdown, restrict, protect
- Sticky MAC learning - автосохранение адресов

**3. DHCP Snooping**
- Защита от rogue DHCP servers
- Trusted vs Untrusted ports
- DHCP Snooping Binding Table
- Option 82 (relay agent information)

**4. DAI (Dynamic ARP Inspection)**
- Защита от ARP spoofing/poisoning
- Использует DHCP Snooping Binding Table
- Trusted vs Untrusted ports
- Validation checks: src-mac, dst-mac, ip

---

## 🎯 Цели раздела

- ✅ Настроить Extended ACL между офисами
- ✅ Применить ACL близко к источнику
- ✅ Настроить Port Security на access портах
- ✅ Настроить DHCP Snooping на access switches
- ✅ Настроить DAI на access switches
- ✅ Проверить работу всех security features

---

## 🗺️ Security Design

### ACL Policy:
- **Разрешить:** ICMP из Office A PCs → Office B PCs
- **Запретить:** Весь другой трафик из Office A PCs → Office B PCs
- **Разрешить:** Весь остальной трафик

### Port Security:
- **Minimum необходимых MAC:** 
  - PC only: 1 MAC
  - PC + Phone: 2 MACs
  - LWAP: 1 MAC
  - Server: 1 MAC
- **Violation Mode:** Restrict (блокирует + логирует)
- **Sticky Learning:** Включено

### DHCP Snooping:
- **Trusted:** Uplinks к Distribution (G0/1-2)
- **Untrusted:** Access ports (F0/1, F0/2)
- **Rate Limit:** 15 pps (F0/1), 100 pps (F0/2 для WLC)
- **Option 82:** Отключено

### DAI:
- **Trusted:** Uplinks к Distribution (G0/1-2)
- **Untrusted:** Access ports
- **Validation:** dst-mac, src-mac, ip

---

## 📝 ЧАСТЬ 1: Extended ACL Configuration

### ШАГ 1: Extended ACL между офисами

**Концепция:**
- Office A PCs (10.1.0.0/24) → Office B PCs (10.3.0.0/24)
- Разрешить: ICMP (ping)
- Запретить: Весь остальной трафик между этими сетями
- Разрешить: Весь другой трафик

**Best Practice:** Extended ACL применяется близко к источнику.

**Где применить:**
- DSW-A1 и DSW-A2
- VLAN 10 SVI (входящий трафик от Office A PCs)

---

### DSW-A1: ACL Configuration

```cisco
DSW-A1(config)# ip access-list extended OfficeA_to_OfficeB
DSW-A1(config-ext-nacl)# permit icmp 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
DSW-A1(config-ext-nacl)# deny ip 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
DSW-A1(config-ext-nacl)# permit ip any any
DSW-A1(config-ext-nacl)# exit

DSW-A1(config)# interface vlan 10
DSW-A1(config-if)# ip access-group OfficeA_to_OfficeB in
DSW-A1(config-if)# exit
DSW-A1(config)# do write
```

**Объяснение построчно:**
1. `permit icmp` - разрешить ping из 10.1.0.0/24 в 10.3.0.0/24
2. `deny ip` - запретить весь остальной IP трафик между этими сетями
3. `permit ip any any` - разрешить весь остальной трафик
4. `ip access-group ... in` - применить ACL на входящий трафик VLAN 10

---

### DSW-A2: ACL Configuration

```cisco
DSW-A2(config)# ip access-list extended OfficeA_to_OfficeB
DSW-A2(config-ext-nacl)# permit icmp 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
DSW-A2(config-ext-nacl)# deny ip 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
DSW-A2(config-ext-nacl)# permit ip any any
DSW-A2(config-ext-nacl)# exit

DSW-A2(config)# interface vlan 10
DSW-A2(config-if)# ip access-group OfficeA_to_OfficeB in
DSW-A2(config-if)# exit
DSW-A2(config)# do write
```

**Проверка ACL:**
```cisco
DSW-A1# show ip access-lists
DSW-A1# show ip interface vlan 10 | include access list
```

---

### Тест ACL

**С PC1 (Office A):**
```
PC> ping 10.3.0.11
```
✅ Ping должен работать (ICMP разрешён)

**Попробуйте SSH или Telnet:**
```
PC> ssh -l cisco 10.3.0.11
```
❌ Соединение должно быть заблокировано (не ICMP)

**Ping другие сети - должен работать:**
```
PC> ping 10.5.0.4    (SRV1)
PC> ping 10.0.0.76   (R1)
```
✅ Работает (permit ip any any)

---

## 📝 ЧАСТЬ 2: Port Security Configuration

### ШАГ 2: Port Security на Access Switches

**Параметры:**
- **Maximum MACs:**
  - LWAP (ASW-A1 F0/1, ASW-B1 F0/1): 1 MAC
  - PC + Phone (ASW-A2, A3, B2 F0/1): 2 MACs
  - Server (ASW-B3 F0/1): 1 MAC
- **Violation Mode:** Restrict
- **Sticky Learning:** Enabled

---

### ASW-A1: LWAP Port (1 MAC)

```cisco
ASW-A1(config)# interface f0/1
ASW-A1(config-if)# switchport port-security
ASW-A1(config-if)# switchport port-security violation restrict
ASW-A1(config-if)# switchport port-security mac-address sticky
ASW-A1(config-if)# exit
ASW-A1(config)# do write
```

**Объяснение:**
- `switchport port-security` - включить Port Security
- `violation restrict` - блокировать invalid трафик + логировать
- `mac-address sticky` - автосохранение MAC адресов
- `maximum` не указываем - default = 1

---

### ASW-B1: LWAP Port (1 MAC)

```cisco
ASW-B1(config)# interface f0/1
ASW-B1(config-if)# switchport port-security
ASW-B1(config-if)# switchport port-security violation restrict
ASW-B1(config-if)# switchport port-security mac-address sticky
ASW-B1(config-if)# exit
ASW-B1(config)# do write
```

---

### ASW-B3: Server Port (1 MAC)

```cisco
ASW-B3(config)# interface f0/1
ASW-B3(config-if)# switchport port-security
ASW-B3(config-if)# switchport port-security violation restrict
ASW-B3(config-if)# switchport port-security mac-address sticky
ASW-B3(config-if)# exit
ASW-B3(config)# do write
```

---

### ASW-A2: PC + Phone Port (2 MACs)

```cisco
ASW-A2(config)# interface f0/1
ASW-A2(config-if)# switchport port-security
ASW-A2(config-if)# switchport port-security maximum 2
ASW-A2(config-if)# switchport port-security violation restrict
ASW-A2(config-if)# switchport port-security mac-address sticky
ASW-A2(config-if)# exit
ASW-A2(config)# do write
```

**Ключевое отличие:** `maximum 2` для PC + Phone

---

### ASW-A3: PC + Phone Port (2 MACs)

```cisco
ASW-A3(config)# interface f0/1
ASW-A3(config-if)# switchport port-security
ASW-A3(config-if)# switchport port-security maximum 2
ASW-A3(config-if)# switchport port-security violation restrict
ASW-A3(config-if)# switchport port-security mac-address sticky
ASW-A3(config-if)# exit
ASW-A3(config)# do write
```

---

### ASW-B2: PC + Phone Port (2 MACs)

```cisco
ASW-B2(config)# interface f0/1
ASW-B2(config-if)# switchport port-security
ASW-B2(config-if)# switchport port-security maximum 2
ASW-B2(config-if)# switchport port-security violation restrict
ASW-B2(config-if)# switchport port-security mac-address sticky
ASW-B2(config-if)# exit
ASW-B2(config)# do write
```

**Проверка Port Security:**
```cisco
ASW-A2# show port-security
ASW-A2# show port-security interface f0/1
ASW-A2# show port-security address
```

---

## 📝 ЧАСТЬ 3: DHCP Snooping Configuration

### ШАГ 3: DHCP Snooping на Access Switches

**Концепция:**
- **Trusted ports:** G0/1-2 (uplinks к Distribution)
- **Untrusted ports:** F0/1-2 (access ports)
- **Rate Limiting:**
  - F0/1: 15 pps (стандартные устройства)
  - F0/2 на ASW-A1: 100 pps (WLC - много DHCP запросов)
- **Option 82:** Отключено (no ip dhcp snooping information option)

---

### ASW-A1: DHCP Snooping (LWAP + WLC)

```cisco
ASW-A1(config)# ip dhcp snooping
ASW-A1(config)# ip dhcp snooping vlan 10,20,40,99
ASW-A1(config)# no ip dhcp snooping information option

ASW-A1(config)# interface range g0/1-2
ASW-A1(config-if-range)# ip dhcp snooping trust
ASW-A1(config-if-range)# exit

ASW-A1(config)# interface f0/1
ASW-A1(config-if)# ip dhcp snooping limit rate 15
ASW-A1(config-if)# exit

ASW-A1(config)# interface f0/2
ASW-A1(config-if)# ip dhcp snooping limit rate 100
ASW-A1(config-if)# exit
ASW-A1(config)# do write
```

**Объяснение:**
- `ip dhcp snooping` - включить глобально
- `ip dhcp snooping vlan` - включить для VLANs
- `no ip dhcp snooping information option` - отключить Option 82
- `ip dhcp snooping trust` - доверенные порты (uplinks)
- `limit rate` - ограничение DHCP пакетов в секунду

---

### ASW-A2, ASW-A3: DHCP Snooping (PC + Phone)

```cisco
! ASW-A2
ASW-A2(config)# ip dhcp snooping
ASW-A2(config)# ip dhcp snooping vlan 10,20,40,99
ASW-A2(config)# no ip dhcp snooping information option

ASW-A2(config)# interface range g0/1-2
ASW-A2(config-if-range)# ip dhcp snooping trust
ASW-A2(config-if-range)# exit

ASW-A2(config)# interface f0/1
ASW-A2(config-if)# ip dhcp snooping limit rate 15
ASW-A2(config-if)# exit
ASW-A2(config)# do write
```

```cisco
! ASW-A3
ASW-A3(config)# ip dhcp snooping
ASW-A3(config)# ip dhcp snooping vlan 10,20,40,99
ASW-A3(config)# no ip dhcp snooping information option

ASW-A3(config)# interface range g0/1-2
ASW-A3(config-if-range)# ip dhcp snooping trust
ASW-A3(config-if-range)# exit

ASW-A3(config)# interface f0/1
ASW-A3(config-if)# ip dhcp snooping limit rate 15
ASW-A3(config-if)# exit
ASW-A3(config)# do write
```

---

### ASW-B1, ASW-B2, ASW-B3: DHCP Snooping (Office B)

```cisco
! ASW-B1
ASW-B1(config)# ip dhcp snooping
ASW-B1(config)# ip dhcp snooping vlan 10,20,30,99
ASW-B1(config)# no ip dhcp snooping information option

ASW-B1(config)# interface range g0/1-2
ASW-B1(config-if-range)# ip dhcp snooping trust
ASW-B1(config-if-range)# exit

ASW-B1(config)# interface f0/1
ASW-B1(config-if)# ip dhcp snooping limit rate 15
ASW-B1(config-if)# exit
ASW-B1(config)# do write
```

```cisco
! ASW-B2
ASW-B2(config)# ip dhcp snooping
ASW-B2(config)# ip dhcp snooping vlan 10,20,30,99
ASW-B2(config)# no ip dhcp snooping information option

ASW-B2(config)# interface range g0/1-2
ASW-B2(config-if-range)# ip dhcp snooping trust
ASW-B2(config-if-range)# exit

ASW-B2(config)# interface f0/1
ASW-B2(config-if)# ip dhcp snooping limit rate 15
ASW-B2(config-if)# exit
ASW-B2(config)# do write
```

```cisco
! ASW-B3
ASW-B3(config)# ip dhcp snooping
ASW-B3(config)# ip dhcp snooping vlan 10,20,30,99
ASW-B3(config)# no ip dhcp snooping information option

ASW-B3(config)# interface range g0/1-2
ASW-B3(config-if-range)# ip dhcp snooping trust
ASW-B3(config-if-range)# exit

ASW-B3(config)# interface f0/1
ASW-B3(config-if)# ip dhcp snooping limit rate 15
ASW-B3(config-if)# exit
ASW-B3(config)# do write
```

**Проверка DHCP Snooping:**
```cisco
ASW-A1# show ip dhcp snooping
ASW-A1# show ip dhcp snooping binding
```

---

## 📝 ЧАСТЬ 4: DAI Configuration

### ШАГ 4: Dynamic ARP Inspection

**Концепция:**
- DAI использует DHCP Snooping Binding Table
- **Trusted ports:** G0/1-2 (uplinks)
- **Untrusted ports:** F0/1-2 (access)
- **Validation checks:** dst-mac, src-mac, ip

---

### ASW-A1: DAI Configuration

```cisco
ASW-A1(config)# ip arp inspection vlan 10,20,40,99
ASW-A1(config)# ip arp inspection validate dst-mac src-mac ip

ASW-A1(config)# interface range g0/1-2
ASW-A1(config-if-range)# ip arp inspection trust
ASW-A1(config-if-range)# exit
ASW-A1(config)# do write
```

**Объяснение:**
- `ip arp inspection vlan` - включить DAI для VLANs
- `validate dst-mac src-mac ip` - проверки целостности ARP
- `ip arp inspection trust` - доверенные порты

---

### ASW-A2, ASW-A3: DAI Configuration

```cisco
! ASW-A2
ASW-A2(config)# ip arp inspection vlan 10,20,40,99
ASW-A2(config)# ip arp inspection validate dst-mac src-mac ip
ASW-A2(config)# interface range g0/1-2
ASW-A2(config-if-range)# ip arp inspection trust
ASW-A2(config-if-range)# exit
ASW-A2(config)# do write

! ASW-A3
ASW-A3(config)# ip arp inspection vlan 10,20,40,99
ASW-A3(config)# ip arp inspection validate dst-mac src-mac ip
ASW-A3(config)# interface range g0/1-2
ASW-A3(config-if-range)# ip arp inspection trust
ASW-A3(config-if-range)# exit
ASW-A3(config)# do write
```

---

### ASW-B1, ASW-B2, ASW-B3: DAI Configuration

```cisco
! ASW-B1
ASW-B1(config)# ip arp inspection vlan 10,20,30,99
ASW-B1(config)# ip arp inspection validate dst-mac src-mac ip
ASW-B1(config)# interface range g0/1-2
ASW-B1(config-if-range)# ip arp inspection trust
ASW-B1(config-if-range)# exit
ASW-B1(config)# do write

! ASW-B2
ASW-B2(config)# ip arp inspection vlan 10,20,30,99
ASW-B2(config)# ip arp inspection validate dst-mac src-mac ip
ASW-B2(config)# interface range g0/1-2
ASW-B2(config-if-range)# ip arp inspection trust
ASW-B2(config-if-range)# exit
ASW-B2(config)# do write

! ASW-B3
ASW-B3(config)# ip arp inspection vlan 10,20,30,99
ASW-B3(config)# ip arp inspection validate dst-mac src-mac ip
ASW-B3(config)# interface range g0/1-2
ASW-B3(config-if-range)# ip arp inspection trust
ASW-B3(config-if-range)# exit
ASW-B3(config)# do write
```

**Проверка DAI:**
```cisco
ASW-A1# show ip arp inspection
ASW-A1# show ip arp inspection interfaces
```

---

## ✅ Проверка конфигурации

### 1. ACL Verification

```cisco
DSW-A1# show ip access-lists
DSW-A1# show ip access-lists OfficeA_to_OfficeB
DSW-A1# show ip interface vlan 10 | include access list
```

**Тест с PC1:**
```
PC> ping 10.3.0.11       (✅ должен работать - ICMP)
PC> telnet 10.3.0.11     (❌ должен блокироваться)
PC> ping 10.5.0.4        (✅ работает - другая сеть)
```

---

### 2. Port Security Verification

```cisco
ASW-A2# show port-security
ASW-A2# show port-security interface f0/1
ASW-A2# show port-security address
```

**Ожидаемый вывод:**
```
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Restrict
Maximum MAC Addresses      : 2
Total MAC Addresses        : 1 (или 2)
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1 (или 2)
Security Violation Count   : 0
```

---

### 3. DHCP Snooping Verification

```cisco
ASW-A1# show ip dhcp snooping
ASW-A1# show ip dhcp snooping binding
```

**Ожидаемый вывод:**
```
DHCP snooping is enabled
...
Interface        Trusted    Rate limit (pps)
---------        -------    ----------------
GigabitEthernet0/1  yes    unlimited
GigabitEthernet0/2  yes    unlimited
FastEthernet0/1     no     15
FastEthernet0/2     no     100
```

**DHCP Binding Table должен содержать записи:**
```
MacAddress          IpAddress    Lease(sec)  Type           VLAN  Interface
------------------  -----------  ----------  -------------  ----  --------------------
xx:xx:xx:xx:xx:xx   10.1.0.11    86400       dhcp-snooping   10   FastEthernet0/1
```

---

### 4. DAI Verification

```cisco
ASW-A1# show ip arp inspection
ASW-A1# show ip arp inspection interfaces
ASW-A1# show ip arp inspection statistics
```

**Ожидаемый вывод:**
```
Source Mac Validation      : Enabled
Destination Mac Validation : Enabled
IP Address Validation      : Enabled

Interface        Trust State     Rate (pps)
---------------  -----------     ----------
Gi0/1            Trusted         None
Gi0/2            Trusted         None
Fa0/1            Untrusted       15
Fa0/2            Untrusted       100
```

---

## 📊 Security Features Summary

### ACL Configuration

| Device | ACL Name | Applied On | Direction | Purpose |
|--------|----------|------------|-----------|---------|
| DSW-A1 | OfficeA_to_OfficeB | VLAN 10 | IN | Filter A→B traffic |
| DSW-A2 | OfficeA_to_OfficeB | VLAN 10 | IN | Filter A→B traffic |

**ACL Rules:**
1. Permit ICMP (10.1.0.0/24 → 10.3.0.0/24)
2. Deny IP (10.1.0.0/24 → 10.3.0.0/24)
3. Permit IP (any → any)

---

### Port Security Summary

| Switch | Port | Max MACs | Violation | Sticky | Device |
|--------|------|----------|-----------|--------|--------|
| ASW-A1 | F0/1 | 1 | Restrict | Yes | LWAP1 |
| ASW-A2 | F0/1 | 2 | Restrict | Yes | PC+Phone |
| ASW-A3 | F0/1 | 2 | Restrict | Yes | PC+Phone |
| ASW-B1 | F0/1 | 1 | Restrict | Yes | LWAP2 |
| ASW-B2 | F0/1 | 2 | Restrict | Yes | PC+Phone |
| ASW-B3 | F0/1 | 1 | Restrict | Yes | SRV1 |

---

### DHCP Snooping Summary

| Switch | Trusted Ports | Untrusted Ports | Rate Limits | VLANs |
|--------|---------------|-----------------|-------------|-------|
| ASW-A1 | G0/1-2 | F0/1(15), F0/2(100) | 15/100 pps | 10,20,40,99 |
| ASW-A2 | G0/1-2 | F0/1(15) | 15 pps | 10,20,40,99 |
| ASW-A3 | G0/1-2 | F0/1(15) | 15 pps | 10,20,40,99 |
| ASW-B1 | G0/1-2 | F0/1(15) | 15 pps | 10,20,30,99 |
| ASW-B2 | G0/1-2 | F0/1(15) | 15 pps | 10,20,30,99 |
| ASW-B3 | G0/1-2 | F0/1(15) | 15 pps | 10,20,30,99 |

---

### DAI Summary

| Switch | VLANs | Trusted Ports | Validation Checks |
|--------|-------|---------------|-------------------|
| ASW-A1 | 10,20,40,99 | G0/1-2 | dst-mac, src-mac, ip |
| ASW-A2 | 10,20,40,99 | G0/1-2 | dst-mac, src-mac, ip |
| ASW-A3 | 10,20,40,99 | G0/1-2 | dst-mac, src-mac, ip |
| ASW-B1 | 10,20,30,99 | G0/1-2 | dst-mac, src-mac, ip |
| ASW-B2 | 10,20,30,99 | G0/1-2 | dst-mac, src-mac, ip |
| ASW-B3 | 10,20,30,99 | G0/1-2 | dst-mac, src-mac, ip |

---

## 💡 Практические советы

### ACL Best Practices:
1. ✅ Extended ACL близко к источнику
2. ✅ Standard ACL близко к destination
3. ✅ Named ACL для читаемости
4. ✅ Завершайте с `permit ip any any` если нужно
5. ✅ Документируйте ACL с комментариями (`remark`)

### Port Security Best Practices:
1. ✅ Используйте `restrict` mode (не `shutdown`)
2. ✅ Включайте sticky learning
3. ✅ Настраивайте minimum necessary MACs
4. ✅ Мониторьте violation счётчики

### DHCP Snooping Best Practices:
1. ✅ Всегда отключайте Option 82 (`no ip dhcp snooping information option`)
2. ✅ Trust uplinks, не trust access ports
3. ✅ Настраивайте rate limits на untrusted ports
4. ✅ Проверяйте binding table регулярно

### DAI Best Practices:
1. ✅ Требует DHCP Snooping
2. ✅ Включайте все validation checks
3. ✅ Trust те же порты что и для DHCP Snooping
4. ✅ Мониторьте statistics на атаки

---

## 🧪 Тестирование Security

### Тест 1: ACL Filtering

```
! С PC1 (10.1.0.x)
PC> ping 10.3.0.11       (✅ ICMP allowed)
PC> telnet 10.3.0.11     (❌ TCP denied)
PC> ping 10.5.0.4        (✅ other traffic allowed)
```

---

### Тест 2: Port Security Violation

1. Подключите дополнительное устройство к порту
2. Порт должен заблокировать трафик от нового MAC
3. Проверьте violation counter

```cisco
ASW-A2# show port-security interface f0/1
! Security Violation Count должен увеличиться
```

---

### Тест 3: DHCP Snooping

1. Попробуйте запустить rogue DHCP server на untrusted порту
2. DHCP Snooping должен заблокировать DHCP Offer

```cisco
ASW-A1# show ip dhcp snooping statistics
```

---

### Тест 4: DAI Protection

1. Попробуйте ARP spoofing с untrusted порта
2. DAI должен заблокировать invalid ARP packets

```cisco
ASW-A1# show ip arp inspection statistics
```

---

## 🎓 Ключевые команды Части 7

```cisco
# Extended ACL
ip access-list extended [name]
 permit|deny [protocol] [source] [wildcard] [dest] [wildcard]
 permit ip any any
interface [type] [number]
 ip access-group [name] in|out

# Port Security
interface [type] [number]
 switchport port-security
 switchport port-security maximum [number]
 switchport port-security violation {shutdown|restrict|protect}
 switchport port-security mac-address sticky

# DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan [vlan-list]
no ip dhcp snooping information option
interface [type] [number]
 ip dhcp snooping trust
 ip dhcp snooping limit rate [pps]

# DAI
ip arp inspection vlan [vlan-list]
ip arp inspection validate {dst-mac|src-mac|ip}
interface [type] [number]
 ip arp inspection trust

# Verification
show ip access-lists [name]
show ip interface [type] [number] | include access list
show port-security
show port-security interface [type] [number]
show port-security address
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
show ip arp inspection vlan [vlan-id]
```

---

## 🔍 Troubleshooting Security Features

### Проблема 1: ACL не работает

**Симптомы:** Трафик не фильтруется

**Причины и решения:**

1. **ACL не применён к интерфейсу**
```cisco
! Проверка
show ip interface vlan 10 | include access list

! Решение
interface vlan 10
 ip access-group OfficeA_to_OfficeB in
```

2. **Неправильный wildcard mask**
```cisco
! Проверка
show ip access-lists OfficeA_to_OfficeB

! Исправление
ip access-list extended OfficeA_to_OfficeB
 no 10
 10 permit icmp 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
```

3. **Неправильное направление (in vs out)**
```cisco
! Extended ACL обычно применяется IN (inbound)
interface vlan 10
 ip access-group OfficeA_to_OfficeB in
```

---

### Проблема 2: Port Security блокирует легитимный трафик

**Симптомы:** Порт в err-disabled или violation count растёт

**Причины и решения:**

1. **Maximum MACs слишком мал**
```cisco
! Проверка
show port-security interface f0/1

! Решение
interface f0/1
 switchport port-security maximum 2
```

2. **Порт в err-disabled (violation mode = shutdown)**
```cisco
! Проверка
show interfaces status | include err-disabled

! Решение
interface f0/1
 shutdown
 no shutdown
 
! Или изменить mode
 switchport port-security violation restrict
```

---

### Проблема 3: DHCP не работает

**Симптомы:** Клиенты не получают IP адреса

**Причины и решения:**

1. **Option 82 не отключен**
```cisco
! Проверка
show ip dhcp snooping

! Решение
no ip dhcp snooping information option
```

2. **Uplinks не trusted**
```cisco
! Проверка
show ip dhcp snooping

! Решение
interface range g0/1-2
 ip dhcp snooping trust
```

3. **Rate limit слишком низкий**
```cisco
! Проверка
show ip dhcp snooping

! Решение
interface f0/1
 ip dhcp snooping limit rate 50
```

---

### Проблема 4: DAI блокирует весь ARP трафик

**Симптомы:** Нет ARP entries, no connectivity

**Причины и решения:**

1. **DHCP Snooping не включен**
```cisco
! DAI требует DHCP Snooping binding table
! Решение
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,99
```

2. **Uplinks не trusted**
```cisco
! Проверка
show ip arp inspection interfaces

! Решение
interface range g0/1-2
 ip arp inspection trust
```

3. **Статические IP без ARP ACL**
```cisco
! Для статических IP нужен ARP ACL
arp access-list STATIC-HOSTS
 permit ip host 10.5.0.4 mac host xxxx.xxxx.xxxx
 
ip arp inspection filter STATIC-HOSTS vlan 30
```

---

## 📊 Security Violation Modes Comparison

| Mode | Forwards Traffic | Sends Log/SNMP | Error-disables Port |
|------|------------------|----------------|---------------------|
| **Protect** | ✅ Valid only | ❌ No | ❌ No |
| **Restrict** | ✅ Valid only | ✅ Yes | ❌ No |
| **Shutdown** | ❌ No | ✅ Yes | ✅ Yes |

💡 **Рекомендация:** Используйте **Restrict** - блокирует invalid + логирует, но не отключает порт.

---

## 📊 DHCP Snooping Rate Limits

| Device Type | Recommended Rate Limit |
|-------------|------------------------|
| PC | 10-15 pps |
| IP Phone | 10-15 pps |
| PC + Phone | 15-20 pps |
| WLC | 50-100 pps |
| Server | 20-30 pps |
| Printer | 5-10 pps |

💡 Если rate limit превышен, порт переходит в err-disabled.

---

## 📊 ACL Placement Guidelines

| ACL Type | Placement | Reason |
|----------|-----------|--------|
| **Standard ACL** | Close to destination | Filters only by source IP |
| **Extended ACL** | Close to source | Filters by src/dst IP + protocol + port |

**В нашей лабе:**
- Extended ACL на DSW-A1/A2 VLAN 10 SVI
- Близко к источнику (Office A PCs)
- Фильтрует трафик до того как он пройдёт через сеть

---

## 🎯 Чек-лист завершения Части 7

### Extended ACL:
- [ ] ACL создан на DSW-A1 (OfficeA_to_OfficeB)
- [ ] ACL создан на DSW-A2 (OfficeA_to_OfficeB)
- [ ] Правило 1: Permit ICMP (10.1.0.0/24 → 10.3.0.0/24)
- [ ] Правило 2: Deny IP (10.1.0.0/24 → 10.3.0.0/24)
- [ ] Правило 3: Permit IP (any → any)
- [ ] ACL применён на VLAN 10 SVI (inbound)
- [ ] Ping работает (PC1 → PC3)
- [ ] Telnet/SSH блокируется (PC1 → PC3)
- [ ] Ping другим сетям работает

### Port Security:
- [ ] Port Security на ASW-A1 F0/1 (1 MAC, restrict, sticky)
- [ ] Port Security на ASW-A2 F0/1 (2 MACs, restrict, sticky)
- [ ] Port Security на ASW-A3 F0/1 (2 MACs, restrict, sticky)
- [ ] Port Security на ASW-B1 F0/1 (1 MAC, restrict, sticky)
- [ ] Port Security на ASW-B2 F0/1 (2 MACs, restrict, sticky)
- [ ] Port Security на ASW-B3 F0/1 (1 MAC, restrict, sticky)
- [ ] Sticky MACs изучены и сохранены
- [ ] Violation count = 0 на всех портах

### DHCP Snooping:
- [ ] DHCP Snooping включен на всех ASW
- [ ] VLANs настроены (Office A: 10,20,40,99 | Office B: 10,20,30,99)
- [ ] Option 82 отключен на всех ASW
- [ ] G0/1-2 trusted на всех ASW
- [ ] Rate limit 15 pps на F0/1 (кроме ASW-A1)
- [ ] Rate limit 100 pps на ASW-A1 F0/2 (WLC)
- [ ] DHCP Binding Table содержит записи
- [ ] DHCP работает на всех клиентах

### DAI:
- [ ] DAI включен на всех ASW
- [ ] VLANs настроены (Office A: 10,20,40,99 | Office B: 10,20,30,99)
- [ ] Validation checks: dst-mac, src-mac, ip
- [ ] G0/1-2 trusted на всех ASW
- [ ] DAI statistics показывают 0 dropped packets
- [ ] ARP работает нормально

### General:
- [ ] Все конфигурации сохранены (write memory)
- [ ] Connectivity работает end-to-end
- [ ] Security features не блокируют legitimate traffic
- [ ] Verification commands выполнены

---

## 📚 Дополнительная теория

### Port Security Sticky MAC

**Как работает:**
1. Порт изучает MAC адрес
2. MAC автоматически добавляется в running-config
3. После `write`, MAC сохраняется в startup-config
4. После перезагрузки, MAC уже настроен

**Преимущества:**
- Не нужно вручную вводить MAC адреса
- Автоматическое сохранение легитимных MACs
- Защита от MAC spoofing

---

### DHCP Snooping Binding Table

**Содержит:**
- MAC адрес клиента
- IP адрес полученный через DHCP
- Lease time
- VLAN
- Interface

**Используется:**
- DAI проверяет ARP против этой таблицы
- IP Source Guard (не в лабе)
- Защита от IP spoofing

---

### DAI Validation Checks

**dst-mac:** Проверяет destination MAC в ARP против Ethernet header

**src-mac:** Проверяет source MAC в ARP против Ethernet header

**ip:** Проверяет IP адреса в ARP против DHCP Snooping Binding Table

💡 **Best Practice:** Включайте все три проверки!

---

## 🔐 Security Best Practices Summary

### Defense in Depth (Эшелонированная защита):

**Layer 2 Security:**
1. ✅ Port Security - ограничение MAC адресов
2. ✅ DHCP Snooping - защита от rogue DHCP
3. ✅ DAI - защита от ARP spoofing
4. ✅ Disable unused ports
5. ✅ BPDU Guard - защита от rogue switches

**Layer 3 Security:**
1. ✅ ACLs - фильтрация трафика
2. ✅ Private VLANs (advanced, не в лабе)
3. ✅ uRPF - Unicast Reverse Path Forwarding (advanced)

**Access Security:**
1. ✅ SSH only (no Telnet)
2. ✅ Strong passwords
3. ✅ AAA authentication (advanced, не в лабе)
4. ✅ ACL на VTY lines
5. ✅ Privilege levels (advanced)

**Monitoring:**
1. ✅ Syslog централизованное логирование
2. ✅ SNMP мониторинг
3. ✅ NTP синхронизация времени (для корреляции логов)

---

## 💡 Real-World Scenarios

### Scenario 1: Guest Network Isolation

**Требование:** Гости могут выходить в Internet, но не в corporate LAN

**Решение:**
```cisco
ip access-list extended GUEST-ACL
 deny ip 10.7.0.0 0.0.255.255 10.0.0.0 0.255.255.255
 permit ip any any
 
interface vlan 70
 ip access-group GUEST-ACL in
```

---

### Scenario 2: Server Protection

**Требование:** Только определённые хосты могут SSH к серверам

**Решение:**
```cisco
ip access-list extended SERVER-PROTECTION
 permit tcp 10.0.0.0 0.0.0.255 10.5.0.0 0.0.0.255 eq 22
 deny tcp any 10.5.0.0 0.0.0.255 eq 22
 permit ip any any
 
interface vlan 30
 ip access-group SERVER-PROTECTION in
```

---

### Scenario 3: BYOD Security

**Требование:** BYOD устройства в отдельном VLAN с Port Security

**Решение:**
```cisco
interface range f0/10-20
 switchport mode access
 switchport access vlan 80
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

---

## 📖 Exam Tips для CCNA

### ACL Tips:
1. ⚠️ Implicit DENY в конце каждого ACL
2. ⚠️ Wildcard mask ≠ Subnet mask (инвертирован)
3. ⚠️ ACL читается top-down (первое совпадение)
4. ⚠️ Extended ACL: src, dst порядок важен
5. ⚠️ `0.0.0.0` wildcard = точный IP (host)

### Port Security Tips:
1. ⚠️ Default maximum = 1 MAC
2. ⚠️ Default violation mode = shutdown
3. ⚠️ Sticky MACs появляются в running-config
4. ⚠️ Port security требует `switchport mode access/trunk`
5. ⚠️ Violation counter НЕ сбрасывается автоматически

### DHCP Snooping Tips:
1. ⚠️ По умолчанию все порты untrusted
2. ⚠️ Option 82 должно быть отключено (`no ip dhcp snooping information option`)
3. ⚠️ Trust uplinks к DHCP серверу
4. ⚠️ Rate limit предотвращает DHCP DoS
5. ⚠️ Binding table очищается после reboot

### DAI Tips:
1. ⚠️ Требует DHCP Snooping
2. ⚠️ По умолчанию все порты untrusted
3. ⚠️ Trust те же порты что и для DHCP Snooping
4. ⚠️ Validation checks опциональны, но рекомендованы
5. ⚠️ Статические IP требуют ARP ACL

---

## 🎓 Final Security Commands Reference

```cisco
# Quick Security Setup на Access Switch
!
! Port Security
interface f0/1
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
!
! DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,99
no ip dhcp snooping information option
interface range g0/1-2
 ip dhcp snooping trust
interface f0/1
 ip dhcp snooping limit rate 15
!
! DAI
ip arp inspection vlan 10,20,30,99
ip arp inspection validate dst-mac src-mac ip
interface range g0/1-2
 ip arp inspection trust
!
! Save
do write
```

---

## 🚀 Готовы к Части 8?

В следующей части мы настроим **IPv6**:

### IPv6 (Часть 8):
1. 🌐 **IPv6 Addressing** - настройка адресов на интерфейсах
2. 🔧 **EUI-64** - автоматическая генерация Interface ID
3. 🔗 **Link-Local Addresses** - fe80::/10
4. 📡 **IPv6 Routing** - включение unicast routing
5. 🛣️ **Static Routes** - IPv6 default routes

Это будет короткая, но важная часть!

**До встречи в Части 8! 🎓**