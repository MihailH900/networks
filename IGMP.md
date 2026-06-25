# IGMP и Multicast в MikroTik RouterOS v7

**Файл:** `IGMP_Multicast_MikroTik_RouterOS_v7.md`

---

# 1. Введение

Multicast — технология доставки одного потока данных нескольким получателям одновременно.

Наиболее частые применения:

* IPTV;
* видеонаблюдение;
* аудио-вещание;
* системы видеостен;
* Dante;
* mDNS и сервисы обнаружения устройств.

В IPv4 multicast использует диапазон:

```text
224.0.0.0 – 239.255.255.255
```

Для управления членством в группах используется IGMP (Internet Group Management Protocol). Клиенты отправляют Join/Leave запросы, а коммутаторы и маршрутизаторы строят таблицы подписчиков и пересылают трафик только туда, где есть получатели. ([MikroTik Help][1])

---

# 2. Компоненты multicast в MikroTik

В RouterOS v7 используются следующие механизмы:

| Функция       | Назначение                          |
| ------------- | ----------------------------------- |
| IGMP          | Подписка клиентов на группы         |
| IGMP Snooping | Фильтрация multicast на L2          |
| IGMP Querier  | Опрос подписчиков                   |
| IGMP Proxy    | Передача multicast между подсетями  |
| PIM-SM        | Полноценная multicast-маршрутизация |

Для данной инструкции рассматриваются только:

* IGMP;
* IGMP Snooping;
* IGMP Proxy.

---

# 3. Базовая схема работы

```text
              Multicast Source
                      |
                 239.1.1.1
                      |
                MikroTik Router
                      |
              -----------------
              |               |
            VLAN10         VLAN20
              |               |
          Client A       Client B
```

---

# 4. IGMP

## Назначение

IGMP используется клиентами для подписки на multicast-группы.

Пример:

```text
Client -> Join 239.1.1.1
```

После получения Join маршрутизатор или коммутатор понимает, что в сегменте появился подписчик.

---

## Версии IGMP

| Версия | Возможности               |
| ------ | ------------------------- |
| IGMPv1 | Базовая                   |
| IGMPv2 | Join/Leave                |
| IGMPv3 | Source-Specific Multicast |

RouterOS поддерживает IGMP v1/v2/v3. ([MikroTik Help][2])

---

# 5. IGMP Snooping

## Зачем нужен

Без IGMP Snooping любой multicast-фрейм будет отправляться на все порты bridge.

```text
Source
   |
 Switch
 / | \
A  B  C
```

Все устройства получают поток.

---

## С включенным Snooping

Пусть:

```text
B -> Join 239.1.1.1
```

Тогда bridge создаёт MDB:

```text
239.1.1.1 -> ether2
```

Пакеты будут отправляться только на ether2. Это значительно уменьшает нагрузку на сеть. ([MikroTik Help][1])

---

# 6. Включение IGMP Snooping через WinBox

Путь:

```text
Bridge
    →
    Bridge
        →
        Edit
```

Включить:

```text
IGMP Snooping = yes
```

---

## CLI

```bash
/interface bridge
set bridge1 igmp-snooping=yes
```

([MikroTik community forum][3])

---

# 7. Проверка MDB таблицы

MDB (Multicast Database) содержит информацию:

```text
Group -> Ports
```

---

## CLI

```bash
/interface bridge mdb print
```

Пример:

```text
GROUP          PORTS
239.1.1.1      ether2
239.1.1.1      ether3
```

([MikroTik community forum][3])

---

# 8. IGMP Querier

## Назначение

Querier периодически отправляет запросы:

```text
Who is subscribed?
```

Если клиент не отвечает, запись удаляется.

Без Querier таблицы членства могут устареть. ([MikroTik Help][1])

---

## Включение через WinBox

```text
Bridge
  →
  Bridge
    →
    Edit
```

Включить:

```text
Multicast Querier
```

---

## CLI

```bash
/interface bridge
set bridge1 multicast-querier=yes
```

([MikroTik Help][4])

---

# 9. Fast Leave

Используется для IPTV.

При выходе клиента из группы поток немедленно прекращает доставляться на порт.

---

## CLI

```bash
/interface bridge port
set ether2 fast-leave=yes
```

Рекомендуется для STB приставок.

([MikroTik Help][5])

---

# 10. Проверка работы Snooping

## Bridge Monitor

```bash
/interface bridge monitor bridge1
```

Проверяем:

```text
igmp-snooping: yes
multicast-querier: yes
```

---

## MDB

```bash
/interface bridge mdb print
```

---

## Torch

```bash
/tool torch
```

---

## Packet Sniffer

```bash
/tool/sniffer
```

Фильтр:

```text
Protocol = IGMP
```

---

# 11. Типичные проблемы Snooping

## Multicast идет на все порты

Причины:

* Snooping выключен;
* отсутствует Querier;
* клиент не отправляет Join;
* используется неизвестная multicast-группа.

([MikroTik Help][1])

---

## Таблица MDB пустая

Проверить:

```bash
/interface bridge mdb print
```

Если записей нет:

* нет IGMP Join;
* нет Querier;
* клиент использует Broadcast вместо Multicast.

---

## VLAN и Snooping

При использовании VLAN необходимо помнить:

* Querier в RouterOS имеет ограничения при работе с несколькими VLAN;
* необходимо внимательно проверять формирование MDB для каждого VLAN. ([MikroTik community forum][6])

---

# 12. IGMP Proxy

## Назначение

IGMP Proxy используется для multicast-маршрутизации между подсетями без PIM.

Наиболее распространенный сценарий:

```text
IPTV Provider
      |
WAN (upstream)
      |
MikroTik
      |
LAN (downstream)
      |
STB
```

IGMP Proxy является наиболее простым способом организации multicast routing в RouterOS. ([MikroTik Help][7])

---

# 13. Схема работы IGMP Proxy

```text
        Source
           |
       Upstream
           |
      MikroTik
           |
      Downstream
           |
      Receivers
```

---

# 14. Настройка IGMP Proxy через WinBox

Путь:

```text
Routing
   →
   IGMP Proxy
```

---

## Включение

Создать интерфейсы:

### Upstream

```text
Interface = ether1
Upstream = yes
```

### Downstream

```text
Interface = bridge
```

---

# 15. Настройка через CLI

Включение Proxy:

```bash
/routing igmp-proxy
set quick-leave=yes
```

Upstream:

```bash
/routing igmp-proxy interface
add interface=ether1 upstream=yes
```

Downstream:

```bash
/routing igmp-proxy interface
add interface=bridge
```

([MikroTik Help][7])

---

# 16. Проверка IGMP Proxy

Просмотр интерфейсов:

```bash
/routing igmp-proxy interface print
```

---

Просмотр групп:

```bash
/routing igmp-proxy membership print
```

---

Просмотр MFC

(Multicast Forwarding Cache)

```bash
/routing igmp-proxy mfc print
```

---

# 17. Multicast между подсетями без PIM

Это один из самых важных сценариев.

Пусть есть:

```text
VLAN10
192.168.10.0/24

VLAN20
192.168.20.0/24
```

Источник расположен в VLAN10.

Получатели расположены в VLAN20.

---

## Решение

Настраиваем:

```text
VLAN10 = upstream
VLAN20 = downstream
```

через IGMP Proxy.

После этого:

```text
Join -> Proxy -> Source
```

и multicast начинает проходить между подсетями.

IGMP Proxy специально предназначен для подобных сценариев, когда полноценный PIM-SM не требуется. ([MikroTik Help][7])

---

# 18. Диагностика

## Просмотр IGMP

```bash
/tool sniffer
```

Фильтр:

```text
ip-protocol=2
```

---

## Проверка групп

```bash
/interface bridge mdb print
```

---

## Проверка Proxy

```bash
/routing igmp-proxy membership print
```

---

## Проверка трафика

```bash
/tool torch
```

---

# 19. Рекомендации для IPTV

В большинстве IPTV-сетей рекомендуется:

```bash
/interface bridge
set bridge1 \
igmp-snooping=yes \
multicast-querier=yes
```

и

```bash
/routing igmp-proxy
set quick-leave=yes
```

Это уменьшает флуд multicast-потоков и обеспечивает корректное распространение IPTV между сегментами сети. ([MikroTik Help][5])

---

# 20. Краткое резюме

* IGMP отвечает за подписку клиентов.
* IGMP Snooping ограничивает multicast только заинтересованными портами.
* MDB показывает соответствие «группа → порт».
* Querier поддерживает актуальность таблиц подписчиков.
* IGMP Proxy обеспечивает multicast-маршрутизацию между подсетями без использования PIM.
* Для IPTV почти всегда используются одновременно IGMP Snooping и IGMP Proxy.
* Основные команды диагностики:

```bash
/interface bridge mdb print
/routing igmp-proxy membership print
/routing igmp-proxy mfc print
/tool sniffer
/tool torch
```

что позволяет полностью контролировать процесс доставки multicast-трафика в RouterOS v7. ([MikroTik Help][1])

[1]: https://help.mikrotik.com/docs/spaces/ROS/pages/59277403/Bridge%2BIGMP%2BMLD%2Bsnooping?utm_source=chatgpt.com "Bridge IGMP/MLD snooping - RouterOS"
[2]: https://help.mikrotik.com/docs/spaces/ROS/pages/131366949/Group%2BManagement%2BProtocol?utm_source=chatgpt.com "Group Management Protocol - RouterOS"
[3]: https://forum.mikrotik.com/t/igmp-snooping-configuration-and-testing-information/110531?utm_source=chatgpt.com "IGMP Snooping Configuration and Testing Information"
[4]: https://help.mikrotik.com/docs/spaces/ROS/pages/328068/Bridging%2Band%2BSwitching?utm_source=chatgpt.com "Bridging and Switching - RouterOS - MikroTik Documentation"
[5]: https://help.mikrotik.com/docs/spaces/ROS/pages/189497483/Quality%2Bof%2BService?utm_source=chatgpt.com "Quality of Service - RouterOS - MikroTik Documentation"
[6]: https://forum.mikrotik.com/t/igmp-snooping-with-vlans/152875?utm_source=chatgpt.com "IGMP Snooping with VLANs - General"
[7]: https://help.mikrotik.com/docs/spaces/ROS/pages/128221386/IGMP%2BProxy?utm_source=chatgpt.com "IGMP Proxy - RouterOS - MikroTik Documentation"
