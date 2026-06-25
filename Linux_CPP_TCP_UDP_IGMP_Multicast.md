# Linux C++ TCP/UDP сокеты и IGMP Multicast

**Файл:** `Linux_CPP_TCP_UDP_IGMP_Multicast.md`

---

# 1. Введение

В Linux для сетевого взаимодействия используются сокеты (sockets).

Основные типы:

| Тип           | Протокол    | Назначение                            |
| ------------- | ----------- | ------------------------------------- |
| TCP           | SOCK_STREAM | Надежное соединение                   |
| UDP           | SOCK_DGRAM  | Передача датаграмм                    |
| UDP Multicast | SOCK_DGRAM  | Один источник → множество получателей |

Multicast позволяет отправлять один поток данных сразу нескольким клиентам без создания отдельных соединений для каждого.

В отличие от Broadcast:

* Broadcast получают все устройства сегмента.
* Multicast получают только подписавшиеся клиенты.

Для IPv4 используются адреса:

```text
224.0.0.0 - 239.255.255.255
```

IGMP используется для управления членством в multicast-группах. Клиенты отправляют IGMP Join/Leave сообщения, а маршрутизаторы и коммутаторы используют эту информацию для доставки трафика только заинтересованным получателям. ([Wikipedia][1])

---

# 2. TCP сокеты

## Схема работы

```text
TCP Server
    |
listen()
    |
accept()
    |
TCP Client
```

---

## TCP Server (C++17)

```cpp
#include <arpa/inet.h>
#include <unistd.h>
#include <cstring>
#include <iostream>

int main()
{
    int server = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(5000);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(server, (sockaddr*)&addr, sizeof(addr));

    listen(server, 10);

    std::cout << "Waiting..." << std::endl;

    int client = accept(server, nullptr, nullptr);

    const char* msg = "Hello TCP";

    send(client, msg, strlen(msg), 0);

    close(client);
    close(server);
}
```

---

## TCP Client

```cpp
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>

int main()
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in server{};
    server.sin_family = AF_INET;
    server.sin_port = htons(5000);

    inet_pton(AF_INET, "192.168.1.1", &server.sin_addr);

    connect(sock, (sockaddr*)&server, sizeof(server));

    char buffer[1024];

    recv(sock, buffer, sizeof(buffer), 0);

    std::cout << buffer << std::endl;

    close(sock);
}
```

---

# 3. UDP сокеты

UDP не использует соединения.

Сервер просто слушает порт.

---

## UDP Server

```cpp
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>

int main()
{
    int sock = socket(AF_INET, SOCK_DGRAM, 0);

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(5000);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(sock, (sockaddr*)&addr, sizeof(addr));

    char buffer[1024];

    recv(sock, buffer, sizeof(buffer), 0);

    std::cout << buffer << std::endl;
}
```

---

## UDP Client

```cpp
#include <arpa/inet.h>
#include <unistd.h>

int main()
{
    int sock = socket(AF_INET, SOCK_DGRAM, 0);

    sockaddr_in dst{};
    dst.sin_family = AF_INET;
    dst.sin_port = htons(5000);

    inet_pton(AF_INET,
              "192.168.1.10",
              &dst.sin_addr);

    sendto(
        sock,
        "Hello UDP",
        9,
        0,
        (sockaddr*)&dst,
        sizeof(dst));
}
```

---

# 4. Что такое IGMP Multicast

Multicast работает по схеме:

```text
        Server
           |
      239.1.1.1
           |
   ----------------
   |      |      |
Client1 Client2 Client3
```

Сервер отправляет только одну копию пакета.

Коммутатор и маршрутизатор размножают трафик только там, где есть подписчики. ([MikroTik Help][2])

---

# 5. Создание Multicast сервера

Сервер НЕ выполняет Join.

Он только отправляет пакеты на multicast адрес.

---

## Сервер

```cpp
#include <arpa/inet.h>
#include <unistd.h>
#include <string>

int main()
{
    int sock =
        socket(AF_INET, SOCK_DGRAM, 0);

    sockaddr_in group{};
    group.sin_family = AF_INET;
    group.sin_port = htons(5000);

    inet_pton(
        AF_INET,
        "239.1.1.1",
        &group.sin_addr);

    while(true)
    {
        std::string msg = "HELLO";

        sendto(
            sock,
            msg.c_str(),
            msg.size(),
            0,
            (sockaddr*)&group,
            sizeof(group));

        sleep(1);
    }
}
```

---

# 6. Создание Multicast клиента

Клиент обязан:

1. открыть UDP сокет;
2. выполнить bind();
3. выполнить Join группы;
4. принимать пакеты.

Для Join используется:

```cpp
setsockopt(
    socket,
    IPPROTO_IP,
    IP_ADD_MEMBERSHIP,
    ...
);
```

Это основной механизм подписки на multicast-группу в Linux. ([man7.org][3])

---

## Клиент

```cpp
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>

int main()
{
    int sock =
        socket(AF_INET, SOCK_DGRAM, 0);

    int reuse = 1;

    setsockopt(
        sock,
        SOL_SOCKET,
        SO_REUSEADDR,
        &reuse,
        sizeof(reuse));

    sockaddr_in local{};
    local.sin_family = AF_INET;
    local.sin_port = htons(5000);
    local.sin_addr.s_addr = INADDR_ANY;

    bind(
        sock,
        (sockaddr*)&local,
        sizeof(local));

    ip_mreq mreq{};

    mreq.imr_multiaddr.s_addr =
        inet_addr("239.1.1.1");

    mreq.imr_interface.s_addr =
        INADDR_ANY;

    setsockopt(
        sock,
        IPPROTO_IP,
        IP_ADD_MEMBERSHIP,
        &mreq,
        sizeof(mreq));

    char buffer[1024];

    while(true)
    {
        int n = recv(
            sock,
            buffer,
            sizeof(buffer),
            0);

        std::cout.write(buffer, n);
        std::cout << std::endl;
    }
}
```

---

# 7. Выбор интерфейса

Если сервер имеет несколько сетевых карт:

```text
eth0
eth1
bond0
```

можно указать интерфейс для multicast:

```cpp
in_addr iface{};

iface.s_addr =
    inet_addr("192.168.100.10");

setsockopt(
    sock,
    IPPROTO_IP,
    IP_MULTICAST_IF,
    &iface,
    sizeof(iface));
```

Linux поддерживает настройку через `IP_MULTICAST_IF` и `IP_ADD_MEMBERSHIP`. ([man7.org][3])

---

# 8. Проверка членства в группе

Показать multicast группы:

```bash
ip maddr show
```

Показать состояние IGMP:

```bash
cat /proc/net/igmp
```

Эти команды позволяют убедиться, что клиент успешно выполнил Join. ([OneUptime][4])

---

# 9. Почему трафик передается только на заинтересованные порты

Это один из самых важных вопросов multicast.

---

## Без IGMP Snooping

```text
         Switch

      +----------+
      |          |
      +----------+
      | | | | | |
      A B C D E F
```

Пакет multicast будет отправлен на ВСЕ порты.

Фактически получится аналог broadcast. ([MikroTik Help][2])

---

## С IGMP Snooping

Пусть:

```text
B -> Join 239.1.1.1
D -> Join 239.1.1.1
```

Тогда таблица коммутатора выглядит так:

```text
239.1.1.1 -> B,D
```

При поступлении multicast пакета:

```text
239.1.1.1
```

он будет отправлен только:

```text
B
D
```

Остальные порты пакет не увидят. IGMP Snooping анализирует IGMP Join/Leave сообщения и строит таблицу соответствия «группа → порт». ([MikroTik Help][2])

---

# 10. Требование наличия Querier

Для корректной работы IGMP Snooping в сети обычно должен существовать IGMP Querier.

Им может быть:

* маршрутизатор;
* L3-коммутатор;
* MikroTik с включенным Querier.

Без Querier таблицы членства могут устаревать или не формироваться корректно, из-за чего multicast может снова начать флудиться по VLAN. ([Wikipedia][5])

---

# 11. Диагностика

## Проверка multicast пакетов

```bash
tcpdump -ni eth0 multicast
```

или

```bash
tcpdump -ni eth0 host 239.1.1.1
```

---

## Проверка IGMP

```bash
tcpdump -ni eth0 igmp
```

---

## Проверка сокетов

```bash
ss -ulpn
```

---

## Проверка маршрутов multicast

```bash
ip route
```

---

# 12. Типичные проблемы

## Клиент не получает трафик

Проверить:

```bash
cat /proc/net/igmp
```

Нет группы → Join не выполнен.

---

## Пакеты идут на все порты

Проверить:

```text
IGMP Snooping
IGMP Querier
```

на коммутаторе.

---

## Несколько интерфейсов

Указать:

```cpp
IP_MULTICAST_IF
```

---

## Несколько клиентов на одном ПК

Использовать:

```cpp
SO_REUSEADDR
```

---

# 13. Краткое резюме

Для multicast в Linux необходимо:

1. Создать UDP сокет.
2. Выполнить `bind()`.
3. Подписаться через `IP_ADD_MEMBERSHIP`.
4. При необходимости указать интерфейс через `IP_MULTICAST_IF`.
5. Проверить членство через `ip maddr show`.
6. Использовать IGMP Snooping на коммутаторах.
7. Обеспечить наличие IGMP Querier.
8. Для доставки трафика только заинтересованным клиентам коммутатор должен строить таблицу членства на основе IGMP Join/Leave сообщений и пересылать multicast только на порты подписчиков. ([MikroTik Help][2])

[1]: https://en.wikipedia.org/wiki/Internet_Group_Management_Protocol?utm_source=chatgpt.com "Internet Group Management Protocol"
[2]: https://help.mikrotik.com/docs/spaces/ROS/pages/59277403/Bridge%2BIGMP%2BMLD%2Bsnooping?utm_source=chatgpt.com "Bridge IGMP/MLD snooping - RouterOS"
[3]: https://man7.org/linux/man-pages/man7/ip.7.html?utm_source=chatgpt.com "ip(7) - Linux manual page"
[4]: https://oneuptime.com/blog/post/2026-03-20-join-multicast-group-linux/view?utm_source=chatgpt.com "How to Join a Multicast Group on a Linux Interface"
[5]: https://en.wikipedia.org/wiki/IGMP_snooping?utm_source=chatgpt.com "IGMP snooping"
