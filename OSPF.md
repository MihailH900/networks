# OSPF в MikroTik RouterOS v7

**Файл:** `OSPF_MikroTik_RouterOS_v7.md`

---

# 1. Введение

OSPF (Open Shortest Path First) — протокол динамической маршрутизации класса Link-State.

Особенности:

* быстрое восстановление после отказов;
* отсутствие маршрутных петель;
* поддержка иерархии областей (Area);
* расчет оптимального маршрута по алгоритму SPF (Dijkstra);
* поддержка балансировки маршрутов;
* поддержка аутентификации;
* поддержка перераспределения маршрутов (Redistribution). ([Wikipedia][1])

В RouterOS v7 архитектура OSPF была полностью переработана:

* OSPFv2 и OSPFv3 объединены;
* отсутствуют дефолтные Instance и Area;
* вместо вкладки Networks используются Interface Templates;
* соседство и интерфейсы являются динамическими объектами мониторинга. ([MikroTik Help][2])

---

# 2. Пример лабораторной схемы

```text
        R1
      10.0.12.0/30
        |
        |
        R2
      10.0.23.0/30
        |
        |
        R3
```

LAN сети:

```text
R1 = 192.168.1.0/24
R2 = 192.168.2.0/24
R3 = 192.168.3.0/24
```

Все маршрутизаторы находятся в Backbone Area.

---

# 3. Router ID

## Назначение

Router ID — уникальный идентификатор маршрутизатора в OSPF.

Формат:

```text
32-битный IPv4 адрес
```

Пример:

```text
1.1.1.1
2.2.2.2
3.3.3.3
```

Router ID должен быть уникальным в пределах OSPF-домена. ([RFC][3])

---

## Настройка через WinBox

```text
Routing
  →
Router ID
```

Создаем:

```text
Name = RID-R1
ID = 1.1.1.1
```

---

## CLI

```bash
/routing id
add name=RID-R1 id=1.1.1.1
```

([RFC][3])

---

# 4. OSPF Instance

## Назначение

Instance определяет:

* Router ID;
* тип OSPF;
* VRF;
* распространение default route;
* redistribution.

В RouterOS v7 OSPF не работает без созданного Instance. ([MikroTik Help][2])

---

## Через WinBox

```text
Routing
 →
OSPF
 →
Instances
```

Создаем:

```text
Name = OSPF-MAIN
Version = 2
Router ID = RID-R1
```

---

## CLI

```bash
/routing ospf instance
add name=OSPF-MAIN \
router-id=RID-R1 \
version=2
```

---

# 5. Area 0 (Backbone)

## Что такое Area

Area — логическая область OSPF.

Обязательная область:

```text
0.0.0.0
```

или

```text
Area 0
```

Все остальные области должны быть связаны с Backbone Area. ([Wikipedia][1])

---

## Создание через WinBox

```text
Routing
 →
OSPF
 →
Areas
```

Создаем:

```text
Name = Backbone
Area ID = 0.0.0.0
Instance = OSPF-MAIN
```

---

## CLI

```bash
/routing ospf area
add instance=OSPF-MAIN \
name=Backbone \
area-id=0.0.0.0
```

([RFC][3])

---

# 6. Interface Templates

## Назначение

В RouterOS v7 интерфейсы добавляются через Interface Template.

Шаблон определяет:

* интерфейс;
* сеть;
* тип сети;
* cost;
* priority;
* authentication;
* passive mode. ([MikroTik Help][4])

---

## Через WinBox

```text
Routing
 →
OSPF
 →
Interface Templates
```

---

Пример:

```text
Area = Backbone
Interfaces = ether1
Networks = 10.0.12.0/30
Type = ptp
```

---

## CLI

```bash
/routing ospf interface-template
add area=Backbone \
interfaces=ether1 \
networks=10.0.12.0/30 \
type=ptp
```

([RFC][3])

---

# 7. Распространение сетей

В RouterOS v6 использовалась вкладка Networks.

В RouterOS v7 распространение выполняется через Interface Templates. ([MikroTik Help][2])

---

Пример LAN:

```bash
/routing ospf interface-template
add area=Backbone \
interfaces=bridge \
networks=192.168.1.0/24
```

---

# 8. Установление соседства

После настройки интерфейсов маршрутизаторы начинают обмениваться:

```text
Hello packets
```

---

Процесс соседства:

```text
Down
Init
2-Way
ExStart
Exchange
Loading
Full
```

---

Полностью работающее соседство:

```text
State = Full
```

([RFC][3])

---

# 9. Проверка соседей

## WinBox

```text
Routing
 →
OSPF
 →
Neighbors
```

---

## CLI

```bash
/routing ospf neighbor print
```

Пример:

```text
address=10.0.12.2
router-id=2.2.2.2
state=Full
```

([RFC][3])

---

# 10. DR и BDR

В сетях типа Broadcast выбираются:

```text
DR  (Designated Router)
BDR (Backup Designated Router)
```

Для уменьшения количества соседств.

---

Количество соседств:

Без DR:

```text
n(n-1)/2
```

С DR:

```text
n-1
```

---

Выбор производится по:

1. Priority;
2. Router ID.

([Wikipedia][1])

---

## Настройка Priority

```bash
/routing ospf interface-template
add area=Backbone \
interfaces=ether2 \
priority=200
```

---

Запрет участия в выборах:

```bash
priority=0
```

---

# 11. Cost

## Назначение

Cost определяет предпочтительность маршрута.

Меньший Cost = лучший путь.

---

Пример

Канал 1:

```text
Cost 10
```

Канал 2:

```text
Cost 100
```

Будет выбран первый.

---

## Настройка

```bash
/routing ospf interface-template
add area=Backbone \
interfaces=ether1 \
cost=10
```

([RFC][3])

---

# 12. Default Route

## Назначение

OSPF может распространять:

```text
0.0.0.0/0
```

от пограничного маршрутизатора.

---

Через WinBox:

```text
Routing
 →
OSPF
 →
Instances
```

Параметр:

```text
Originate Default
```

---

Варианты:

```text
always
if-installed
never
```

([MikroTik Help][5])

---

## CLI

```bash
/routing ospf instance
set OSPF-MAIN \
originate-default=if-installed
```

---

Постоянная генерация:

```bash
originate-default=always
```

([MikroTik Help][5])

---

# 13. Проверка LSDB

LSDB (Link State Database) содержит все LSA.

---

## CLI

```bash
/routing ospf lsa print
```

---

Проверка количества LSA:

```bash
/routing ospf lsa print count-only
```

---

Полезно для поиска:

* рассинхронизации;
* ошибок area;
* ошибок соседства.

---

# 14. Проверка таблицы маршрутизации

## Все маршруты OSPF

```bash
/ip/route/print where ospf
```

или

```bash
/routing/route/print
```

---

Пример:

```text
o 192.168.2.0/24
o 192.168.3.0/24
```

Буква:

```text
o = OSPF
```

---

# 15. Отказ канала

Пример:

```text
R1 --- R2 --- R3
```

Если линк:

```text
R1-R2
```

разрывается:

1. прекращаются Hello;
2. истекает Dead Timer;
3. удаляется сосед;
4. SPF пересчитывается;
5. выбирается резервный путь.

([Wikipedia][1])

---

# 16. Причины отсутствия соседства

## Несовпадение Area

```text
R1 = Area 0
R2 = Area 1
```

Соседства не будет.

---

## Разный Hello Timer

```text
10 сек
```

и

```text
5 сек
```

Соседства не будет.

---

## Разный Dead Timer

---

## Разная аутентификация

---

## Блокировка multicast

OSPF использует:

```text
224.0.0.5
224.0.0.6
```

Фильтрация этих адресов ломает соседство.

---

## Несовпадение MTU

Частая причина зависания на:

```text
ExStart
Exchange
```

---

## Проверка

```bash
/routing ospf neighbor print detail
```

([MikroTik community forum][6])

---

# 17. Диагностика

## Проверка интерфейсов

```bash
/routing ospf interface print
```

---

## Проверка соседей

```bash
/routing ospf neighbor print
```

---

## Проверка LSA

```bash
/routing ospf lsa print
```

---

## Проверка маршрутов

```bash
/routing route print where protocol=ospf
```

---

## Сниффер OSPF

```bash
/tool/sniffer
```

Фильтр:

```text
IP Protocol 89
```

---

# 18. Перераспределение маршрутов (Redistribution)

## Назначение

Redistribution используется для публикации в OSPF маршрутов:

* Static;
* Connected;
* BGP;
* RIP;
* VPN.

В RouterOS v7 перераспределение выполняется через параметры Instance и routing filters. ([MikroTik Help][5])

---

## Redistribute Connected

```bash
/routing ospf instance
set OSPF-MAIN \
redistribute=connected
```

---

## Redistribute Static

```bash
/routing ospf instance
set OSPF-MAIN \
redistribute=static
```

---

## Несколько источников

```bash
/routing ospf instance
set OSPF-MAIN \
redistribute=connected,static,bgp
```

([MikroTik community forum][7])

---

## Фильтрация маршрутов

Пример публикации только сети:

```text
10.10.10.0/24
```

Создаем фильтр:

```bash
/routing filter rule
add chain=ospf_out \
rule="if (dst == 10.10.10.0/24) {accept} else {reject}"
```

Затем применяем его к OSPF Instance.

([MikroTik Help][8])

---

# 19. Рекомендуемая минимальная конфигурация

```bash
/routing id
add name=RID-R1 id=1.1.1.1

/routing ospf instance
add name=OSPF-MAIN \
router-id=RID-R1 \
version=2 \
originate-default=if-installed

/routing ospf area
add instance=OSPF-MAIN \
name=Backbone \
area-id=0.0.0.0

/routing ospf interface-template
add area=Backbone \
interfaces=ether1 \
networks=10.0.12.0/30 \
type=ptp

/routing ospf interface-template
add area=Backbone \
interfaces=bridge \
networks=192.168.1.0/24
```

([RFC][3])

---

# 20. Краткое резюме

Для запуска OSPF в RouterOS v7 необходимо:

1. Создать Router ID.
2. Создать OSPF Instance.
3. Создать Backbone Area (0.0.0.0).
4. Настроить Interface Templates.
5. Проверить соседство (`Full`).
6. Проверить LSDB.
7. Проверить маршруты OSPF.
8. Настроить Cost и Priority при необходимости.
9. Настроить Originate Default для публикации маршрута по умолчанию.
10. Использовать Redistribution и Routing Filters для контролируемого импорта маршрутов.

Архитектура OSPF в RouterOS v7 существенно отличается от RouterOS v6, поэтому при миграции следует уделять особое внимание Instance, Area и Interface Templates. ([MikroTik Help][2])

[1]: https://en.wikipedia.org/wiki/Open_Shortest_Path_First?utm_source=chatgpt.com "Open Shortest Path First"
[2]: https://help.mikrotik.com/docs/spaces/ROS/pages/115736772/Upgrading%2Bto%2Bv7?utm_source=chatgpt.com "Upgrading to v7 - RouterOS - MikroTik Documentation"
[3]: https://rickfreyconsulting.com/rosv7-ospf-basic-configuration/?utm_source=chatgpt.com "ROSv7 – OSPF Basic Configuration - RFC"
[4]: https://help.mikrotik.com/docs/spaces/ROS/pages/9863229/OSPF?utm_source=chatgpt.com "OSPF - RouterOS - MikroTik Documentation"
[5]: https://help.mikrotik.com/docs/spaces/ROS/pages/331612216/routing%2Bospf?utm_source=chatgpt.com "routing/ospf - RouterOS"
[6]: https://forum.mikrotik.com/t/ospf-not-working-v7-15-1/176749?utm_source=chatgpt.com "OSPF not working V7.15.1 - Forwarding Protocols"
[7]: https://forum.mikrotik.com/t/ospf-routing-syntax/149617?utm_source=chatgpt.com "OSPF routing syntax - RouterOS beta"
[8]: https://help.mikrotik.com/docs/spaces/ROS/pages/74678285/Route%2BSelection%2Band%2BFilters?utm_source=chatgpt.com "Route Selection and Filters - RouterOS"
