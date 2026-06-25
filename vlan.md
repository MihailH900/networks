# VLAN, trunk и межвлановая маршрутизация

Основная настройка выполняется на MikroTik через WebFig. Для коммутаторов Rapier приведена отдельная ветка AlliedWare CLI.

Перед началом выполните подготовку из [`00-basic-setup`](../00-basic-setup/).

Обозначения:

* `<BRIDGE>` — VLAN-aware bridge;
* `<ACCESS_PORT>` — порт конечного устройства;
* `<TRUNK_PORT>` — порт между коммутаторами или к маршрутизатору;
* `<VLAN_ID>` — номер VLAN;
* `<SUBNET>` — IP-подсеть VLAN;
* `<GATEWAY>` — IP-адрес шлюза VLAN;
* `<BOND>` — агрегированный LACP-интерфейс.

В основных примерах используются:

```text
VLAN 10 — Services — 192.168.10.0/24
VLAN 20 — Staff    — 192.168.20.0/24
VLAN 30 — Guest    — 192.168.30.0/24
VLAN 100 — Management
VLAN 999 — Quarantine
```

Для обычных пользовательских VLAN не используйте номера `0`, `1` и `4095`.

---

## Содержание

1. [Планирование VLAN](#planning)
2. [VLAN на одном MikroTik](#single-mikrotik)
3. [VLAN между двумя MikroTik](#two-mikrotik)
4. [VLAN через LACP](#vlan-lacp)
5. [Межвлановая маршрутизация на MikroTik](#inter-vlan)
6. [Router-on-a-stick](#router-on-stick)
7. [VLAN на Rapier](#rapier)
8. [GVRP на Rapier и MVRP на MikroTik](#gvrp-mvrp)
9. [Динамическая VLAN через RADIUS](#radius-vlan)
10. [Очистка конфигурации](#cleanup)
11. [Краткое соответствие параметров](#mapping)

---

<a id="planning"></a>

## 1. Планирование VLAN

До настройки составьте три таблицы:

1. таблицу VLAN;
2. таблицу физических портов;
3. таблицу IP-адресов.

### 1.1. Таблица VLAN

| VLAN ID | Назначение | Подсеть            | Шлюз            |
| ------: | ---------- | ------------------ | --------------- |
|      10 | Services   | `192.168.10.0/24`  | `192.168.10.1`  |
|      20 | Staff      | `192.168.20.0/24`  | `192.168.20.1`  |
|      30 | Guest      | `192.168.30.0/24`  | `192.168.30.1`  |
|     100 | Management | `192.168.100.0/24` | `192.168.100.1` |
|     999 | Quarantine | `192.168.99.0/24`  | `192.168.99.1`  |

Шлюзы нужны только при маршрутизации между VLAN или выходе в другие сети.

### 1.2. Таблица портов

Пример одного MikroTik:

| Порт      | Подключение       | Режим      | VLAN                |
| --------- | ----------------- | ---------- | ------------------- |
| `ether2`  | PC Staff 1        | access     | 20 untagged         |
| `ether3`  | PC Staff 2        | access     | 20 untagged         |
| `ether4`  | сервер            | access     | 10 untagged         |
| `ether5`  | второй коммутатор | trunk      | 10, 20, 30 tagged   |
| `ether10` | администратор     | management | вне учебного bridge |

### 1.3. Access-порт

Access-порт используется для обычного компьютера, принтера, кассы или другого устройства без настройки VLAN.

Он должен:

```text
принимать untagged-кадры
назначать им один PVID
передавать кадры этой VLAN без тега
запрещать произвольные tagged-кадры клиента
```

Один access-порт обычно принадлежит только одной untagged VLAN.

### 1.4. Trunk-порт

Trunk передаёт несколько VLAN между:

* двумя коммутаторами;
* коммутатором и маршрутизатором;
* коммутатором и сервером с VLAN-подинтерфейсами;
* двумя устройствами через LACP.

На trunk кадры передаются с тегами 802.1Q.

### 1.5. PVID

`PVID` назначается входящему кадру, пришедшему без VLAN-тега.

Пример:

```text
ether2 имеет PVID 20
```

Нетегированный кадр с `ether2` внутри bridge обрабатывается как кадр VLAN 20.

PVID не определяет сам по себе, через какие порты VLAN может выходить. Это задаётся отдельно в **Bridge VLAN Table**.

### 1.6. Management

Самый безопасный вариант:

```text
management-порт не входит в VLAN-aware bridge
```

Если управление должно идти через VLAN 100, сначала полностью настройте management VLAN и убедитесь, что новый IP доступен. Только затем включайте `VLAN Filtering`.

---

<a id="single-mikrotik"></a>

## 2. VLAN на одном MikroTik

### 2.1. Задача

Создать две изолированные VLAN:

```text
VLAN 10 — Services
VLAN 20 — Staff
```

Схема:

```text
PC-S1 ─ ether2       ether4 ─ PC-A1
          \           /
             MikroTik
          /           \
PC-S2 ─ ether3       ether5 ─ PC-A2
```

Назначение портов:

| Порт     | VLAN | Режим  |
| -------- | ---: | ------ |
| `ether2` |   10 | access |
| `ether3` |   10 | access |
| `ether4` |   20 | access |
| `ether5` |   20 | access |

Адреса:

| Хост  | IP                 |
| ----- | ------------------ |
| PC-S1 | `192.168.10.10/24` |
| PC-S2 | `192.168.10.20/24` |
| PC-A1 | `192.168.20.10/24` |
| PC-A2 | `192.168.20.20/24` |

Шлюз пока не задаётся.

### 2.2. Проверить старую конфигурацию

Откройте:

> **Bridge → Ports**

Убедитесь, что `ether2–ether5` не находятся в другом bridge.

Откройте:

> **Interfaces**

Убедитесь, что эти порты:

* не входят в bonding;
* не имеют собственных IP;
* не используются для управления.

### 2.3. Создать bridge

Откройте:

> **Bridge → Bridge → Add New**

Укажите:

| Поле           | Значение  |
| -------------- | --------- |
| Name           | `br-vlan` |
| Protocol Mode  | `rstp`    |
| VLAN Filtering | `no`      |

Сначала фильтрация должна быть выключена.

### 2.4. Добавить access-порты VLAN 10

Откройте:

> **Bridge → Ports → Add New**

Для `ether2`:

| Поле              | Значение                                  |
| ----------------- | ----------------------------------------- |
| Interface         | `ether2`                                  |
| Bridge            | `br-vlan`                                 |
| PVID              | `10`                                      |
| Ingress Filtering | `yes`                                     |
| Frame Types       | `admit only untagged and priority tagged` |
| Edge              | `yes`                                     |

Повторите для `ether3`.

### 2.5. Добавить access-порты VLAN 20

Для `ether4` и `ether5`:

| Поле              | Значение                                  |
| ----------------- | ----------------------------------------- |
| Bridge            | `br-vlan`                                 |
| PVID              | `20`                                      |
| Ingress Filtering | `yes`                                     |
| Frame Types       | `admit only untagged and priority tagged` |
| Edge              | `yes`                                     |

### 2.6. Заполнить Bridge VLAN Table

Откройте:

> **Bridge → VLANs → Add New**

Для VLAN 10:

| Поле     | Значение         |
| -------- | ---------------- |
| Bridge   | `br-vlan`        |
| VLAN IDs | `10`             |
| Tagged   | пусто            |
| Untagged | `ether2, ether3` |

Для VLAN 20:

| Поле     | Значение         |
| -------- | ---------------- |
| Bridge   | `br-vlan`        |
| VLAN IDs | `20`             |
| Tagged   | пусто            |
| Untagged | `ether4, ether5` |

Не объединяйте разные VLAN в одну строку, если у них различаются untagged-порты.

### 2.7. Включить VLAN Filtering

Ещё раз проверьте:

> **Bridge → Ports**
> **Bridge → VLANs**

Затем откройте:

> **Bridge → Bridge → br-vlan**

Установите:

```text
VLAN Filtering: yes
```

### 2.8. Настроить компьютеры

Назначьте адреса из таблицы через настройки IPv4 ОС.

### 2.9. Минимальная проверка

Должно работать:

```text
PC-S1 → PC-S2
PC-A1 → PC-A2
```

Не должно работать:

```text
PC-S1 → PC-A1
PC-S2 → PC-A2
```

До включения маршрутизации разные VLAN изолированы на L2.

---

<a id="two-mikrotik"></a>

## 3. VLAN между двумя MikroTik

### 3.1. Задача

Передать VLAN 10 и VLAN 20 через один trunk.

```text
PC-S1 ─ ether2 SW1 ether5 ═════ ether5 SW2 ether2 ─ PC-S2
PC-A1 ─ ether3 SW1                   SW2 ether3 ─ PC-A2
```

На обоих MikroTik:

| Порт     | Назначение         |
| -------- | ------------------ |
| `ether2` | access VLAN 10     |
| `ether3` | access VLAN 20     |
| `ether5` | trunk VLAN 10 и 20 |

### 3.2. Создать bridge на каждом MikroTik

На SW1 и SW2:

> **Bridge → Bridge → Add New**

```text
Name: br-vlan
Protocol Mode: rstp
VLAN Filtering: no
```

### 3.3. Настроить access-порты

На каждом устройстве:

#### `ether2`

```text
Bridge: br-vlan
PVID: 10
Ingress Filtering: yes
Frame Types: admit only untagged and priority tagged
Edge: yes
```

#### `ether3`

```text
Bridge: br-vlan
PVID: 20
Ingress Filtering: yes
Frame Types: admit only untagged and priority tagged
Edge: yes
```

### 3.4. Настроить trunk

На каждом устройстве:

> **Bridge → Ports → Add New**

Для `ether5`:

| Поле              | Значение                 |
| ----------------- | ------------------------ |
| Interface         | `ether5`                 |
| Bridge            | `br-vlan`                |
| Ingress Filtering | `yes`                    |
| Frame Types       | `admit only VLAN tagged` |
| Edge              | `no`                     |
| Point To Point    | `yes` или `auto`         |

На trunk не требуется назначать пользовательский PVID, если принимаются только tagged-кадры.

### 3.5. Настроить VLAN Table на SW1

VLAN 10:

```text
Tagged: ether5
Untagged: ether2
```

VLAN 20:

```text
Tagged: ether5
Untagged: ether3
```

### 3.6. Настроить VLAN Table на SW2

Укажите те же VLAN:

VLAN 10:

```text
Tagged: ether5
Untagged: ether2
```

VLAN 20:

```text
Tagged: ether5
Untagged: ether3
```

### 3.7. Включить VLAN Filtering

На обоих устройствах:

> **Bridge → Bridge → br-vlan**

```text
VLAN Filtering: yes
```

### 3.8. Проверка

Должно работать:

```text
PC-S1 на SW1 → PC-S2 на SW2
PC-A1 на SW1 → PC-A2 на SW2
```

Не должно работать:

```text
VLAN 10 → VLAN 20
```

### 3.9. VLAN через три коммутатора

```text
SW1 ─── SW2 ─── SW3
```

Если SW2 является промежуточным, оба его межкоммутаторных порта должны быть tagged для VLAN 10 и 20.

Пример SW2:

| VLAN | Tagged           |
| ---: | ---------------- |
|   10 | `ether5, ether6` |
|   20 | `ether5, ether6` |

На промежуточном trunk-коммутаторе access-порты для этих VLAN не обязательны.

---

<a id="vlan-lacp"></a>

## 4. VLAN через LACP

Используйте этот вариант, когда между двумя коммутаторами имеется несколько физических кабелей, объединённых в LACP.

```text
SW1 ether6 ═════ ether6 SW2
SW1 ether7 ═════ ether7 SW2
        \         /
          bond-lacp
```

Создание LACP описано в:

[`03-link-aggregation`](../03-link-aggregation/)

### 4.1. Основное правило

В bridge добавляется:

```text
bond-lacp
```

Не добавляйте в bridge отдельно:

```text
ether6
ether7
```

### 4.2. Добавить bond в bridge

На каждом MikroTik:

> **Bridge → Ports → Add New**

| Поле              | Значение                 |
| ----------------- | ------------------------ |
| Interface         | `bond-lacp`              |
| Bridge            | `br-vlan`                |
| Ingress Filtering | `yes`                    |
| Frame Types       | `admit only VLAN tagged` |
| Edge              | `no`                     |
| Point To Point    | `yes`                    |

### 4.3. Использовать bond как trunk

В:

> **Bridge → VLANs**

Для VLAN 10:

```text
Tagged: bond-lacp
```

Для VLAN 20:

```text
Tagged: bond-lacp
```

Если у VLAN есть локальный access-порт:

```text
VLAN 10:
Tagged: bond-lacp
Untagged: ether2
```

### 4.4. Проверка

После отключения одного кабеля LACP связь VLAN должна сохраниться через второй участник агрегата.

---

<a id="inter-vlan"></a>

## 5. Межвлановая маршрутизация на MikroTik

### 5.1. Задача

Разрешить обмен между:

```text
VLAN 10 — 192.168.10.0/24
VLAN 20 — 192.168.20.0/24
```

MikroTik будет шлюзом:

```text
VLAN 10 → 192.168.10.1
VLAN 20 → 192.168.20.1
```

Сначала полностью настройте L2-часть VLAN.

### 5.2. Создать VLAN-интерфейс VLAN 10

Откройте:

> **Interfaces → VLAN → Add New**

Укажите:

| Поле      | Значение          |
| --------- | ----------------- |
| Name      | `vlan10-services` |
| VLAN ID   | `10`              |
| Interface | `br-vlan`         |

### 5.3. Создать VLAN-интерфейс VLAN 20

```text
Name: vlan20-staff
VLAN ID: 20
Interface: br-vlan
```

VLAN-интерфейс создаётся поверх bridge, а не поверх отдельного slave-порта bridge.

### 5.4. Проверить доступ CPU к VLAN

Откройте:

> **Bridge → VLANs**

В актуальном RouterOS bridge обычно появляется в поле **Current Tagged** автоматически после создания VLAN-интерфейса.

Для VLAN 10 должно быть логически:

```text
Tagged: br-vlan и trunk-порты
Untagged: access-порты VLAN 10
```

Для VLAN 20:

```text
Tagged: br-vlan и trunk-порты
Untagged: access-порты VLAN 20
```

Если `br-vlan` не появился в **Current Tagged**, вручную добавьте его в поле **Tagged** соответствующей VLAN.

### 5.5. Назначить адреса шлюзов

Откройте:

> **IP → Addresses → Add New**

VLAN 10:

```text
Address: 192.168.10.1/24
Interface: vlan10-services
```

VLAN 20:

```text
Address: 192.168.20.1/24
Interface: vlan20-staff
```

Connected routes появятся автоматически.

### 5.6. Настроить клиентов

Для клиента VLAN 10:

```text
IP: 192.168.10.10/24
Gateway: 192.168.10.1
```

Для клиента VLAN 20:

```text
IP: 192.168.20.10/24
Gateway: 192.168.20.1
```

### 5.7. Разрешить маршрутизацию

RouterOS маршрутизирует между непосредственно подключёнными сетями автоматически.

Если связь блокируется, откройте:

> **IP → Firewall → Filter Rules**

Проверьте, нет ли правила `drop` в цепочке `forward`, запрещающего обмен между внутренними VLAN.

### 5.8. Заблокировать Guest

Пример: запретить Guest доступ к Staff.

Откройте:

> **IP → Firewall → Filter Rules → Add New**

На вкладке **General**:

```text
Chain: forward
Src. Address: 192.168.30.0/24
Dst. Address: 192.168.20.0/24
```

На вкладке **Action**:

```text
Action: drop
```

Аналогично создайте запрет Guest → Services:

```text
Src. Address: 192.168.30.0/24
Dst. Address: 192.168.10.0/24
Action: drop
```

Разместите правила выше более общих разрешающих правил.

### 5.9. Заблокировать Quarantine

Для карантинной сети можно запретить доступ ко всему внутреннему диапазону.

Пример:

```text
Chain: forward
Src. Address: 192.168.99.0/24
Dst. Address: 192.168.0.0/16
Action: drop
```

Разрешения DHCP, DNS или доступа к порталу создаются отдельными правилами до общего `drop`.

---

<a id="router-on-stick"></a>

## 6. Router-on-a-stick

Router-on-a-stick используется, когда:

* отдельный коммутатор выполняет L2;
* отдельный маршрутизатор выполняет L3;
* между ними проходит один trunk.

### 6.1. Схема

```text
PC-10 ─ ether2 SW
PC-20 ─ ether3 SW
             ether5 ═════ ether1 Router
                    trunk
```

### 6.2. Настроить коммутатор

На SW:

| Порт     | Настройка          |
| -------- | ------------------ |
| `ether2` | access VLAN 10     |
| `ether3` | access VLAN 20     |
| `ether5` | trunk VLAN 10 и 20 |

Bridge VLAN Table:

```text
VLAN 10:
Tagged: ether5
Untagged: ether2

VLAN 20:
Tagged: ether5
Untagged: ether3
```

### 6.3. Подготовить физический порт маршрутизатора

На отдельном MikroTik Router:

* `ether1` не должен входить в обычный bridge;
* на `ether1` не должно быть нетегированного IP;
* кабель от trunk-порта SW подключается в `ether1`.

### 6.4. Создать VLAN-подинтерфейсы на маршрутизаторе

Откройте:

> **Interfaces → VLAN → Add New**

VLAN 10:

```text
Name: vlan10
VLAN ID: 10
Interface: ether1
```

VLAN 20:

```text
Name: vlan20
VLAN ID: 20
Interface: ether1
```

Здесь VLAN-интерфейсы создаются поверх физического trunk-порта, потому что `ether1` не является slave-портом bridge.

### 6.5. Назначить шлюзы

```text
192.168.10.1/24 → vlan10
192.168.20.1/24 → vlan20
```

### 6.6. Настроить клиентов

```text
VLAN 10 gateway: 192.168.10.1
VLAN 20 gateway: 192.168.20.1
```

После этого маршрутизатор сможет передавать пакеты между VLAN, если firewall не блокирует forwarding.

---

<a id="rapier"></a>

## 7. VLAN на Rapier

На Rapier VLAN настраиваются через AlliedWare CLI.

В примерах:

```text
порт 2 — access VLAN 10
порт 3 — access VLAN 20
порт 5 — tagged trunk
```

Не изменяйте зарезервированные порты управления без разрешения преподавателя.

### 7.1. Создать VLAN

```text
create vlan=SERVICES vid=10
create vlan=STAFF vid=20
```

### 7.2. Удалить access-порты из default VLAN

Порт не может быть untagged одновременно в двух VLAN.

```text
delete vlan=default port=2
delete vlan=default port=3
```

### 7.3. Добавить access-порты

```text
add vlan=SERVICES port=2 frame=untagged
add vlan=STAFF port=3 frame=untagged
```

В лабораторной конфигурации untagged-членство используется как access-режим порта.

### 7.4. Добавить trunk

```text
add vlan=SERVICES port=5 frame=tagged
add vlan=STAFF port=5 frame=tagged
```

На втором Rapier выполните аналогичную настройку.

### 7.5. Посмотреть VLAN

```text
show vlan
```

Конкретная VLAN:

```text
show vlan=SERVICES
show vlan=STAFF
```

### 7.6. VLAN через два отдельных ISL

Старый лабораторный вариант допускает отдельный untagged-кабель на каждую VLAN.

Пример:

```text
порт 5 — untagged VLAN 10
порт 6 — untagged VLAN 20
```

Команды:

```text
delete vlan=default port=5,6
add vlan=SERVICES port=5 frame=untagged
add vlan=STAFF port=6 frame=untagged
```

Современный и более экономный вариант — один tagged trunk для нескольких VLAN.

### 7.7. Межвлановая маршрутизация на L3 Rapier

Если модель Rapier работает как L3-коммутатор:

```text
enable ip
```

Создайте IP-интерфейс для VLAN 10:

```text
add ip interface=vlan10 ipaddress=192.168.10.1 mask=255.255.255.0
```

Для VLAN 20:

```text
add ip interface=vlan20 ipaddress=192.168.20.1 mask=255.255.255.0
```

На клиентах задайте соответствующие шлюзы.

Перед вводом команды проверьте имя VLAN-интерфейса через:

```text
show vlan
```

На конкретной версии AlliedWare используйте `?` для проверки синтаксиса:

```text
add ip interface=?
```

### 7.8. Rapier как L2-коммутатор с MikroTik-маршрутизатором

Более универсальная схема:

```text
Rapier trunk port 5
        ║
MikroTik ether1
```

На Rapier порт 5 добавляется tagged в обе VLAN, а на MikroTik выполняется настройка из раздела [Router-on-a-stick](#router-on-stick).

### 7.9. Удаление VLAN на Rapier

Сначала удалите все порты из VLAN:

```text
delete vlan=SERVICES port=all
delete vlan=STAFF port=all
```

Затем:

```text
destroy vlan=SERVICES
destroy vlan=STAFF
```

Верните клиентские порты в default VLAN:

```text
add vlan=default port=2,3 frame=untagged
```

---

<a id="gvrp-mvrp"></a>

## 8. GVRP на Rapier и MVRP на MikroTik

Используйте динамическое распространение VLAN только при прямом требовании задания.

Для небольшой экзаменационной топологии статическая настройка trunk проще.

### 8.1. Лабораторная схема GVRP

```text
PC-A ─ SW1 ─ SW2 ─ SW3 ─ PC-B
       VLAN10       VLAN10
```

VLAN 10 статически существует на SW1 и SW3.

SW2 должен динамически зарегистрировать её на межкоммутаторных портах.

### 8.2. Подготовить крайние Rapier

На SW1:

```text
create vlan=TEST vid=10
delete vlan=default port=2
add vlan=TEST port=2 frame=untagged
```

На SW3:

```text
create vlan=TEST vid=10
delete vlan=default port=2
add vlan=TEST port=2 frame=untagged
```

Межкоммутаторные порты не добавляйте статически в VLAN TEST либо удалите ранее созданное членство.

### 8.3. Включить GVRP на всех Rapier

На SW1, SW2 и SW3:

```text
enable garp=gvrp
```

### 8.4. Разрешить участие trunk-портов

Пример:

```text
set garp=gvrp port=5 mode=normal
set garp=gvrp port=6 mode=normal
```

Можно передать GVRP управление несколькими портами:

```text
set garp=gvrp port=1-22 mode=normal
```

Не включайте лабораторные зарезервированные management-порты без разрешения.

### 8.5. Посмотреть состояние

```text
show garp=gvrp
show garp=gvrp database
show garp=gvrp machine
show vlan
```

После регистрации промежуточные порты должны появиться как динамические участники VLAN.

### 8.6. Отключить GVRP

```text
set garp=gvrp port=1-22 mode=none
disable garp=gvrp
```

При необходимости очистить состояние:

```text
purge garp=gvrp
```

После отключения верните статическое tagged-членство trunk-портов.

---

## 8.7. MVRP на MikroTik

### Схема

```text
PC-A ─ SW1 ─ SW2 ─ SW3 ─ PC-B
       VLAN10       VLAN10
```

Access-порты VLAN 10 на SW1 и SW3 настраиваются статически.

Trunk-порты между устройствами должны входить в один VLAN-aware bridge.

### Шаг 1. Включить VLAN Filtering

На каждом устройстве сначала настройте bridge и порты, затем включите:

> **Bridge → Bridge → br-vlan**

```text
VLAN Filtering: yes
```

### Шаг 2. Включить MVRP

В том же окне:

```text
MVRP: yes
```

### Шаг 3. Настроить trunk-порты

Откройте:

> **Bridge → Ports → `<TRUNK_PORT>`**

Установите:

```text
MVRP Applicant State: normal participant
MVRP Registrar State: normal
Edge: no
Point To Point: yes
```

### Шаг 4. Настроить крайние access-порты

На SW1 и SW3 VLAN 10 создаётся статически:

```text
PVID: 10
Ingress Filtering: yes
Frame Types: admit only untagged and priority tagged
```

В Bridge VLAN Table access-порт указывается как `Untagged`.

Статическое или созданное через PVID членство будет объявляться MVRP соседним устройствам.

### Шаг 5. Не создавать VLAN статически на промежуточных trunk

На SW2 не добавляйте VLAN 10 вручную в Bridge VLAN Table для trunk-портов.

После получения MVRP-регистрации они должны динамически стать tagged-членами VLAN 10.

### Шаг 6. Проверка

Откройте состояние trunk-порта и найдите:

```text
Declared VLAN IDs
Registered VLAN IDs
```

Также можно открыть таблицу MVRP, если она отображается в используемой версии WebFig.

Для зарегистрированной VLAN состояние должно соответствовать `IN`.

### Шаг 7. Отключение MVRP

На bridge:

```text
MVRP: no
```

После отключения создайте необходимые VLAN на trunk статически.

---

<a id="radius-vlan"></a>

## 9. Динамическая VLAN через RADIUS

Полная настройка RADIUS, IAS/NPS, 802.1X и MAC-аутентификации находится в:

[`05-radius-access-control`](../05-radius-access-control/)

Здесь выполняется только подготовка VLAN на MikroTik.

### 9.1. Задача

После авторизации клиент должен получить:

```text
Access-Accept → VLAN 20
Access-Reject → VLAN 999
RADIUS недоступен → VLAN 998
```

### 9.2. Подготовить VLAN на trunk

Предположим:

```text
ether1 — trunk к маршрутизатору или следующему коммутатору
ether2 — порт клиента
```

Создайте bridge с VLAN Filtering.

Добавьте `ether1` как trunk:

```text
Ingress Filtering: yes
Frame Types: admit only VLAN tagged
Edge: no
```

Добавьте `ether2` как клиентский порт:

```text
Ingress Filtering: yes
Frame Types: admit only untagged and priority tagged
Edge: yes
```

Не назначайте `ether2` статически в рабочую VLAN.

### 9.3. Создать записи VLAN

В:

> **Bridge → VLANs**

VLAN 20:

```text
Tagged: ether1
Untagged: пусто
```

VLAN 999:

```text
Tagged: ether1
Untagged: пусто
```

VLAN 998:

```text
Tagged: ether1
Untagged: пусто
```

Клиентский порт будет добавлен в **Current Untagged** динамически после результата аутентификации.

### 9.4. Настроить VLAN-интерфейсы шлюза

Если этот же MikroTik является шлюзом:

```text
vlan20 → 192.168.20.1/24
vlan999 → 192.168.99.1/24
vlan998 → 192.168.98.1/24
```

Если шлюз расположен на другом устройстве, VLAN нужно только передать по trunk.

### 9.5. Атрибуты RADIUS

Сервер должен вернуть для рабочей VLAN:

```text
Tunnel-Type = VLAN
Tunnel-Medium-Type = IEEE-802
Tunnel-Private-Group-ID = 20
```

Коммутатор получает VLAN ID и динамически назначает его порту.

### 9.6. Проверка после авторизации

Откройте:

> **Dot1X → Active**

Проверьте VLAN активной сессии.

Затем:

> **Bridge → VLANs**

Клиентский порт должен появиться в **Current Untagged** у назначенной VLAN.

Не добавляйте этот порт статически в ту же VLAN: его членством управляет RADIUS.

---

<a id="cleanup"></a>

## 10. Очистка конфигурации

### 10.1. MikroTik

Удаляйте настройки в таком порядке:

1. отключите RADIUS/802.1X на клиентских портах;
2. отключите MVRP;
3. удалите firewall-правила между VLAN;
4. удалите IP-адреса шлюзов;
5. удалите VLAN-интерфейсы;
6. удалите записи **Bridge → VLANs**;
7. верните PVID портов к исходным значениям;
8. удалите access- и trunk-порты из учебного bridge;
9. удалите bond из bridge, если использовался;
10. удалите учебный bridge.

Не удаляйте management bridge, management VLAN и порт управления.

### 10.2. Rapier

Отключите GVRP:

```text
disable garp=gvrp
```

Удалите tagged и untagged-членство:

```text
delete vlan=SERVICES port=all
delete vlan=STAFF port=all
```

Удалите VLAN:

```text
destroy vlan=SERVICES
destroy vlan=STAFF
```

Верните обычные порты в default VLAN:

```text
add vlan=default port=2,3,5 frame=untagged
```

Не добавляйте в default VLAN trunk или management-порт, если исходная конфигурация была другой.

---

<a id="mapping"></a>

## 11. Краткое соответствие параметров

| Задача                    | MikroTik                      | Rapier                         |
| ------------------------- | ----------------------------- | ------------------------------ |
| Создать VLAN              | `Bridge → VLANs`              | `create vlan=... vid=...`      |
| Access-порт               | PVID + Untagged               | `frame=untagged`               |
| Trunk-порт                | Tagged                        | `frame=tagged`                 |
| Запрет tagged на клиенте  | Frame Types                   | untagged membership            |
| Проверка входной VLAN     | Ingress Filtering             | зависит от конфигурации порта  |
| VLAN между коммутаторами  | tagged trunk                  | tagged ISL                     |
| Шлюз VLAN                 | VLAN-интерфейс + IP           | IP interface VLAN              |
| Router-on-a-stick         | VLAN поверх физического trunk | Rapier как L2 + внешний router |
| Динамическая регистрация  | MVRP                          | GVRP                           |
| Динамическая VLAN клиента | Dot1X/RADIUS                  | Port Authentication/RADIUS     |
| Таблица VLAN              | `Bridge → VLANs`              | `show vlan`                    |

---

## Минимальный алгоритм выполнения

```text
1. Составить таблицу VLAN и портов.
2. Сохранить отдельный management-доступ.
3. Создать один bridge с VLAN Filtering = no.
4. Добавить access-порты и задать PVID.
5. Добавить trunk-порты.
6. Заполнить Bridge VLAN Table.
7. Включить VLAN Filtering.
8. При необходимости создать VLAN-интерфейсы и шлюзы.
9. Назначить клиентам адреса и gateway.
10. Для динамической VLAN не добавлять клиентский порт статически.
```
