# DHCP Server, DHCP Relay и защита DHCP

Основные варианты:

1. **Локальная подсеть или VLAN** — DHCP Server на MikroTik через WebFig.
2. **Единый сервер для нескольких подсетей** — Windows Server DHCP и DHCP Relay на MikroTik.
3. **Rapier или Windows RRAS** — альтернативные реализации DHCP Relay.
4. **Linux Kea DHCP** — резервная серверная реализация.

Перед началом подготовьте оборудование по [`00-basic-setup`](../00-basic-setup/) и VLAN по [`02-vlan`](../02-vlan/).

Обозначения:

* `<CLIENT_INTERFACE>` — интерфейс или VLAN клиентов;
* `<SERVER_IP>` — адрес центрального DHCP-сервера;
* `<GATEWAY>` — адрес шлюза клиентской подсети;
* `<POOL_START>` — первый адрес динамического диапазона;
* `<POOL_END>` — последний адрес динамического диапазона;
* `<CLIENT_MAC>` — MAC-адрес клиента;
* `<DNS_IP>` — адрес DNS-сервера;
* `<DOMAIN>` — доменное имя, выдаваемое клиентам.

Основной пример:

```text
Серверная сеть:
192.168.100.0/24

Windows DHCP Server:
192.168.100.10/24
Gateway: 192.168.100.1

VLAN 20 — Staff:
192.168.20.0/24
Gateway: 192.168.20.1
Pool: 192.168.20.100–192.168.20.200

VLAN 30 — Guest:
192.168.30.0/24
Gateway: 192.168.30.1
Pool: 192.168.30.100–192.168.30.200
```

---

## Содержание

1. [Выбор схемы DHCP](#scheme-selection)
2. [DHCP Server на MikroTik](#mikrotik-server)
3. [Статические аренды MikroTik](#mikrotik-static-lease)
4. [Несколько DHCP-серверов для VLAN](#mikrotik-multiple-vlan)
5. [Центральный Windows DHCP Server](#windows-server)
6. [DHCP Relay на MikroTik](#mikrotik-relay)
7. [DHCP Relay для нескольких VLAN](#multiple-relays)
8. [DHCP Relay на Rapier](#rapier-relay)
9. [DHCP Relay на Windows RRAS](#windows-relay)
10. [Разные параметры для POS, KIOSK и PC](#client-types)
11. [DHCP и VLAN](#dhcp-vlan)
12. [DHCP Snooping и защита DHCP](#dhcp-security)
13. [Linux DHCP Server на Kea](#linux-kea)
14. [Настройка DHCP-клиентов](#clients)
15. [Очистка конфигурации](#cleanup)
16. [Краткое соответствие параметров](#mapping)
17. [Минимальные алгоритмы](#minimal-algorithms)

---

<a id="scheme-selection"></a>

## 1. Выбор схемы DHCP

| Условие                                    | Реализация                           |
| ------------------------------------------ | ------------------------------------ |
| Одна локальная сеть за MikroTik            | DHCP Server на MikroTik              |
| Несколько VLAN на одном MikroTik           | отдельный DHCP Server на каждой VLAN |
| Один центральный сервер для всех сегментов | Windows DHCP + MikroTik Relay        |
| DHCP Server находится в другой подсети     | DHCP Relay                           |
| Маршрутизатором является Rapier            | BOOTP/DHCP Relay на Rapier           |
| Маршрутизатором является Windows Server    | DHCP Relay Agent в RRAS              |
| Доступен только Linux                      | Kea DHCP Server                      |
| Нужно запретить посторонние DHCP-серверы   | DHCP Snooping                        |

### Основные правила

1. В одной клиентской VLAN должен работать только один ожидаемый DHCP-сервис.
2. На одном клиентском интерфейсе не запускайте одновременно:

   * локальный DHCP Server;
   * DHCP Relay.
3. Адрес шлюза и адреса самого сервера не включайте в динамический пул.
4. Для каждой маршрутизируемой подсети требуется отдельный Scope или DHCP Network.
5. DHCP Relay должен слушать интерфейс, обращённый к клиентам.
6. Между relay и сервером должна существовать маршрутизация в обе стороны.

---

<a id="mikrotik-server"></a>

## 2. DHCP Server на MikroTik

### 2.1. Схема

```text
PC-A ─── ether2 MikroTik
PC-B ─── ether3 MikroTik
```

Порты `ether2` и `ether3` входят в один bridge:

```text
br-staff
```

Адресный план:

```text
Сеть:       192.168.20.0/24
Шлюз:       192.168.20.1
DHCP-пул:   192.168.20.100–192.168.20.200
DNS:        192.168.20.1
Домен:      staff.local
```

DHCP Server создаётся на логическом интерфейсе клиентской сети:

* bridge;
* VLAN-интерфейсе;
* отдельном физическом L3-интерфейсе.

Не создавайте DHCP Server на физическом порту, если этот порт является участником bridge. В таком случае выбирайте сам bridge или VLAN-интерфейс.

### 2.2. Назначить шлюз клиентской сети

Откройте:

> **IP → Addresses → Add New**

Укажите:

| Поле      | Значение          |
| --------- | ----------------- |
| Address   | `192.168.20.1/24` |
| Interface | `br-staff`        |

Если используется VLAN:

```text
Address: 192.168.20.1/24
Interface: vlan20-staff
```

После этого автоматически появится connected route:

```text
192.168.20.0/24
```

### 2.3. Создать IP Pool

Откройте:

> **IP → Pool → Add New**

Укажите:

| Поле      | Значение                        |
| --------- | ------------------------------- |
| Name      | `pool-staff`                    |
| Addresses | `192.168.20.100-192.168.20.200` |

В пул не включайте:

```text
192.168.20.1    — шлюз
192.168.20.2–99 — статические адреса
192.168.20.255  — broadcast
```

### 2.4. Создать DHCP Network

Откройте:

> **IP → DHCP Server → Networks → Add New**

Укажите:

| Поле        | Значение          |
| ----------- | ----------------- |
| Address     | `192.168.20.0/24` |
| Gateway     | `192.168.20.1`    |
| DNS Servers | `192.168.20.1`    |
| Domain      | `staff.local`     |

При использовании внешнего DNS:

```text
DNS Servers: 192.168.100.10
```

Можно указать несколько серверов:

```text
192.168.100.10, 1.1.1.1
```

Для внутренней сети первым указывайте внутренний DNS-сервер.

### 2.5. MikroTik как DNS forwarder

Если клиентам выдаётся адрес самого MikroTik:

```text
DNS Servers: 192.168.20.1
```

откройте:

> **IP → DNS**

Укажите внешние или внутренние upstream DNS:

```text
Servers: 192.168.100.10
```

Включите:

```text
Allow Remote Requests: yes
```

Не разрешайте DNS-запросы к MikroTik из недоверенной внешней сети.

### 2.6. Создать DHCP Server

Откройте:

> **IP → DHCP Server → DHCP → Add New**

Укажите:

| Поле          | Значение                      |
| ------------- | ----------------------------- |
| Name          | `dhcp-staff`                  |
| Interface     | `br-staff` или `vlan20-staff` |
| Address Pool  | `pool-staff`                  |
| Lease Time    | `1h`                          |
| Authoritative | `yes`                         |
| Disabled      | `no`                          |

Для постоянной внутренней сети можно задать:

```text
Lease Time: 1d
```

Для гостевой сети:

```text
Lease Time: 1h
```

### 2.7. Быстрая настройка через DHCP Setup

Вместо ручного создания можно использовать:

> **IP → DHCP Server → DHCP Setup**

Выберите клиентский интерфейс.

Мастер последовательно запросит:

1. DHCP Address Space;
2. Gateway;
3. Address Pool;
4. DNS Servers;
5. Lease Time.

После завершения проверьте созданные объекты:

* **IP → Pool**;
* **DHCP Server → Networks**;
* **DHCP Server → DHCP**.

Для экзамена ручная настройка удобнее, если нужно точно использовать заданные диапазоны.

### 2.8. Настроить клиента

На Windows или Linux включите:

```text
Получать IPv4 автоматически
Получать DNS автоматически
```

После получения адреса клиент должен иметь:

```text
IP:      192.168.20.100–192.168.20.200
Mask:    255.255.255.0
Gateway: 192.168.20.1
DNS:     192.168.20.1
```

### 2.9. Просмотр выданных адресов

Откройте:

> **IP → DHCP Server → Leases**

В таблице отображаются:

* IP-адрес;
* MAC-адрес;
* имя хоста;
* сервер;
* состояние;
* оставшееся время аренды.

Рабочая динамическая аренда обычно имеет статус:

```text
bound
```

---

<a id="mikrotik-static-lease"></a>

## 3. Статические аренды MikroTik

Статическая аренда нужна, когда одному устройству всегда должен выдаваться один адрес.

Примеры:

* сервер;
* кассовый терминал;
* принтер;
* информационный экран;
* устройство, для которого настроены firewall или QoS.

### 3.1. Преобразовать динамическую аренду

1. Подключите устройство и дождитесь получения адреса.

2. Откройте:

   > **IP → DHCP Server → Leases**

3. Выберите нужную аренду.

4. Нажмите:

   ```text
   Make Static
   ```

5. Откройте созданную запись.

6. При необходимости измените адрес.

Пример:

```text
MAC Address: 00:11:22:33:44:55
Address: 192.168.20.50
Server: dhcp-staff
```

### 3.2. Создать статическую аренду вручную

Откройте:

> **IP → DHCP Server → Leases → Add New**

Укажите:

| Поле        | Значение            |
| ----------- | ------------------- |
| Address     | `192.168.20.50`     |
| MAC Address | `00:11:22:33:44:55` |
| Server      | `dhcp-staff`        |

Статический адрес должен:

* принадлежать клиентской подсети;
* не использоваться другим устройством;
* не пересекаться с адресами, назначенными вручную.

Удобно выделить отдельный диапазон:

```text
192.168.20.2–99    — статические адреса
192.168.20.100–200 — динамический пул
```

---

<a id="mikrotik-multiple-vlan"></a>

## 4. Несколько DHCP-серверов для VLAN

### 4.1. Адресный план

|        VLAN | Подсеть           | Шлюз           | Пул         |
| ----------: | ----------------- | -------------- | ----------- |
| 10 Services | `192.168.10.0/24` | `192.168.10.1` | `.100–.200` |
|    20 Staff | `192.168.20.0/24` | `192.168.20.1` | `.100–.200` |
|    30 Guest | `192.168.30.0/24` | `192.168.30.1` | `.100–.200` |

Сначала создайте VLAN-интерфейсы:

```text
vlan10-services
vlan20-staff
vlan30-guest
```

### 4.2. Назначить шлюзы

> **IP → Addresses**

```text
192.168.10.1/24 → vlan10-services
192.168.20.1/24 → vlan20-staff
192.168.30.1/24 → vlan30-guest
```

### 4.3. Создать отдельные пулы

> **IP → Pool**

```text
pool-services: 192.168.10.100-192.168.10.200
pool-staff:    192.168.20.100-192.168.20.200
pool-guest:    192.168.30.100-192.168.30.200
```

### 4.4. Создать DHCP Networks

> **IP → DHCP Server → Networks**

```text
192.168.10.0/24
Gateway: 192.168.10.1
DNS: 192.168.100.10
Domain: services.local
```

```text
192.168.20.0/24
Gateway: 192.168.20.1
DNS: 192.168.100.10
Domain: staff.local
```

```text
192.168.30.0/24
Gateway: 192.168.30.1
DNS: 192.168.100.10
Domain: guest.local
```

### 4.5. Создать сервер для каждой VLAN

> **IP → DHCP Server → DHCP**

| Name            | Interface         | Pool            |
| --------------- | ----------------- | --------------- |
| `dhcp-services` | `vlan10-services` | `pool-services` |
| `dhcp-staff`    | `vlan20-staff`    | `pool-staff`    |
| `dhcp-guest`    | `vlan30-guest`    | `pool-guest`    |

Каждый DHCP Server слушает только свою VLAN.

---

<a id="windows-server"></a>

## 5. Центральный Windows DHCP Server

Центральный сервер обслуживает:

* свою локальную подсеть;
* удалённые VLAN через DHCP Relay.

### 5.1. Статический адрес сервера

Настройте сервер:

```text
IP:      192.168.100.10
Mask:    255.255.255.0
Gateway: 192.168.100.1
DNS:     <DNS_IP>
```

Сервер не должен получать собственный адрес по DHCP.

---

## 5.2. Установка DHCP на Windows Server 2003

Откройте:

> **Панель управления → Установка и удаление программ → Установка компонентов Windows**

Выберите:

> **Сетевые службы → Состав**

Отметьте:

```text
Dynamic Host Configuration Protocol (DHCP)
```

Завершите установку.

Откройте:

> **Администрирование → DHCP**

### Современный Windows Server

Откройте:

> **Server Manager → Manage → Add Roles and Features**

Выберите:

```text
Role-based or feature-based installation
→ DHCP Server
→ Add Features
→ Install
```

После установки откройте:

> **Server Manager → Tools → DHCP**

---

## 5.3. Авторизация DHCP-сервера

Если сервер входит в домен Active Directory:

1. Откройте оснастку DHCP.

2. Нажмите правой кнопкой на имя сервера.

3. Выберите:

   ```text
   Authorize
   ```

4. Обновите список.

В рабочем состоянии значок сервера должен стать зелёным.

Если сервер является автономным и домена Active Directory нет, авторизация не выполняется.

---

## 5.4. Создать Scope VLAN 20

В оснастке DHCP:

1. Раскройте имя сервера.

2. Нажмите правой кнопкой на **IPv4**.

3. Выберите:

   ```text
   New Scope
   ```

4. Укажите имя:

   ```text
   Staff VLAN 20
   ```

5. Укажите диапазон:

   ```text
   Start IP: 192.168.20.100
   End IP:   192.168.20.200
   Mask:     255.255.255.0
   ```

6. Добавьте исключения, например:

   ```text
   192.168.20.190–192.168.20.200
   ```

7. Укажите Lease Duration:

   ```text
   1 day
   ```

8. Выберите:

   ```text
   Yes, I want to configure these options now
   ```

### Option 003 — Router

```text
192.168.20.1
```

Это адрес шлюза именно VLAN 20, а не адрес Windows Server.

### Option 006 — DNS Servers

```text
192.168.100.10
```

или другой действующий DNS-сервер.

### Option 015 — DNS Domain Name

```text
staff.local
```

WINS можно пропустить, если он не используется.

В конце выберите:

```text
Activate this scope
```

---

## 5.5. Создать Scope VLAN 30

Повторите мастер:

```text
Name: Guest VLAN 30
Range: 192.168.30.100–192.168.30.200
Mask: 255.255.255.0
Router: 192.168.30.1
DNS: 192.168.100.10
Domain: guest.local
```

Для каждой удалённой IP-подсети создаётся собственный Scope.

---

## 5.6. Создать Reservation

1. Откройте нужный Scope.

2. Нажмите правой кнопкой:

   > **Reservations → New Reservation**

3. Укажите:

| Поле             | Значение               |
| ---------------- | ---------------------- |
| Reservation Name | `POS-01`               |
| IP Address       | `192.168.20.110`       |
| MAC Address      | `001122334455`         |
| Supported Types  | `Both` или `DHCP only` |

В Windows адрес Reservation должен находиться внутри диапазона Scope.

Не выдавайте этот же адрес вручную другому устройству.

---

## 5.7. Сервер выбирает Scope удалённой сети

DHCP Relay передаёт серверу адрес клиентского интерфейса в поле `giaddr`.

Пример:

```text
Relay VLAN 20: 192.168.20.1
```

Windows Server выбирает:

```text
Scope 192.168.20.0/24
```

Для VLAN 30 relay передаст:

```text
192.168.30.1
```

и сервер выберет Scope `192.168.30.0/24`.

На Windows Server не требуется физический сетевой адаптер в каждой клиентской VLAN.

---

<a id="mikrotik-relay"></a>

## 6. DHCP Relay на MikroTik

### 6.1. Схема

```text
Windows DHCP Server
192.168.100.10
        │
192.168.100.0/24
        │
     MikroTik
        │
vlan20-staff
192.168.20.1/24
        │
      Clients
```

### 6.2. Настроить клиентский интерфейс

Создайте VLAN или используйте существующий L3-интерфейс.

Назначьте шлюз:

> **IP → Addresses → Add New**

```text
Address: 192.168.20.1/24
Interface: vlan20-staff
```

### 6.3. Обеспечить маршрут до сервера

MikroTik должен иметь маршрут к:

```text
192.168.100.10
```

Windows Server должен иметь обратный маршрут к:

```text
192.168.20.0/24
```

Обычно Windows Server использует default gateway:

```text
192.168.100.1
```

Если этим шлюзом является MikroTik, отдельный статический маршрут на сервере не нужен.

### 6.4. Не запускать локальный DHCP Server

Откройте:

> **IP → DHCP Server → DHCP**

На `vlan20-staff` не должно быть активного локального DHCP Server.

На одном интерфейсе используйте либо Server, либо Relay.

### 6.5. Создать Relay

Откройте:

> **IP → DHCP Relay → Add New**

Укажите:

| Поле           | Значение         |
| -------------- | ---------------- |
| Name           | `relay-staff`    |
| Interface      | `vlan20-staff`   |
| DHCP Server    | `192.168.100.10` |
| Local Address  | `192.168.20.1`   |
| Add Relay Info | `no`             |
| Disabled       | `no`             |

`Local Address` — адрес MikroTik в клиентской подсети. По нему Windows Server определяет нужный Scope.

### 6.6. Local Address as Source IP

Обычно оставьте:

```text
Local Address as Src. IP: no
```

Включайте параметр только тогда, когда сервер или firewall должен видеть запросы с адреса `Local Address`.

В обычной топологии достаточно правильно настроенной маршрутизации.

### 6.7. Option 82

Для простой схемы:

```text
Add Relay Info: no
```

Включайте Option 82 только тогда, когда сервер настроен использовать:

* Circuit ID;
* Remote ID;
* данные порта и клиента.

---

<a id="multiple-relays"></a>

## 7. DHCP Relay для нескольких VLAN

Для каждой клиентской VLAN создаётся отдельная запись Relay.

### 7.1. VLAN 20

```text
Name: relay-staff
Interface: vlan20-staff
DHCP Server: 192.168.100.10
Local Address: 192.168.20.1
```

### 7.2. VLAN 30

```text
Name: relay-guest
Interface: vlan30-guest
DHCP Server: 192.168.100.10
Local Address: 192.168.30.1
```

### 7.3. VLAN 10

```text
Name: relay-services
Interface: vlan10-services
DHCP Server: 192.168.100.10
Local Address: 192.168.10.1
```

На Windows Server должны существовать:

```text
Scope 192.168.10.0/24
Scope 192.168.20.0/24
Scope 192.168.30.0/24
```

### 7.4. Несколько DHCP-серверов

В поле **DHCP Server** можно указать несколько адресов:

```text
192.168.100.10, 192.168.100.11
```

Relay отправит запрос каждому указанному серверу.

Используйте это только при согласованной серверной конфигурации, например DHCP Failover. Два независимых сервера с пересекающимися пулами создадут конфликт.

---

<a id="rapier-relay"></a>

## 8. DHCP Relay на Rapier

На старых Rapier функция называется BOOTP Relay, но используется и для DHCP.

Настройка выполняется через AlliedWare CLI.

### 8.1. Схема

```text
VLAN 20 clients
192.168.20.0/24
        │
Rapier VLAN20
192.168.20.1
        │
маршрутизируемая сеть
        │
Windows DHCP Server
192.168.100.10
```

### 8.2. Создать VLAN

```text
create vlan=STAFF vid=20
```

Добавить клиентские порты:

```text
delete vlan=default port=2,3
add vlan=STAFF port=2,3 frame=untagged
```

### 8.3. Включить IP-маршрутизацию

```text
enable ip
```

### 8.4. Назначить адрес VLAN

```text
add ip interface=vlan20 ipaddress=192.168.20.1 mask=255.255.255.0
```

Rapier должен иметь маршрут до `192.168.100.10`.

При необходимости добавьте default route:

```text
add ip route=0.0.0.0 mask=0.0.0.0 next=<NEXT_HOP>
```

Синтаксис маршрута может отличаться по версии AlliedWare. Перед вводом используйте:

```text
add ip route ?
```

### 8.5. Включить Relay

```text
enable bootp relay
```

Добавьте адрес сервера:

```text
add bootp relay=192.168.100.10
```

Rapier будет ретранслировать DHCP-запросы из локальных маршрутизируемых VLAN на указанный сервер.

### 8.6. Несколько серверов

```text
add bootp relay=192.168.100.11
```

Не добавляйте два независимых сервера с пересекающимися пулами.

### 8.7. Просмотр

```text
show bootp
```

или:

```text
show bootp relay
```

Доступная команда зависит от версии AlliedWare. Используйте:

```text
show bootp ?
```

### 8.8. Удаление Relay

```text
delete bootp relay=192.168.100.10
```

Если relay больше не используется:

```text
disable bootp relay
```

---

<a id="windows-relay"></a>

## 9. DHCP Relay на Windows RRAS

Windows Server должен иметь сетевой интерфейс в клиентской подсети либо VLAN.

### 9.1. Установить Routing and Remote Access

На Windows Server 2003 откройте:

> **Администрирование → Маршрутизация и удалённый доступ**

Нажмите правой кнопкой на имя сервера:

```text
Настроить и включить маршрутизацию и удалённый доступ
```

Выберите:

```text
Особая конфигурация
→ Маршрутизация локальной сети
```

Запустите службу.

На современном Windows Server сначала установите:

```text
Remote Access
└── Routing
```

через **Add Roles and Features**.

### 9.2. Добавить DHCP Relay Agent

В оснастке RRAS раскройте:

> **IPv4 → General**

Нажмите правой кнопкой:

```text
New Routing Protocol
```

Выберите:

```text
DHCP Relay Agent
```

### 9.3. Добавить клиентский интерфейс

Нажмите правой кнопкой:

> **DHCP Relay Agent → New Interface**

Выберите интерфейс клиентской сети, например:

```text
VLAN20
```

Оставьте пересылку DHCP включённой.

Для каждой клиентской сети добавьте соответствующий интерфейс.

### 9.4. Указать DHCP Server

Откройте:

> **DHCP Relay Agent → Properties**

Добавьте:

```text
192.168.100.10
```

### 9.5. Маршрутизация

Windows RRAS должен знать маршруты:

* к центральному DHCP Server;
* к клиентским подсетям.

Центральный сервер должен иметь обратный маршрут через RRAS.

---

<a id="client-types"></a>

## 10. Разные параметры для POS, KIOSK и PC

Необходимо выбрать способ в зависимости от требований.

| Требование                                              | Рекомендуемый способ                     |
| ------------------------------------------------------- | ---------------------------------------- |
| Известны MAC всех устройств                             | Reservation                              |
| Каждому известному устройству нужен фиксированный адрес | Reservation                              |
| Разным типам нужны разные DNS или другие options        | Vendor/User Class                        |
| Нужны разные динамические диапазоны автоматически       | Windows DHCP Policies или отдельные VLAN |
| Используется Windows Server 2003                        | Reservations либо разные VLAN            |
| Устройства должны быть в разных подсетях                | отдельные VLAN и Scope                   |
| Тип устройства нельзя надёжно определить                | Reservations                             |

---

## 10.1. Reservations по MAC

Пример разделения одной подсети:

```text
192.168.20.10–39   — POS
192.168.20.40–69   — KIOSK
192.168.20.100–200 — обычные PC
```

Для каждого POS создайте Reservation:

```text
POS-01 → 192.168.20.10
POS-02 → 192.168.20.11
```

Для KIOSK:

```text
KIOSK-01 → 192.168.20.40
KIOSK-02 → 192.168.20.41
```

Обычные PC получают адрес из общего динамического пула.

Это самый надёжный способ для Windows Server 2003.

---

## 10.2. User Class

На Windows-клиенте можно установить класс:

```cmd
ipconfig /setclassid "Local Area Connection" POS
```

Для просмотра:

```cmd
ipconfig /showclassid "Local Area Connection"
```

Удалить класс:

```cmd
ipconfig /setclassid "Local Area Connection"
```

На DHCP Server создайте User Class:

1. Откройте оснастку DHCP.

2. Нажмите правой кнопкой на **IPv4**.

3. Выберите:

   ```text
   Define User Classes
   ```

4. Создайте классы:

   ```text
   POS
   KIOSK
   PC
   ```

Классы удобно использовать для выдачи разных DHCP options.

Например:

```text
POS → отдельный DNS или сервер приложения
KIOSK → адрес сервера управления
PC → стандартные параметры
```

### Ограничение Windows Server 2003

На Windows Server 2003 User Class и Vendor Class предназначены прежде всего для разных наборов options. Они не являются надёжным способом автоматически выбирать отдельный диапазон адресов внутри одного Scope.

Для разных диапазонов используйте:

* Reservations;
* отдельные VLAN;
* более современный Windows Server с DHCP Policies.

---

## 10.3. Vendor Class

Vendor Class передаётся клиентом в DHCP Option 60.

Она может содержать:

```text
PXEClient
MSFT 5.0
идентификатор производителя устройства
```

На Windows Server:

1. Нажмите правой кнопкой на **IPv4**.

2. Выберите:

   ```text
   Define Vendor Classes
   ```

3. Создайте требуемый класс.

4. Настройте отдельные options для этого класса.

Vendor Class подходит только тогда, когда устройство действительно отправляет стабильное и известное значение Option 60.

---

## 10.4. DHCP Policies на современных Windows Server

На Windows Server 2012 и новее можно создавать DHCP Policies по условиям:

* MAC prefix;
* Vendor Class;
* User Class;
* Client Identifier;
* Relay Agent Information;
* имя клиента.

Откройте:

> **IPv4 → Scope → Policies → New Policy**

Пример:

```text
Policy: POS
Condition: MAC Address begins with <VENDOR_PREFIX>
Address Range: 192.168.20.10–192.168.20.39
```

Создайте отдельную политику для KIOSK.

Используйте этот вариант только на поддерживаемой версии Windows Server.

---

## 10.5. Отдельные VLAN

Наиболее масштабируемый вариант:

```text
VLAN 31 — POS
VLAN 32 — KIOSK
VLAN 33 — PC
```

Для каждой VLAN создайте:

* отдельную подсеть;
* отдельный Scope;
* отдельный Relay;
* собственные gateway и options.

Этот способ изменяет исходное требование «одна подсеть», поэтому применяйте его только когда разделение на подсети разрешено.

---

<a id="dhcp-vlan"></a>

## 11. DHCP и VLAN

### 11.1. Общая схема

```text
Windows DHCP Server
192.168.100.10
        │
   серверная сеть
        │
      MikroTik
   ┌────┼────┐
 VLAN10 VLAN20 VLAN30
```

### 11.2. Для каждой VLAN необходимо

1. Создать VLAN-интерфейс.
2. Назначить IP шлюза.
3. Создать Windows Scope.
4. Создать MikroTik DHCP Relay.
5. Передать VLAN по нужным trunk.
6. Настроить access-порты.
7. Не запускать локальный DHCP Server на той же VLAN.

### 11.3. Таблица конфигурации

| VLAN | MikroTik Interface | Local Address Relay | Windows Scope     |
| ---: | ------------------ | ------------------- | ----------------- |
|   10 | `vlan10-services`  | `192.168.10.1`      | `192.168.10.0/24` |
|   20 | `vlan20-staff`     | `192.168.20.1`      | `192.168.20.0/24` |
|   30 | `vlan30-guest`     | `192.168.30.1`      | `192.168.30.0/24` |

### 11.4. Динамическая VLAN через RADIUS

После назначения клиенту VLAN через RADIUS:

1. порт становится участником нужной VLAN;
2. клиент должен обновить DHCP-аренду;
3. запрос попадёт на Relay этой VLAN;
4. сервер выберет соответствующий Scope.

На Windows-клиенте:

```cmd
ipconfig /release
ipconfig /renew
```

---

<a id="dhcp-security"></a>

## 12. DHCP Snooping и защита DHCP

DHCP Snooping блокирует ответы посторонних DHCP-серверов на клиентских портах.

### 12.1. Trusted и untrusted порты

| Тип порта                                                  | Trusted              |
| ---------------------------------------------------------- | -------------------- |
| Порт к настоящему внешнему DHCP-серверу                    | yes                  |
| Trunk к коммутатору, за которым находится настоящий сервер | yes                  |
| Клиентский access-порт                                     | no                   |
| Порт с неизвестным устройством                             | no                   |
| Обычный пользовательский trunk вниз                        | зависит от топологии |

По умолчанию bridge-порты считаются untrusted.

### 12.2. MikroTik: включить DHCP Snooping

Откройте:

> **Bridge → Bridge → `<BRIDGE>`**

Установите:

```text
DHCP Snooping: yes
```

Включение DHCP Snooping отключает Bridge Fast Path и может увеличить нагрузку на CPU.

На RB1100AHx4 не следует рассчитывать на полную аппаратную обработку DHCP Snooping. Используйте функцию прежде всего для учебной или умеренно нагруженной сети.

### 12.3. Назначить trusted-порт

Откройте:

> **Bridge → Ports → `<SERVER_OR_UPLINK_PORT>`**

Установите:

```text
Trusted: yes
```

Пример:

```text
ether1 — trunk к настоящему DHCP-серверу
Trusted: yes
```

На клиентских портах оставьте:

```text
Trusted: no
```

### 12.4. Несколько коммутаторов

```text
DHCP Server
    │
   SW1
    │
   SW2
    │
 Clients
```

На SW1:

* порт к серверу — trusted;
* trunk к SW2 — DHCP-сообщения сервера выходят через него.

На SW2:

* trunk к SW1 — trusted;
* клиентские порты — untrusted.

Если через порт приходят DHCP-сообщения с Option 82, этот порт также должен быть trusted.

### 12.5. Option 82

Option 82 может передавать серверу:

* Circuit ID;
* Remote ID;
* имя интерфейса;
* VLAN ID;
* идентификатор коммутатора.

В RouterOS 7.23 и новее настройте на bridge:

```text
DHCP Agent Circuit ID: $(INTERFACE):$(VID)
DHCP Agent Remote ID: $(HOSTNAME):$(BRIDGEMAC)
```

Включайте Option 82 только тогда, когда сервер настроен принимать и использовать эту информацию.

Для обычного DHCP Snooping Option 82 не обязателен.

---

## 12.6. DHCP Snooping на Rapier

Включить DHCP Snooping:

```text
enable dhcpsnooping
```

Если сервер находится за портом 25:

```text
set dhcpsnooping port=25 trusted=yes
```

Клиентские порты по умолчанию остаются untrusted.

Ограничить количество аренд за клиентским портом:

```text
set dhcpsnooping port=1-24 maxlease=1
```

При необходимости включить Option 82:

```text
enable dhcpsnooping option82
```

Просмотр:

```text
show dhcpsnooping
show dhcpsnooping database
```

Доступные команды зависят от версии AlliedWare:

```text
show dhcpsnooping ?
```

### 12.7. Защита от DHCP starvation

DHCP Snooping блокирует rogue DHCP Server, но сам по себе не предотвращает исчерпание пула множеством поддельных клиентов.

Дополнительные меры:

* Port Security;
* ограничение количества MAC на клиентском порту;
* `maxlease=1` на Rapier;
* 802.1X или MAC-аутентификация;
* короткие клиентские сегменты;
* наблюдение за числом выданных аренд.

Настройка Port Security находится в [`01-l2-switching`](../01-l2-switching/).

---

<a id="linux-kea"></a>

## 13. Linux DHCP Server на Kea

Резервная ветка для Debian или Ubuntu.

### 13.1. Схема

```text
Kea Server:
192.168.100.10/24
Interface: enp2s0

Локальная сеть:
192.168.100.0/24

Удалённые сети через Relay:
192.168.20.0/24
192.168.30.0/24
```

### 13.2. Установить Kea

```bash
sudo apt update
sudo apt install kea-dhcp4-server
```

Основной конфигурационный файл:

```text
/etc/kea/kea-dhcp4.conf
```

### 13.3. Создать резервную копию

```bash
sudo cp /etc/kea/kea-dhcp4.conf \
  /etc/kea/kea-dhcp4.conf.backup
```

### 13.4. Открыть конфигурацию

```bash
sudo nano /etc/kea/kea-dhcp4.conf
```

Используйте:

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [
        "enp2s0"
      ]
    },

    "renew-timer": 1800,
    "rebind-timer": 3150,
    "valid-lifetime": 3600,

    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.100.0/24",
        "interface": "enp2s0",
        "pools": [
          {
            "pool": "192.168.100.100 - 192.168.100.150"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.100.1"
          },
          {
            "name": "domain-name-servers",
            "data": "192.168.100.10"
          },
          {
            "name": "domain-name",
            "data": "services.local"
          }
        ]
      },

      {
        "id": 20,
        "subnet": "192.168.20.0/24",
        "pools": [
          {
            "pool": "192.168.20.100 - 192.168.20.200"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.20.1"
          },
          {
            "name": "domain-name-servers",
            "data": "192.168.100.10"
          },
          {
            "name": "domain-name",
            "data": "staff.local"
          }
        ],
        "reservations": [
          {
            "hw-address": "00:11:22:33:44:55",
            "ip-address": "192.168.20.110",
            "hostname": "pos-01"
          }
        ]
      },

      {
        "id": 30,
        "subnet": "192.168.30.0/24",
        "pools": [
          {
            "pool": "192.168.30.100 - 192.168.30.200"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.30.1"
          },
          {
            "name": "domain-name-servers",
            "data": "192.168.100.10"
          },
          {
            "name": "domain-name",
            "data": "guest.local"
          }
        ]
      }
    ]
  }
}
```

### 13.5. Relay и выбор подсети

Для удалённого клиента MikroTik передаёт:

```text
giaddr = 192.168.20.1
```

Kea выбирает:

```text
subnet 192.168.20.0/24
```

Для VLAN 30:

```text
giaddr = 192.168.30.1
```

Kea выбирает:

```text
subnet 192.168.30.0/24
```

Специально указывать relay в конфигурации не требуется, если его `local-address` принадлежит соответствующей клиентской подсети.

### 13.6. Проверить синтаксис

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

При успешной проверке команда не должна выводить ошибку конфигурации.

### 13.7. Запустить службу

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl enable kea-dhcp4-server
```

Состояние:

```bash
systemctl status kea-dhcp4-server
```

### 13.8. Firewall Linux

Если используется UFW:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 67 proto udp
```

Если запросы Relay приходят из других маршрутизируемых сетей, разрешите адреса самих relay.

Не открывайте UDP 67 на внешнем публичном интерфейсе.

---

<a id="clients"></a>

## 14. Настройка DHCP-клиентов

### 14.1. Windows

Откройте:

> **Сетевые подключения → Ethernet → Свойства → Internet Protocol TCP/IPv4**

Выберите:

```text
Получить IP-адрес автоматически
Получить адрес DNS-сервера автоматически
```

Обновить аренду:

```cmd
ipconfig /release
ipconfig /renew
```

Просмотреть параметры:

```cmd
ipconfig /all
```

### 14.2. Linux через интерфейс

Откройте:

> **Настройки → Сеть → Проводное подключение → IPv4**

Выберите:

```text
Автоматически (DHCP)
```

Переподключите интерфейс.

### 14.3. Linux через NetworkManager

```bash
sudo nmcli connection modify "<CONNECTION>" ipv4.method auto
sudo nmcli connection down "<CONNECTION>"
sudo nmcli connection up "<CONNECTION>"
```

---

<a id="cleanup"></a>

## 15. Очистка конфигурации

### 15.1. MikroTik DHCP Server

Удаляйте в следующем порядке:

1. отключите DHCP Server;
2. удалите статические Leases;
3. удалите DHCP Server;
4. удалите DHCP Network;
5. удалите IP Pool;
6. удалите IP шлюза только тогда, когда сеть больше не используется.

Не удаляйте VLAN-интерфейс или bridge, если они нужны другим заданиям.

### 15.2. MikroTik Relay

1. Откройте:

   > **IP → DHCP Relay**

2. Отключите и удалите созданные Relay.

3. Не удаляйте IP шлюза VLAN, если маршрутизация продолжает использоваться.

### 15.3. DHCP Snooping MikroTik

1. Откройте:

   > **Bridge → Bridge**

2. Установите:

   ```text
   DHCP Snooping: no
   ```

3. Откройте:

   > **Bridge → Ports**

4. Верните `Trusted` к исходным значениям.

### 15.4. Windows DHCP Server

Для удаления Scope:

1. Откройте оснастку DHCP.

2. Раскройте **IPv4**.

3. Нажмите правой кнопкой на Scope.

4. Выберите:

   ```text
   Deactivate
   ```

5. Затем выберите:

   ```text
   Delete
   ```

Не удаляйте общие рабочие Scope других сегментов.

### 15.5. Windows RRAS Relay

1. Откройте:

   > **Routing and Remote Access → IPv4 → DHCP Relay Agent**

2. Удалите клиентские интерфейсы.

3. Удалите адрес DHCP Server из Properties.

4. Удалите DHCP Relay Agent, если он больше не нужен.

5. Не отключайте RRAS, если он используется для маршрутизации.

### 15.6. Rapier

```text
delete bootp relay=<SERVER_IP>
disable bootp relay
```

Отключить DHCP Snooping:

```text
disable dhcpsnooping
```

Не удаляйте IP-интерфейсы VLAN, если они используются как шлюзы.

### 15.7. Linux Kea

Остановить сервер:

```bash
sudo systemctl stop kea-dhcp4-server
```

Отключить автозапуск:

```bash
sudo systemctl disable kea-dhcp4-server
```

Вернуть конфигурацию:

```bash
sudo cp /etc/kea/kea-dhcp4.conf.backup \
  /etc/kea/kea-dhcp4.conf
```

---

<a id="mapping"></a>

## 16. Краткое соответствие параметров

| Задача                | MikroTik                        | Windows Server              | Rapier                | Kea                       |
| --------------------- | ------------------------------- | --------------------------- | --------------------- | ------------------------- |
| Диапазон адресов      | IP Pool                         | Scope Range                 | центральный сервер    | `pools`                   |
| Шлюз                  | DHCP Network                    | Option 003                  | центральный сервер    | `routers`                 |
| DNS                   | DHCP Network                    | Option 006                  | центральный сервер    | `domain-name-servers`     |
| Домен                 | DHCP Network                    | Option 015                  | центральный сервер    | `domain-name`             |
| Время аренды          | Lease Time                      | Lease Duration              | центральный сервер    | `valid-lifetime`          |
| Фиксированный адрес   | Static Lease                    | Reservation                 | центральный сервер    | `reservations`            |
| Relay                 | IP → DHCP Relay                 | RRAS Relay Agent            | BOOTP Relay           | сервер принимает `giaddr` |
| Scope удалённой сети  | DHCP Network/Pool               | отдельный Scope             | —                     | отдельный `subnet4`       |
| Rogue DHCP protection | DHCP Snooping                   | AD authorization            | DHCP Snooping         | firewall/управляемая сеть |
| Trusted port          | Bridge Port Trusted             | —                           | `trusted=yes`         | —                         |
| Классы клиентов       | options/matcher                 | Vendor/User Class, Policies | Option 82             | Client Classes            |
| Несколько VLAN        | Server или Relay на каждой VLAN | Scope на каждую подсеть     | Relay для routed VLAN | `subnet4` на каждую сеть  |

---

<a id="minimal-algorithms"></a>

## 17. Минимальные алгоритмы

### 17.1. Локальный DHCP Server на MikroTik

```text
1. Назначить интерфейсу IP шлюза.
2. Создать IP Pool.
3. Создать DHCP Network.
4. Указать gateway, DNS и domain.
5. Создать DHCP Server на правильном интерфейсе.
6. Включить DHCP на клиенте.
7. При необходимости сделать Lease статической.
```

### 17.2. Windows DHCP и MikroTik Relay

```text
1. Назначить Windows Server статический IP.
2. Установить DHCP Server.
3. Авторизовать сервер, если используется Active Directory.
4. Создать Scope для клиентской подсети.
5. Указать Router, DNS и Domain.
6. На MikroTik назначить IP шлюза клиентской VLAN.
7. Обеспечить маршруты между MikroTik и сервером.
8. Создать DHCP Relay на клиентском VLAN-интерфейсе.
9. Указать Windows Server и Local Address.
10. Не запускать локальный DHCP Server на той же VLAN.
```

### 17.3. Несколько VLAN

```text
Для каждой VLAN:
1. VLAN-интерфейс.
2. IP шлюза.
3. Windows Scope.
4. MikroTik Relay.
5. VLAN по trunk.
6. Access-порты.
```

### 17.4. Защита от rogue DHCP

```text
1. Включить DHCP Snooping на bridge.
2. Порт к настоящему серверу или upstream сделать Trusted.
3. Клиентские порты оставить Untrusted.
4. Не включать Option 82 без настройки сервера.
5. Дополнить защиту ограничением MAC на клиентских портах.
```
