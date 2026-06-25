# LACP и Bonding

Основная настройка выполняется на MikroTik через WebFig. Для Rapier, Linux Server и Windows Server приведены отдельные законченные ветки.

Перед началом выполните подготовку из [`00-basic-setup`](../00-basic-setup/).

Обозначения:

* `<BOND>` — логический bonding-интерфейс;
* `<PORT_1>`, `<PORT_2>` — физические участники агрегата;
* `<BRIDGE>` — bridge, в который добавляется bond;
* `<ACCESS_VLAN>` — VLAN сервера или пользователя;
* `<TRUNK_VLANS>` — список VLAN на trunk;
* `<IP>` — IP-адрес логического интерфейса;
* `<PREFIX>` — длина сетевого префикса.

В основных примерах используются:

```text
MikroTik-1: ether6, ether7
MikroTik-2: ether6, ether7
Bond:       bond-lacp
Bridge:     br-vlan
```

---

## Содержание

1. [Выбор режима агрегации](#mode-selection)
2. [Подготовка физических линий](#preparation)
3. [LACP между двумя MikroTik](#mikrotik-lacp)
4. [Использование LACP как access-соединения](#lacp-access)
5. [Использование LACP как VLAN trunk](#lacp-trunk)
6. [LACP между двумя Rapier](#rapier-lacp)
7. [Статическая агрегация](#static-aggregation)
8. [Linux Server и MikroTik](#linux-mikrotik)
9. [Windows Server и MikroTik](#windows-mikrotik)
10. [Active-backup](#active-backup)
11. [Удаление конфигурации](#cleanup)
12. [Краткое соответствие параметров](#mapping)

---

<a id="mode-selection"></a>

## 1. Выбор режима агрегации

Перед настройкой определите, что требует задание.

| Требование                                           | Режим                                                        |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| Объединить линии по стандарту LACP                   | `802.3ad`                                                    |
| Использовать несколько активных линий                | `802.3ad`                                                    |
| Автоматически обнаруживать участников                | `802.3ad`                                                    |
| Создать агрегат без LACP                             | `balance-xor` или статический trunk                          |
| Основной кабель и резервный кабель                   | `active-backup`                                              |
| После восстановления вернуться на основной           | `active-backup` с Primary                                    |
| Подключить сервер двумя линиями к одному коммутатору | обычно `802.3ad`                                             |
| Подключить сервер к двум независимым коммутаторам    | требуется MLAG либо `active-backup` при подходящей топологии |

Обычный LACP-агрегат должен начинаться и заканчиваться на одной паре логических устройств:

```text
MikroTik-1 ether6 ─── ether6 MikroTik-2
MikroTik-1 ether7 ─── ether7 MikroTik-2
```

Нельзя подключить один участник обычного LACP к SW1, а второй к независимому SW2, если эти устройства не образуют MLAG.

### LACP и скорость одного потока

LACP распределяет разные потоки по физическим линиям на основе hash.

Один поток:

```text
один клиент → один сервер
```

обычно использует один физический участник.

Несколько независимых потоков могут распределяться между разными участниками. Поэтому два канала по 1 Гбит/с дают увеличенную суммарную пропускную способность, но не гарантируют 2 Гбит/с для одного TCP-соединения.

---

<a id="preparation"></a>

## 2. Подготовка физических линий

### 2.1. Нарисовать подключение

Пример:

```text
            LACP
MikroTik-1 ether6 ═════════ ether6 MikroTik-2
MikroTik-1 ether7 ═════════ ether7 MikroTik-2
```

Составьте таблицу:

| Сторона A | Сторона B | Назначение |
| --------- | --------- | ---------- |
| `ether6`  | `ether6`  | участник 1 |
| `ether7`  | `ether7`  | участник 2 |

### 2.2. Требования к участникам

Все физические порты агрегата должны:

* соединять одну и ту же пару устройств;
* иметь одинаковую скорость;
* работать в full-duplex;
* иметь одинаковый MTU;
* использовать одинаковую VLAN-конфигурацию;
* не иметь отдельных IP-адресов;
* не входить отдельно в bridge;
* не входить в другой bond.

Для LACP на обеих сторонах должен совпадать логический состав агрегата.

### 2.3. Убрать порты из bridge MikroTik

На каждом MikroTik откройте:

> **Bridge → Ports**

Если `ether6` и `ether7` уже находятся в bridge:

1. выберите запись `ether6`;
2. нажмите **Remove**;
3. повторите для `ether7`.

После создания bond в bridge будет добавлен сам `bond-lacp`.

### 2.4. Удалить IP с физических портов

Откройте:

> **IP → Addresses**

Удалите адреса, назначенные непосредственно:

```text
ether6
ether7
```

IP в дальнейшем назначается:

* самому bond, если это L3-соединение;
* VLAN-интерфейсу поверх bond;
* bridge или VLAN-интерфейсу bridge.

### 2.5. Проверить Ethernet-параметры

Откройте:

> **Interfaces → Ethernet**

Для обоих портов рекомендуется:

```text
Auto Negotiation: yes
```

Проверьте одинаковые:

```text
Rate
Full Duplex
MTU
```

Не объединяйте порт 100 Мбит/с и порт 1 Гбит/с в один LACP-агрегат.

---

<a id="mikrotik-lacp"></a>

## 3. LACP между двумя MikroTik

### 3.1. Схема

```text
PC-A ─── MikroTik-1
              ║ ether6
              ║ ether7
          bond-lacp
              ║ ether6
              ║ ether7
PC-B ─── MikroTik-2
```

Сначала настройте bond на обоих MikroTik, затем подключайте его к bridge или назначайте IP.

### 3.2. Создать bond на MikroTik-1

Откройте:

> **Interfaces → Bonding → Add New**

Укажите:

| Поле                 | Значение         |
| -------------------- | ---------------- |
| Name                 | `bond-lacp`      |
| Slaves               | `ether6, ether7` |
| Mode                 | `802.3ad`        |
| LACP Mode            | `active`         |
| LACP Rate            | `1sec`           |
| Link Monitoring      | `mii`            |
| MII Interval         | `100ms`          |
| Transmit Hash Policy | `layer-2-and-3`  |
| Min Links            | `1`              |

Нажмите **Apply**, затем **OK**.

### 3.3. Создать bond на MikroTik-2

Создайте такой же интерфейс:

```text
Name: bond-lacp
Slaves: ether6, ether7
Mode: 802.3ad
LACP Mode: active
LACP Rate: 1sec
Link Monitoring: mii
MII Interval: 100ms
Transmit Hash Policy: layer-2-and-3
Min Links: 1
```

### 3.4. Active и passive

Допустимые сочетания:

| Сторона A | Сторона B | Результат              |
| --------- | --------- | ---------------------- |
| active    | active    | работает               |
| active    | passive   | работает               |
| passive   | active    | работает               |
| passive   | passive   | агрегат не формируется |

Для экзамена проще использовать:

```text
active ↔ active
```

либо:

```text
active ↔ passive
```

### 3.5. Fast и slow

В RouterOS:

```text
LACP Rate: 1sec  — быстрый режим
LACP Rate: 30secs — медленный режим
```

Для учебной отказоустойчивой схемы используйте `1sec` на обеих сторонах.

### 3.6. Transmit Hash Policy

Рекомендуемый универсальный вариант:

```text
layer-2-and-3
```

Он учитывает MAC- и IP-адреса.

Варианты:

| Политика        | Используемые поля          |
| --------------- | -------------------------- |
| `layer-2`       | MAC источника и назначения |
| `layer-2-and-3` | MAC и IP                   |
| `layer-3-and-4` | IP и транспортные порты    |

Для обычного LACP используйте `layer-2-and-3`, если условие не требует другого.

Не используйте `balance-rr` вместо LACP: он может распределять пакеты одного потока по разным линиям и нарушать порядок.

### 3.7. Проверить состояние bond

Откройте:

> **Interfaces → Bonding**

У интерфейса `bond-lacp` должен появиться признак **Running**.

Откройте состояние или мониторинг bond.

Проверьте:

```text
Mode: 802.3ad
Active Ports: ether6, ether7
Inactive Ports: пусто
Partner System ID: один и тот же для обоих портов
```

Если активен только один участник, проверьте:

* кабель;
* скорость и duplex;
* LACP на второй стороне;
* одинаковый состав агрегата;
* не входит ли физический порт отдельно в bridge.

---

<a id="lacp-access"></a>

## 4. Использование LACP как access-соединения

Этот вариант применяется, когда через агрегат должна проходить одна обычная VLAN без тегов.

Пример:

```text
Linux Server
  NIC1 ═════ ether6 MikroTik
  NIC2 ═════ ether7 MikroTik

Серверная VLAN: 10
```

### 4.1. Добавить bond в bridge

Откройте:

> **Bridge → Ports → Add New**

Укажите:

| Поле              | Значение                                  |
| ----------------- | ----------------------------------------- |
| Interface         | `bond-lacp`                               |
| Bridge            | `br-vlan`                                 |
| PVID              | `10`                                      |
| Ingress Filtering | `yes`                                     |
| Frame Types       | `admit only untagged and priority tagged` |
| Edge              | `yes`                                     |

Для соединения switch-to-switch значение `Edge` должно быть `no`.

Для конечного сервера допустимо:

```text
Edge: yes
```

### 4.2. Добавить bond в VLAN Table

Откройте:

> **Bridge → VLANs**

В запись VLAN 10 добавьте:

```text
Untagged: bond-lacp
```

Пример:

```text
VLAN IDs: 10
Tagged: trunk-порты и br-vlan при необходимости
Untagged: bond-lacp
```

### 4.3. Адрес сервера

Сервер назначает IP логическому bond:

```text
192.168.10.10/24
Gateway: 192.168.10.1
```

Физические NIC не получают собственные IP.

---

<a id="lacp-trunk"></a>

## 5. Использование LACP как VLAN trunk

Этот вариант применяется:

* между двумя коммутаторами;
* между коммутатором и сервером виртуализации;
* когда через агрегат проходит несколько VLAN.

### 5.1. Добавить bond как trunk

Откройте:

> **Bridge → Ports → Add New**

Укажите:

| Поле              | Значение                 |
| ----------------- | ------------------------ |
| Interface         | `bond-lacp`              |
| Bridge            | `br-vlan`                |
| Ingress Filtering | `yes`                    |
| Frame Types       | `admit only VLAN tagged` |
| Edge              | `no`                     |
| Point To Point    | `yes`                    |

### 5.2. Добавить bond в VLAN Table

Для VLAN 10:

```text
Tagged: bond-lacp
```

Для VLAN 20:

```text
Tagged: bond-lacp
```

Для VLAN 30:

```text
Tagged: bond-lacp
```

Пример:

| VLAN ID | Tagged      | Untagged |
| ------: | ----------- | -------- |
|      10 | `bond-lacp` | `ether2` |
|      20 | `bond-lacp` | `ether3` |
|      30 | `bond-lacp` | `ether4` |

На обеих сторонах агрегата должен быть разрешён одинаковый набор VLAN.

### 5.3. VLAN поверх L3 bond

Если bond не находится в bridge, VLAN-интерфейсы можно создать непосредственно поверх него:

> **Interfaces → VLAN → Add New**

Пример:

```text
Name: vlan10
VLAN ID: 10
Interface: bond-lacp
```

Такой вариант подходит для маршрутизатора или сервера, а не для обычного L2-trunk внутри VLAN-aware bridge.

### 5.4. Проверка отказа линии

1. Запустите постоянный обмен между хостами по разные стороны агрегата.
2. Отключите кабель `ether6`.
3. Связь должна продолжиться через `ether7`.
4. Подключите кабель обратно.
5. Проверьте, что оба участника снова активны.

В момент переключения возможна кратковременная потеря нескольких пакетов.

---

<a id="rapier-lacp"></a>

## 6. LACP между двумя Rapier

На Rapier настройка выполняется через AlliedWare CLI.

В примере используются порты 3 и 4.

Перед началом убедитесь, что преподаватель разрешил изменять эти порты. Порты 23–26 в лабораторной конфигурации изменять запрещено без разрешения.

### 6.1. Схема

```text
Rapier-A port3 ═════════ port3 Rapier-B
Rapier-A port4 ═════════ port4 Rapier-B
```

### 6.2. Временно отключить участники

На обоих коммутаторах:

```text
disable switch port=3,4
```

Это предотвращает временную петлю до формирования агрегата.

### 6.3. Проверить VLAN и скорость

Порты 3 и 4 должны иметь:

* одинаковую скорость;
* full-duplex;
* одинаковое членство VLAN;
* одинаковый режим tagged/untagged.

Просмотр:

```text
show switch port=3,4
show vlan
```

### 6.4. Включить LACP

На обоих Rapier:

```text
enable lacp
```

### 6.5. Задать System Priority

На Rapier-A:

```text
set lacp priority=100
```

На Rapier-B:

```text
set lacp priority=200
```

Меньшее числовое значение имеет более высокий приоритет.

Для простой схемы можно оставить одинаковые значения по умолчанию.

### 6.6. Передать порты под управление LACP

На Rapier-A:

```text
add lacp port=3,4 adminkey=10 priority=128 mode=active periodic=fast
```

На Rapier-B:

```text
add lacp port=3,4 adminkey=10 priority=128 mode=passive periodic=fast
```

`adminkey` должен совпадать для портов одного агрегата.

Можно использовать `active` на обеих сторонах:

```text
mode=active
```

### 6.7. Включить порты

На обоих устройствах:

```text
enable switch port=3,4
```

### 6.8. Проверить LACP

Общая информация:

```text
show lacp
```

Порты:

```text
show lacp port=all
```

Счётчики:

```text
show lacp port=all counter
```

Сформированные агрегаты:

```text
show lacp trunk
```

Оба порта должны входить в один логический trunk.

### 6.9. Fast и slow

Быстрый режим:

```text
set lacp port=3,4 periodic=fast
```

Если эта форма команды не поддерживается версией AlliedWare, удалите порты и добавьте их повторно с:

```text
periodic=fast
```

Медленный режим:

```text
periodic=slow
```

В лабораторной fast соответствует периодичности около одной секунды, slow — около 30 секунд.

### 6.10. Проверить отказ линии

1. Организуйте обмен через агрегат.
2. Отключите один кабель.
3. Проверьте сохранение связи.
4. Подключите кабель обратно.
5. Выполните:

```text
show lacp port=all
show lacp trunk
```

### 6.11. Удалить LACP

Сначала отключите физические порты:

```text
disable switch port=3,4
```

Выведите их из LACP:

```text
delete lacp port=3,4
```

Если других LACP-портов нет:

```text
disable lacp
```

После этого порты можно использовать отдельно или добавить в статический агрегат.

---

<a id="static-aggregation"></a>

## 7. Статическая агрегация

Статический агрегат создаётся вручную и не использует LACPDU.

Он допустим, когда:

* обе стороны явно настроены одинаково;
* LACP не поддерживается;
* условие прямо требует статический trunk;
* топология фиксирована.

Недостаток: статическая агрегация хуже обнаруживает ошибки коммутации и логические обрывы.

---

## 7.1. Статическая агрегация MikroTik ↔ MikroTik

На обоих устройствах откройте:

> **Interfaces → Bonding → Add New**

Укажите:

| Поле                 | Значение         |
| -------------------- | ---------------- |
| Name                 | `bond-static`    |
| Slaves               | `ether6, ether7` |
| Mode                 | `balance-xor`    |
| Link Monitoring      | `mii`            |
| MII Interval         | `100ms`          |
| Transmit Hash Policy | `layer-2-and-3`  |

Не используйте режим `802.3ad`: он включает LACP.

После создания добавьте `bond-static` в bridge как access или trunk.

### 7.2. Статическая агрегация MikroTik ↔ управляемый коммутатор

На MikroTik:

```text
Mode: balance-xor
```

На коммутаторе создайте статический LAG из тех же физических портов.

Обе стороны должны использовать совместимую hash-политику.

---

## 7.3. Статический trunk Rapier ↔ Rapier

Сначала отключите порты:

```text
disable switch port=3,4
```

На Rapier-A:

```text
create switch trunk=TRUNK1 port=3,4 select=macboth
```

На Rapier-B:

```text
create switch trunk=TRUNK1 port=3,4 select=macboth
```

Варианты `select`:

| Значение  | Балансировка      |
| --------- | ----------------- |
| `macsrc`  | по MAC источника  |
| `macdest` | по MAC назначения |
| `macboth` | по обоим MAC      |
| `ipsrc`   | по IP источника   |
| `ipdest`  | по IP назначения  |
| `ipboth`  | по обоим IP       |

Для универсальной схемы:

```text
select=macboth
```

или:

```text
select=ipboth
```

После настройки включите порты:

```text
enable switch port=3,4
```

Просмотр:

```text
show switch trunk
show switch trunk=TRUNK1
```

### 7.4. Добавление и удаление участника Rapier

Добавить порт:

```text
add switch trunk=TRUNK1 port=5
```

Удалить порт:

```text
delete switch trunk=TRUNK1 port=5
```

Удалить все порты:

```text
delete switch trunk=TRUNK1 port=all
```

Удалить агрегат:

```text
destroy switch trunk=TRUNK1
```

Trunk можно уничтожить только после удаления из него портов.

---

<a id="linux-mikrotik"></a>

## 8. Linux Server и MikroTik

В Linux нет единого стандартного графического редактора bonding для всех дистрибутивов. Ниже используется NetworkManager через `nmcli`.

### 8.1. Схема

```text
Linux Server enp2s0 ═════ ether6 MikroTik
Linux Server enp3s0 ═════ ether7 MikroTik
```

На MikroTik сначала создайте `bond-lacp` из `ether6` и `ether7`.

### 8.2. Определить интерфейсы Linux

```bash
ip -br link
```

Предположим, используются:

```text
enp2s0
enp3s0
```

Удалите или отключите старые профили NetworkManager, использующие эти интерфейсы.

Просмотр:

```bash
nmcli connection show
```

### 8.3. Создать Linux bond0 в режиме LACP

```bash
sudo nmcli connection add \
  type bond \
  ifname bond0 \
  con-name bond0 \
  bond.options "mode=802.3ad,miimon=100,lacp_rate=fast,xmit_hash_policy=layer2+3"
```

### 8.4. Добавить первый физический интерфейс

```bash
sudo nmcli connection add \
  type ethernet \
  ifname enp2s0 \
  con-name bond0-port1 \
  master bond0 \
  slave-type bond
```

### 8.5. Добавить второй физический интерфейс

```bash
sudo nmcli connection add \
  type ethernet \
  ifname enp3s0 \
  con-name bond0-port2 \
  master bond0 \
  slave-type bond
```

Не назначайте IP физическим интерфейсам `enp2s0` и `enp3s0`.

---

## 8.6. Bond как обычное access-соединение

Назначьте IP интерфейсу `bond0`:

```bash
sudo nmcli connection modify bond0 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.10/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.1
```

Включите соединение:

```bash
sudo nmcli connection up bond0
```

На MikroTik `bond-lacp` должен быть access-портом соответствующей VLAN.

---

## 8.7. Bond как VLAN trunk

Сам `bond0` оставьте без IPv4:

```bash
sudo nmcli connection modify bond0 ipv4.method disabled
```

Создайте VLAN 10:

```bash
sudo nmcli connection add \
  type vlan \
  con-name bond0.10 \
  ifname bond0.10 \
  dev bond0 \
  id 10
```

Назначьте адрес:

```bash
sudo nmcli connection modify bond0.10 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.10/24 \
  ipv4.gateway 192.168.10.1
```

Создайте VLAN 20:

```bash
sudo nmcli connection add \
  type vlan \
  con-name bond0.20 \
  ifname bond0.20 \
  dev bond0 \
  id 20
```

Назначьте адрес:

```bash
sudo nmcli connection modify bond0.20 \
  ipv4.method manual \
  ipv4.addresses 192.168.20.10/24
```

Включите соединения:

```bash
sudo nmcli connection up bond0
sudo nmcli connection up bond0.10
sudo nmcli connection up bond0.20
```

На MikroTik `bond-lacp` должен быть tagged-портом VLAN 10 и 20.

### 8.8. Минимальная проверка Linux

```bash
cat /proc/net/bonding/bond0
```

Проверьте:

```text
Bonding Mode: IEEE 802.3ad
MII Status: up
Slave Interface: enp2s0
Slave Interface: enp3s0
```

---

<a id="windows-mikrotik"></a>

## 9. Windows Server и MikroTik

### 9.1. Поддерживаемые варианты

Встроенный LBFO NIC Teaming применяется в поддерживающих его версиях Windows Server.

Windows Server 2003 не имеет универсального встроенного NIC Teaming. Для него требуется:

* утилита производителя сетевого адаптера;
* драйвер с поддержкой teaming;
* либо другая серверная ОС.

Если производитель не предоставляет teaming для Windows Server 2003, этот сценарий выполнить штатными средствами нельзя.

### 9.2. Схема

```text
Windows Server Ethernet 2 ═════ ether6 MikroTik
Windows Server Ethernet 3 ═════ ether7 MikroTik
```

На MikroTik создайте `bond-lacp` из `ether6` и `ether7`.

### 9.3. Создать Team через Server Manager

1. Откройте **Server Manager**.
2. Перейдите в **Local Server**.
3. Найдите поле **NIC Teaming**.
4. Нажмите значение **Disabled** или **Enabled**.
5. В окне **NIC Teaming** откройте **Tasks → New Team**.
6. Укажите имя:

```text
Team-LACP
```

7. Выберите два физических адаптера.
8. Откройте **Additional Properties**.
9. Установите:

```text
Teaming Mode: LACP
Load Balancing Mode: Dynamic
Standby Adapter: None
```

10. Создайте team.

### 9.4. Назначить IP логическому Team

Откройте:

> **Сетевые подключения**

Появится новый виртуальный адаптер `Team-LACP`.

Назначьте IP именно ему:

```text
IP: 192.168.10.10
Mask: 255.255.255.0
Gateway: 192.168.10.1
```

Не назначайте IP отдельным физическим участникам.

### 9.5. Настроить LACP Timer

Если графический интерфейс не предоставляет выбор таймера, откройте PowerShell от имени администратора:

```powershell
Set-NetLbfoTeam -Name "Team-LACP" -LacpTimer Fast
```

Просмотр:

```powershell
Get-NetLbfoTeam
Get-NetLbfoTeamMember
```

### 9.6. VLAN на Windows Team

Если сервер должен работать в одной access VLAN, VLAN на Windows не создаётся: untagged membership задаётся на MikroTik.

Если сервер должен принимать несколько tagged VLAN, используйте:

* возможности NIC Teaming;
* Hyper-V virtual switch;
* VLAN-настройки драйвера;
* либо отдельные виртуальные интерфейсы.

Конкретный способ зависит от версии Windows Server и драйвера адаптера.

---

<a id="active-backup"></a>

## 10. Active-backup

Active-backup использует одну активную линию. Вторая включается после отказа основной.

```text
основной:  ether6
резервный: ether7
```

Этот режим не увеличивает суммарную пропускную способность.

---

## 10.1. Active-backup на MikroTik

Откройте:

> **Interfaces → Bonding → Add New**

Укажите:

| Поле             | Значение         |
| ---------------- | ---------------- |
| Name             | `bond-backup`    |
| Slaves           | `ether6, ether7` |
| Mode             | `active-backup`  |
| Primary          | `ether6`         |
| Link Monitoring  | `mii`            |
| MII Interval     | `100ms`          |
| Primary Reselect | `always`         |

`Primary Reselect: always` означает возврат на `ether6` после его восстановления.

### 10.2. Добавить bond в bridge

Для одной access VLAN:

```text
Interface: bond-backup
Bridge: br-vlan
PVID: 10
```

Для trunk:

```text
Interface: bond-backup
Bridge: br-vlan
Frame Types: admit only VLAN tagged
```

### 10.3. Проверить переключение

1. Убедитесь, что активен `ether6`.
2. Запустите обмен через bond.
3. Отключите кабель `ether6`.
4. Активным должен стать `ether7`.
5. Подключите `ether6`.
6. При `Primary Reselect: always` активность должна вернуться на `ether6`.

---

## 10.4. Active-backup на Linux

Создайте bond:

```bash
sudo nmcli connection add \
  type bond \
  ifname bond0 \
  con-name bond0 \
  bond.options "mode=active-backup,miimon=100,primary=enp2s0"
```

Добавьте порты:

```bash
sudo nmcli connection add \
  type ethernet \
  ifname enp2s0 \
  con-name bond0-port1 \
  master bond0 \
  slave-type bond
```

```bash
sudo nmcli connection add \
  type ethernet \
  ifname enp3s0 \
  con-name bond0-port2 \
  master bond0 \
  slave-type bond
```

Назначьте IP на `bond0`.

### 10.5. Active-backup и коммутатор

В режиме active-backup коммутатору обычно не нужно создавать LACP-агрегат.

Оба физических порта должны находиться:

* в одной access VLAN;
* либо иметь одинаковую trunk-конфигурацию.

Не объединяйте их в LACP на стороне коммутатора, если сервер использует `active-backup`.

---

<a id="cleanup"></a>

## 11. Удаление конфигурации

### 11.1. MikroTik

Удаляйте в таком порядке:

1. удалите IP с bond;
2. удалите VLAN-интерфейсы поверх bond;
3. удалите bond из **Bridge → Ports**;
4. удалите bond из **Bridge → VLANs**;
5. откройте **Interfaces → Bonding**;
6. удалите bonding-интерфейс;
7. убедитесь, что физические порты снова доступны отдельно;
8. при необходимости добавьте их обратно в bridge.

Не удаляйте bond до удаления зависимых VLAN-интерфейсов.

### 11.2. Rapier LACP

```text
disable switch port=3,4
delete lacp port=3,4
disable lacp
enable switch port=3,4
```

Не отключайте LACP глобально, если другие агрегаты продолжают его использовать.

### 11.3. Rapier статический trunk

```text
disable switch port=3,4
delete switch trunk=TRUNK1 port=all
destroy switch trunk=TRUNK1
enable switch port=3,4
```

### 11.4. Linux

Отключите соединения:

```bash
sudo nmcli connection down bond0
```

Удалите VLAN:

```bash
sudo nmcli connection delete bond0.10
sudo nmcli connection delete bond0.20
```

Удалите физические участники:

```bash
sudo nmcli connection delete bond0-port1
sudo nmcli connection delete bond0-port2
```

Удалите bond:

```bash
sudo nmcli connection delete bond0
```

### 11.5. Windows Server

1. Перенесите IP с team, если он ещё нужен.
2. Откройте **Server Manager → Local Server → NIC Teaming**.
3. Выберите `Team-LACP`.
4. Нажмите **Tasks → Delete**.
5. Верните IP нужному физическому адаптеру.

---

<a id="mapping"></a>

## 12. Краткое соответствие параметров

| Задача               | MikroTik             | Rapier                | Linux               | Windows Server             |
| -------------------- | -------------------- | --------------------- | ------------------- | -------------------------- |
| LACP                 | `mode=802.3ad`       | `enable/add lacp`     | `mode=802.3ad`      | Teaming Mode LACP          |
| Active               | `LACP Mode=active`   | `mode=active`         | стандартно active   | LACP team                  |
| Passive              | `LACP Mode=passive`  | `mode=passive`        | зависит от драйвера | обычно не выбирается       |
| Fast                 | `LACP Rate=1sec`     | `periodic=fast`       | `lacp_rate=fast`    | `LacpTimer Fast`           |
| Slow                 | `LACP Rate=30secs`   | `periodic=slow`       | `lacp_rate=slow`    | `LacpTimer Slow`           |
| Статический LAG      | `balance-xor`        | `create switch trunk` | `balance-xor`       | зависит от драйвера        |
| Резервирование       | `active-backup`      | отдельная схема       | `active-backup`     | Switch Independent/standby |
| Hash                 | Transmit Hash Policy | `select=...`          | `xmit_hash_policy`  | Load Balancing Mode        |
| Логический интерфейс | bonding interface    | trunk                 | `bond0`             | Team adapter               |
| Access VLAN          | PVID + Untagged      | untagged VLAN         | IP на `bond0`       | IP на team                 |
| VLAN trunk           | bond как Tagged      | trunk в tagged VLAN   | VLAN поверх `bond0` | зависит от версии          |

---

## Минимальный алгоритм LACP

```text
1. Выбрать одинаковые физические линии на обеих сторонах.
2. Удалить физические порты из bridge.
3. Удалить с них IP.
4. Проверить скорость, duplex и MTU.
5. Создать LACP bond на обеих сторонах.
6. Добавить в bridge сам bond.
7. Настроить bond как access или trunk.
8. Проверить оба активных участника.
9. Отключить один кабель.
10. Убедиться, что связь сохранилась.
```
