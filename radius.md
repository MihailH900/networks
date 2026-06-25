# RADIUS, 802.1X и MAC-аутентификация

Основная серверная реализация соответствует лабораторной работе:

```text
Windows Server 2003 IAS
+
MikroTik или Rapier как authenticator
+
Windows-клиент как supplicant
```

Для современных Windows Server приведена дополнительная ветка NPS, для Linux — резервная ветка FreeRADIUS.

Перед началом подготовьте оборудование по [`00-basic-setup`](../00-basic-setup/) и VLAN по [`02-vlan`](../02-vlan/).

> **Важно:** Windows Server 2003 и EAP-MD5 устарели и небезопасны. Они используются здесь только для воспроизведения изолированной лабораторной работы. В реальной сети применяйте NPS или FreeRADIUS с PEAP либо EAP-TLS.

Обозначения:

* `<RADIUS_IP>` — IP-адрес RADIUS-сервера;
* `<NAS_IP>` — IP-адрес MikroTik или Rapier;
* `<SECRET>` — общий RADIUS secret;
* `<CLIENT_PORT>` — порт подключаемого клиента;
* `<TRUNK_PORT>` — trunk до маршрутизатора или другого коммутатора;
* `<CLIENT_MAC>` — MAC-адрес клиента;
* `<WORK_VLAN>` — рабочая VLAN;
* `<REJECT_VLAN>` — карантинная VLAN;
* `<FAIL_VLAN>` — VLAN при недоступности RADIUS.

В основных примерах используются:

```text
Windows Server IAS: 192.168.100.10/24
MikroTik/Rapier:    192.168.100.2/24
RADIUS secret:      LabRadius2026!

Рабочая VLAN:       20
Карантинная VLAN:   999
Server-fail VLAN:   998

Trunk MikroTik:     ether1
Клиентский порт:    ether2
Management-порт:    ether10
```

---

## Содержание

1. [Общая топология](#topology)
2. [Подготовка VLAN и management-связности](#network-preparation)
3. [Установка и настройка Windows Server 2003 IAS](#ias)
4. [802.1X через MikroTik](#mikrotik-dot1x)
5. [802.1X через Rapier](#rapier-dot1x)
6. [Настройка Windows-клиента](#windows-supplicant)
7. [MAC-аутентификация](#mac-auth)
8. [Динамическое назначение VLAN](#dynamic-vlan)
9. [Guest, Reject и Server-Fail VLAN](#fallback-vlans)
10. [Piggybacking и ограничение по MAC](#piggybacking)
11. [Современный Windows Server NPS](#nps)
12. [FreeRADIUS и Linux-клиент](#freeradius)
13. [Удаление конфигурации](#cleanup)
14. [Краткое соответствие параметров](#mapping)
15. [Минимальные алгоритмы](#minimal-algorithms)

---

<a id="topology"></a>

## 1. Общая топология

### 1.1. Компоненты

| Компонент               | Роль                                     |
| ----------------------- | ---------------------------------------- |
| Windows Server 2003 IAS | RADIUS authentication server             |
| MikroTik или Rapier     | RADIUS client/NAS и 802.1X authenticator |
| Windows-клиент          | 802.1X supplicant                        |
| Рабочая VLAN            | сеть успешно авторизованных клиентов     |
| Карантинная VLAN        | сеть клиентов, получивших Access-Reject  |
| Management-сеть         | связь коммутатора с RADIUS-сервером      |

### 1.2. Физическая схема

```text
                         Management-сеть
             ┌────────────────────────────────┐
             │                                │
Windows Server IAS                    MikroTik/Rapier
192.168.100.10                         192.168.100.2
                                             │
                                      ether1 │ trunk
                                             │
                                    рабочие VLAN
                                             │
                                      ether2 │
                                             │
                                      Windows-клиент
```

RADIUS-сервер не должен находиться за тем же клиентским портом, который он авторизует.

Связь:

```text
MikroTik/Rapier → IAS
```

должна работать независимо от состояния `ether2`.

### 1.3. Порты

| Порт           | Назначение                            |
| -------------- | ------------------------------------- |
| `ether1`       | trunk VLAN 20, 998 и 999              |
| `ether2`       | клиентский порт с 802.1X или MAC-auth |
| `ether10`      | отдельное управление MikroTik         |
| серверный порт | подключение IAS к management-сети     |

### 1.4. IP-план

| Устройство      | Адрес               | Шлюз                 |
| --------------- | ------------------- | -------------------- |
| IAS             | `192.168.100.10/24` | шлюз management-сети |
| MikroTik/Rapier | `192.168.100.2/24`  | при необходимости    |
| клиент VLAN 20  | через DHCP          | `192.168.20.1`       |
| клиент VLAN 999 | через DHCP          | `192.168.99.1`       |
| клиент VLAN 998 | через DHCP          | `192.168.98.1`       |

---

<a id="network-preparation"></a>

## 2. Подготовка VLAN и management-связности

### 2.1. Требуемые VLAN

| VLAN | Назначение                                    |
| ---: | --------------------------------------------- |
|   20 | авторизованные сотрудники                     |
|  999 | Access-Reject / карантин                      |
|  998 | RADIUS недоступен                             |
|  100 | управление, если используется management VLAN |

Для каждой клиентской VLAN подготовьте:

* IP-шлюз;
* DHCP-пул либо статическую адресацию;
* необходимые firewall-правила;
* tagged-членство trunk-порта.

### 2.2. MikroTik: подготовить bridge

Используйте существующий VLAN-aware bridge либо создайте новый:

> **Bridge → Bridge → Add New**

```text
Name: br-access
Protocol Mode: rstp
VLAN Filtering: no
```

Добавьте trunk:

> **Bridge → Ports → Add New**

```text
Interface: ether1
Bridge: br-access
Ingress Filtering: yes
Frame Types: admit only VLAN tagged
Edge: no
```

Добавьте клиентский порт:

```text
Interface: ether2
Bridge: br-access
Ingress Filtering: yes
Frame Types: admit only untagged and priority tagged
Edge: yes
```

Не добавляйте `ether2` статически как untagged-порт рабочей VLAN: членство будет назначаться после аутентификации.

### 2.3. MikroTik: подготовить Bridge VLAN Table

> **Bridge → VLANs**

VLAN 20:

```text
VLAN IDs: 20
Tagged: ether1
Untagged: пусто
```

VLAN 999:

```text
VLAN IDs: 999
Tagged: ether1
Untagged: пусто
```

VLAN 998:

```text
VLAN IDs: 998
Tagged: ether1
Untagged: пусто
```

Если этот же MikroTik является шлюзом VLAN, добавьте `br-access` в `Tagged`:

```text
Tagged: ether1, br-access
```

и создайте VLAN-интерфейсы поверх `br-access`.

После заполнения таблицы включите:

> **Bridge → Bridge → br-access**

```text
VLAN Filtering: yes
```

### 2.4. Проверить management-связность

С IAS должен отвечать адрес NAS:

```text
192.168.100.2
```

С MikroTik должен отвечать IAS:

> **Tools → Ping**

```text
Address: 192.168.100.10
Src. Address: 192.168.100.2
```

Для Rapier выполните:

```text
ping 192.168.100.10
```

До продолжения RADIUS-сервер и authenticator должны иметь двустороннюю IP-связь.

---

<a id="ias"></a>

## 3. Установка и настройка Windows Server 2003 IAS

### 3.1. Настроить статический IP

Откройте:

> **Панель управления → Сетевые подключения**

Выберите серверный Ethernet-адаптер:

> **Свойства → Internet Protocol (TCP/IP) → Свойства**

Укажите:

```text
IP-адрес: 192.168.100.10
Маска:    255.255.255.0
Шлюз:     <MANAGEMENT_GATEWAY>
DNS:      <DNS_SERVER>
```

Для полностью локальной схемы шлюз и DNS могут не требоваться.

### 3.2. Установить IAS

Откройте:

> **Панель управления → Установка и удаление программ → Установка компонентов Windows**

Выберите:

> **Сетевые службы → Состав**

Отметьте:

```text
Internet Authentication Service
```

Завершите установку.

### 3.3. Запустить IAS

Откройте:

> **Панель управления → Администрирование → Internet Authentication Service**

Либо запустите MMC и добавьте оснастку IAS.

### 3.4. Запустить службу

Откройте:

> **Администрирование → Службы**

Найдите:

```text
Internet Authentication Service
```

Установите:

```text
Тип запуска: Автоматически
Состояние: Запущена
```

### 3.5. Создать локальную группу

Откройте:

> **Администрирование → Управление компьютером → Локальные пользователи и группы → Группы**

Создайте группу:

```text
NetworkAccess
```

### 3.6. Создать пользователя

Откройте:

> **Локальные пользователи и группы → Пользователи**

Создайте:

```text
Имя: student1
Пароль: NetLab2026!
```

Для лаборатории можно включить:

```text
Срок действия пароля не ограничен
```

Добавьте `student1` в группу:

```text
NetworkAccess
```

В свойствах пользователя на вкладке удалённого доступа выберите:

```text
Управлять доступом через политику удалённого доступа
```

### 3.7. Включить обратимое хранение пароля для EAP-MD5

Этот пункт нужен только для старого лабораторного EAP-MD5.

Откройте:

> **Администрирование → Локальная политика безопасности**

Перейдите:

> **Политики учётных записей → Политика паролей**

Включите:

```text
Хранить пароли с использованием обратимого шифрования
```

После включения ещё раз измените пароль пользователя `student1`, чтобы он был сохранён в требуемом формате.

Не используйте эту настройку в реальной рабочей среде.

### 3.8. Добавить MikroTik или Rapier как RADIUS Client

В IAS откройте:

> **RADIUS Clients → New RADIUS Client**

Укажите:

| Поле           | Значение                |
| -------------- | ----------------------- |
| Friendly Name  | `MIKROTIK` или `RAPIER` |
| Client Address | `192.168.100.2`         |
| Client-Vendor  | `RADIUS Standard`       |
| Shared Secret  | `LabRadius2026!`        |

IP должен совпадать с адресом, с которого NAS отправляет RADIUS-запросы.

### 3.9. Разрешить RADIUS в Windows Firewall

Откройте:

> **Панель управления → Windows Firewall → Исключения → Добавить порт**

Добавьте:

```text
RADIUS Authentication
UDP 1812
```

и:

```text
RADIUS Accounting
UDP 1813
```

Разрешайте эти порты только из management-сети.

---

<a id="mikrotik-dot1x"></a>

## 4. 802.1X через MikroTik

### 4.1. Добавить IAS как RADIUS-сервер

Откройте:

> **RADIUS → Add New**

Укажите:

| Поле                | Значение         |
| ------------------- | ---------------- |
| Service             | `dot1x`          |
| Address             | `192.168.100.10` |
| Secret              | `LabRadius2026!` |
| Authentication Port | `1812`           |
| Accounting Port     | `1813`           |
| Src. Address        | `192.168.100.2`  |
| Timeout             | `1s–3s`          |

Значение `Src. Address` должно совпадать с RADIUS Client, созданным в IAS.

### 4.2. Создать 802.1X-политику в IAS

В IAS откройте:

> **Remote Access Policies → New Remote Access Policy**

Используйте мастер.

Укажите:

```text
Policy Name: Wired-8021X
Access Method: Ethernet
```

В качестве условия доступа выберите группу:

```text
NetworkAccess
```

В качестве EAP-метода выберите:

```text
MD5-Challenge
```

Установите:

```text
Grant remote access permission
```

Разместите эту политику выше общих запрещающих или менее специфичных политик.

### 4.3. Включить Dot1X Server на MikroTik

В WebFig раздел может называться:

```text
Dot1X → Server
```

или:

```text
Interfaces → Dot1X → Server
```

Нажмите **Add New**.

Укажите:

| Поле            | Значение |
| --------------- | -------- |
| Interface       | `ether2` |
| Auth Types      | `dot1x`  |
| Accounting      | `yes`    |
| Auth Timeout    | `1m`     |
| Reauth Timeout  | `30m`    |
| Retrans Timeout | `3s`     |
| Interim Update  | `5m`     |

Для базовой авторизации не задавайте Guest, Reject и Server-Fail VLAN. Тогда до успешной авторизации пользовательский трафик будет заблокирован.

### 4.4. Состояние порта до авторизации

До успешной аутентификации через `ether2` пропускаются EAPOL-пакеты, но обычный пользовательский трафик блокируется.

В:

> **Dot1X → Server → State**

ожидается:

```text
Interface: ether2
Status: un-authorized
```

### 4.5. Состояние после авторизации

После успешного входа откройте:

> **Dot1X → Server → Active**

Проверьте:

```text
Interface: ether2
Username: student1
Client MAC: MAC клиента
Auth Info: dot1x
```

Если серверная политика возвращает VLAN, здесь также отображается `VLAN ID`.

---

<a id="rapier-dot1x"></a>

## 5. 802.1X через Rapier

Настройка выполняется через AlliedWare CLI.

В примере:

```text
порт 2 — клиентский authenticator-порт
```

### 5.1. Добавить IAS

```text
add radius server=192.168.100.10 secret=LabRadius2026!
```

### 5.2. Включить 802.1X глобально

```text
enable portauth=8021x
```

### 5.3. Включить authenticator на порту

Базовый вариант лабораторной:

```text
enable portauth=8021x port=2 type=authenticator
```

Полная безопасная конфигурация одного клиента:

```text
enable portauth=8021x port=2 type=authenticator control=auto mode=single piggyback=false reauthenabled=true reauthperiod=1800 vlanassignment=enabled
```

Значения:

| Параметр                 | Назначение                                     |
| ------------------------ | ---------------------------------------------- |
| `control=auto`           | состояние определяется результатом авторизации |
| `mode=single`            | один основной supplicant                       |
| `piggyback=false`        | блокировать другие MAC после авторизации       |
| `reauthenabled=true`     | периодически повторять авторизацию             |
| `reauthperiod=1800`      | период 30 минут                                |
| `vlanassignment=enabled` | принимать VLAN от RADIUS                       |

### 5.4. Проверить состояние

```text
show portauth
```

Подробно по порту:

```text
show portauth port=2
```

Также проверьте:

```text
show vlan
```

### 5.5. Принудительно разрешить или запретить порт

Для обычной работы используйте:

```text
control=auto
```

Принудительно открыть без проверки:

```text
enable portauth=8021x port=2 type=authenticator control=authorised
```

Принудительно закрыть:

```text
enable portauth=8021x port=2 type=authenticator control=unauthorised
```

Эти режимы используются только для проверки или аварийной настройки.

---

<a id="windows-supplicant"></a>

## 6. Настройка Windows-клиента

### 6.1. Запустить службу проводной автонастройки

Откройте:

> **Панель управления → Администрирование → Службы**

Найдите:

```text
Wired AutoConfig
Проводная автонастройка
```

Установите:

```text
Тип запуска: Автоматически
Состояние: Запущена
```

На старых образах Windows XP название службы может отличаться, но она отвечает за 802.1X на проводном адаптере.

### 6.2. Включить 802.1X на адаптере

Откройте:

> **Сетевые подключения → Ethernet → Свойства → Проверка подлинности**

Включите:

```text
Включить проверку подлинности IEEE 802.1X для этой сети
```

Выберите:

```text
MD5-Challenge
```

Откройте дополнительные параметры и укажите:

```text
Режим проверки подлинности: Проверка подлинности пользователя
```

Для лаборатории отключите:

```text
Запоминать мои учётные данные для этого подключения
```

### 6.3. Выполнить вход

Переподключите Ethernet-кабель либо отключите и включите адаптер.

Введите:

```text
Пользователь: student1
Пароль: NetLab2026!
```

При использовании локальной базы IAS может потребоваться:

```text
SERVERNAME\student1
```

После Access-Accept порт станет авторизованным.

### 6.4. EAP-MD5 на Windows Vista и новее

В современных клиентских Windows EAP-MD5 обычно отсутствует.

Для совместимости с лабораторией откройте Registry Editor:

```text
regedit
```

Перейдите:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\RasMan\PPP\EAP\4
```

Создайте значения:

| Имя                    | Тип             | Значение                            |
| ---------------------- | --------------- | ----------------------------------- |
| `RolesSupported`       | `REG_DWORD`     | `0000000a`                          |
| `FriendlyName`         | `REG_SZ`        | `MD5-Challenge`                     |
| `Path`                 | `REG_EXPAND_SZ` | `%SystemRoot%\System32\Raschap.dll` |
| `InvokeUsernameDialog` | `REG_DWORD`     | `1`                                 |
| `InvokePasswordDialog` | `REG_DWORD`     | `1`                                 |

Перезапустите службу Wired AutoConfig либо перезагрузите клиент.

Используйте эту настройку только в лабораторной среде.

---

<a id="mac-auth"></a>

## 7. MAC-аутентификация

MAC-аутентификация применяется для устройств без 802.1X supplicant:

* принтеров;
* кассовых терминалов;
* информационных экранов;
* устаревших устройств;
* простых embedded-систем.

MAC-адрес не является секретом, поэтому MAC-auth слабее полноценного 802.1X.

### 7.1. Определить MAC клиента

На Windows:

> **Состояние Ethernet → Сведения → Физический адрес**

Пример:

```text
00-11-22-33-44-55
```

На Linux:

```bash
ip link show <INTERFACE>
```

Заранее выберите единый формат:

```text
00-11-22-33-44-55
```

Формат на IAS и MikroTik должен совпадать.

---

## 7.2. Настроить IAS для разрешённого MAC

В IAS откройте:

> **Connection Request Processing → Connection Request Policies**

Создайте:

```text
New Connection Request Policy
```

Укажите:

```text
Policy Name: MAC-00-11-22-33-44-55
Policy Type: A custom policy
```

Добавьте условия:

```text
NAS-Port-Type = Ethernet
Calling-Station-Id = 00-11-22-33-44-55
```

Откройте профиль политики и включите:

```text
Accept users without validating credentials
```

Разместите разрешающую политику выше общих политик.

Для каждого разрешённого MAC можно создать отдельную политику.

Не создавайте широкую политику, принимающую любой `Calling-Station-Id`.

Неизвестный MAC, не совпавший ни с одной разрешающей политикой, должен получить Access-Reject либо не пройти обработку политики.

---

## 7.3. MAC-auth на MikroTik

Откройте существующий Dot1X Server для `ether2` либо создайте новый:

> **Dot1X → Server → Add New**

Укажите:

| Поле              | Значение            |
| ----------------- | ------------------- |
| Interface         | `ether2`            |
| Auth Types        | `mac-auth`          |
| MAC Auth Mode     | `mac-as-username`   |
| RADIUS MAC Format | `XX-XX-XX-XX-XX-XX` |
| Reauth Timeout    | `30m`               |

Для запуска MAC-auth клиент должен отправить хотя бы один Ethernet-кадр. После подключения можно обновить DHCP или отправить любой трафик.

### 7.4. Комбинация 802.1X и MAC fallback

Для устройств разных типов:

```text
Auth Types: dot1x, mac-auth
```

MikroTik сначала пытается использовать 802.1X, затем переходит к MAC-auth.

Рекомендуется:

```text
Retrans Timeout: 3s
```

Иначе переход к MAC-auth может занимать слишком долго.

### 7.5. Проверка MikroTik

Откройте:

> **Dot1X → Server → Active**

Ожидается:

```text
Auth Info: mac-auth
Client MAC: 00:11:22:33:44:55
Status: authorized
```

Для неизвестного MAC:

```text
Status: rejected-holding
```

либо порт останется неавторизованным.

---

## 7.6. MAC-auth на Rapier

Добавьте IAS, если он ещё не настроен:

```text
add radius server=192.168.100.10 secret=LabRadius2026!
```

Включите MAC-аутентификацию глобально:

```text
enable portauth=macbased
```

Включите на порту:

```text
enable portauth=macbased port=2 control=auto vlanassignment=enabled securevlan=on
```

Параметры:

| Параметр                 | Назначение                                                         |
| ------------------------ | ------------------------------------------------------------------ |
| `control=auto`           | решение принимает RADIUS                                           |
| `vlanassignment=enabled` | разрешить динамическую VLAN                                        |
| `securevlan=on`          | дополнительные клиенты должны соответствовать VLAN первого клиента |

Проверка:

```text
show portauth port=2
show vlan
```

---

<a id="dynamic-vlan"></a>

## 8. Динамическое назначение VLAN

### 8.1. Задача

Пользователи разных групп должны попадать в разные VLAN:

```text
NetworkStaff → VLAN 20
NetworkGuest → VLAN 30
```

### 8.2. Создать группы IAS

В:

> **Управление компьютером → Локальные пользователи и группы**

Создайте:

```text
NetworkStaff
NetworkGuest
```

Создайте пользователей:

```text
staff1
guest1
```

Добавьте:

```text
staff1 → NetworkStaff
guest1 → NetworkGuest
```

### 8.3. Создать отдельные IAS-политики

Создайте две политики Remote Access Policy:

```text
01-Staff-VLAN20
02-Guest-VLAN30
```

Условия первой:

```text
NAS-Port-Type = Ethernet
Windows-Groups = NetworkStaff
```

Условия второй:

```text
NAS-Port-Type = Ethernet
Windows-Groups = NetworkGuest
```

Более специфичные политики разместите выше общих.

### 8.4. Добавить RADIUS-атрибуты VLAN 20

Откройте свойства политики:

> **Edit Profile → Advanced → Add**

Добавьте:

| Атрибут               | Значение              |
| --------------------- | --------------------- |
| `Tunnel-Type`         | `Virtual LANs (VLAN)` |
| `Tunnel-Medium-Type`  | `802`                 |
| `Tunnel-Pvt-Group-ID` | `20` типа String      |

Для VLAN 30 укажите:

```text
Tunnel-Pvt-Group-ID = 30
```

Для MikroTik не добавляйте `Tunnel-Tag`, если этого не требует конкретная конфигурация.

### 8.5. Подготовить VLAN на MikroTik

В:

> **Bridge → VLANs**

VLAN 20:

```text
Tagged: ether1
Untagged: пусто
```

VLAN 30:

```text
Tagged: ether1
Untagged: пусто
```

Клиентский `ether2` не добавляйте статически в эти VLAN.

После Access-Accept он появится в `Current Untagged` у назначенной VLAN.

### 8.6. Включить динамическую VLAN на MikroTik

Дополнительного переключателя не требуется, если:

* `ether2` находится в bridge;
* `VLAN Filtering=yes`;
* IAS возвращает правильные Tunnel-атрибуты;
* VLAN разрешена по trunk.

Откройте:

> **Dot1X → Server → Active**

После входа `staff1` ожидается:

```text
VLAN ID: 20
```

После входа `guest1`:

```text
VLAN ID: 30
```

### 8.7. IP после смены VLAN

После изменения VLAN клиент должен заново получить DHCP-адрес.

На Windows:

```cmd
ipconfig /release
ipconfig /renew
```

Для каждой VLAN должен существовать отдельный DHCP scope или DHCP Relay. См. [`06-dhcp`](../06-dhcp/).

---

## 8.8. Динамическая VLAN на Rapier

Создайте VLAN:

```text
create vlan=STAFF vid=20
create vlan=GUEST vid=30
```

Добавьте обе VLAN на trunk:

```text
add vlan=STAFF port=5 frame=tagged
add vlan=GUEST port=5 frame=tagged
```

Клиентский порт не добавляйте статически в STAFF или GUEST.

Включите:

```text
enable portauth=8021x
```

Настройте порт:

```text
enable portauth=8021x port=2 type=authenticator control=auto mode=single piggyback=false vlanassignment=enabled
```

После авторизации проверьте:

```text
show portauth port=2
show vlan=STAFF
show vlan=GUEST
```

Порт должен динамически оказаться в VLAN, которую вернул IAS.

---

<a id="fallback-vlans"></a>

## 9. Guest, Reject и Server-Fail VLAN

Эти VLAN решают разные задачи.

| Событие                       | MikroTik-параметр     |
| ----------------------------- | --------------------- |
| Клиент не поддерживает 802.1X | `Guest VLAN ID`       |
| RADIUS вернул Access-Reject   | `Reject VLAN ID`      |
| RADIUS не отвечает            | `Server Fail VLAN ID` |

### 9.1. Настройка MikroTik

Откройте:

> **Dot1X → Server → ether2**

Укажите:

```text
Guest VLAN ID: 997
Reject VLAN ID: 999
Server Fail VLAN ID: 998
```

Все VLAN должны:

* существовать в Bridge VLAN Table;
* быть разрешены на trunk;
* иметь DHCP или статическую адресацию;
* иметь ограничительные firewall-правила.

### 9.2. Guest VLAN

`Guest VLAN ID` используется, когда:

* включён только `dot1x`;
* клиент не отправляет EAPOL;
* fallback на `mac-auth` не настроен.

Если настроено:

```text
Auth Types: dot1x, mac-auth
```

устройство без 802.1X будет сначала проверяться по MAC, а не отправляться в Guest VLAN.

### 9.3. Reject VLAN

`Reject VLAN ID` используется только после явного:

```text
Access-Reject
```

от RADIUS-сервера.

Пример:

```text
неправильный пароль
неразрешённый пользователь
неизвестный MAC
```

### 9.4. Server-Fail VLAN

`Server Fail VLAN ID` используется, когда RADIUS-сервер не отвечает вообще.

Не путайте:

```text
Access-Reject → Reject VLAN
Нет ответа     → Server-Fail VLAN
```

### 9.5. Полное блокирование

Для максимальной безопасности не задавайте соответствующий VLAN ID.

Например, без `Server Fail VLAN ID` при отказе RADIUS клиент останется заблокированным.

### 9.6. Firewall карантинной VLAN

Пример для VLAN 999:

```text
разрешить DHCP
разрешить DNS
при необходимости разрешить портал обновления
запретить доступ к Staff
запретить доступ к Services
запретить управление оборудованием
```

Настройка firewall выполняется после создания IP-интерфейса VLAN.

---

## 9.7. Guest VLAN на Rapier

Создайте VLAN:

```text
create vlan=GUEST vid=999
```

Разрешите её на trunk:

```text
add vlan=GUEST port=5 frame=tagged
```

Настройте клиентский порт:

```text
enable portauth=8021x port=2 type=authenticator control=auto mode=single piggyback=false guestvlan=GUEST vlanassignment=enabled
```

До появления EAPOL порт может находиться в Guest VLAN.

После начала 802.1X порт удаляется из Guest VLAN и проходит обычную авторизацию.

### 9.8. RADIUS недоступен на Rapier

Для старого Rapier базовое безопасное поведение — заблокировать порт.

Для MAC-auth можно включить critical-port режим:

```text
set radius deadtime=1
enable portauth=macbased port=2 control=auto autoauthenticate=true
```

При недоступности всех RADIUS-серверов порт будет считаться авторизованным и останется в своей обычной VLAN.

Используйте `autoauthenticate=true` только при прямом требовании задания: это снижает безопасность.

Отдельная Server-Fail VLAN старой лабораторной конфигурацией не гарантируется.

---

<a id="piggybacking"></a>

## 10. Piggybacking и ограничение по MAC

### 10.1. Что такое piggybacking

Схема:

```text
один клиент успешно прошёл 802.1X
↓
порт стал авторизован
↓
через тот же порт пытается работать другое устройство
```

В port-based режиме второе устройство может использовать уже открытый порт без собственной авторизации.

Это особенно важно, если между клиентом и коммутатором подключены:

* неуправляемый коммутатор;
* концентратор;
* IP-телефон с проходным портом;
* точка доступа;
* виртуальный bridge.

---

## 10.2. MikroTik

RouterOS Dot1X Server по умолчанию авторизует интерфейс целиком.

После успешной авторизации одного клиента порт принимает трафик и от других MAC за этим портом.

### Строгое ограничение на RB1100AHx4

Для заранее известного MAC используйте статический Bridge Filter из [`01-l2-switching`](../01-l2-switching/):

Разрешающее правило:

```text
Chain: forward
In. Interface: ether2
Src. MAC Address: <CLIENT_MAC>
Action: accept
```

После него:

```text
Chain: forward
In. Interface: ether2
Action: drop
```

Для software Bridge Filter на защищаемом порту может потребоваться отключить Hardware Offload.

Такое ограничение подходит для фиксированного известного MAC, но не меняется автоматически при входе другого пользователя.

### Динамический фильтр RADIUS

RouterOS поддерживает атрибут:

```text
Mikrotik-Switching-Filter
```

для создания временных правил, например:

```text
action allow, src-mac-address none action drop
```

Однако динамические hardware switch rules поддерживаются не всеми switch chip. Для RB1100AHx4 не используйте этот способ как основной без предварительной проверки поддержки оборудования.

---

## 10.3. Rapier: разрешить piggybacking

```text
enable portauth=8021x port=2 type=authenticator mode=single piggyback=true
```

После успешной авторизации первого supplicant трафик других устройств также разрешается.

### Запретить piggybacking

```text
enable portauth=8021x port=2 type=authenticator mode=single piggyback=false
```

В этом режиме разрешён только MAC первого успешно авторизованного клиента.

### Несколько supplicant

```text
enable portauth=8021x port=2 type=authenticator mode=multi securevlan=on vlanassignment=enabled
```

Каждый клиент проходит отдельную авторизацию.

При:

```text
securevlan=on
```

следующие клиенты должны получить ту же VLAN, что и первый клиент.

Режим `mode=multi` используйте только по прямому требованию задания.

---

<a id="nps"></a>

## 11. Современный Windows Server NPS

Эта ветка используется вместо IAS на современных Windows Server.

### 11.1. Установить NPS

Откройте:

> **Server Manager → Add Roles and Features**

Добавьте:

```text
Network Policy and Access Services
└── Network Policy Server
```

### 11.2. Открыть NPS

> **Server Manager → Tools → Network Policy Server**

Если сервер входит в домен, зарегистрируйте его в Active Directory:

> **NPS (Local) → Register server in Active Directory**

### 11.3. Добавить RADIUS Client

> **RADIUS Clients and Servers → RADIUS Clients → New**

Укажите:

```text
Friendly Name: MIKROTIK
Address: 192.168.100.2
Shared Secret: LabRadius2026!
Vendor: RADIUS Standard
```

### 11.4. Создать 802.1X-политику

В главном окне NPS выберите:

```text
RADIUS server for 802.1X Wireless or Wired Connections
```

Запустите:

```text
Configure 802.1X
```

Выберите:

```text
Secure Wired (Ethernet) Connections
```

Добавьте MikroTik как RADIUS client.

Выберите группу пользователей, которым разрешён доступ.

Используйте:

```text
PEAP с EAP-MSCHAPv2
```

либо:

```text
EAP-TLS
```

Для PEAP и EAP-TLS серверу нужен подходящий сертификат.

### 11.5. Назначить VLAN через NPS

Откройте:

> **Policies → Network Policies → нужная политика → Settings → RADIUS Attributes → Standard**

Добавьте:

```text
Tunnel-Type = Virtual LANs
Tunnel-Medium-Type = 802
Tunnel-Pvt-Group-ID = 20
```

`Tunnel-Tag` добавляйте только если его требует конкретный NAS.

---

<a id="freeradius"></a>

## 12. FreeRADIUS и Linux-клиент

Резервная ветка для Debian/Ubuntu-подобной системы.

### 12.1. Установить FreeRADIUS

```bash
sudo apt update
sudo apt install freeradius freeradius-utils
```

Основной каталог обычно:

```text
/etc/freeradius/3.0/
```

### 12.2. Добавить MikroTik как RADIUS client

Откройте:

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

Добавьте:

```text
client mikrotik {
    ipaddr = 192.168.100.2
    secret = LabRadius2026!
    nas_type = other
}
```

### 12.3. Добавить пользователя 802.1X

Откройте:

```bash
sudo nano /etc/freeradius/3.0/mods-config/files/authorize
```

Добавьте в начало файла:

```text
student1 Cleartext-Password := "NetLab2026!"
    Tunnel-Type := VLAN,
    Tunnel-Medium-Type := IEEE-802,
    Tunnel-Private-Group-Id := "20"
```

### 12.4. Добавить разрешённый MAC

Настройте MikroTik:

```text
MAC Auth Mode: mac-as-username-and-password
RADIUS MAC Format: XX-XX-XX-XX-XX-XX
```

В FreeRADIUS добавьте:

```text
"00-11-22-33-44-55" Cleartext-Password := "00-11-22-33-44-55"
    Tunnel-Type := VLAN,
    Tunnel-Medium-Type := IEEE-802,
    Tunnel-Private-Group-Id := "20"
```

### 12.5. Запустить сервер

Проверить конфигурацию:

```bash
sudo freeradius -XC
```

Перезапустить:

```bash
sudo systemctl restart freeradius
sudo systemctl enable freeradius
```

Для отладочного запуска:

```bash
sudo systemctl stop freeradius
sudo freeradius -X
```

После проверки завершите debug-режим и снова запустите службу.

### 12.6. Разрешить firewall

При использовании UFW:

```bash
sudo ufw allow from 192.168.100.2 to any port 1812 proto udp
sudo ufw allow from 192.168.100.2 to any port 1813 proto udp
```

---

## 12.7. Linux как 802.1X supplicant

Установите:

```bash
sudo apt install wpasupplicant
```

Скопируйте CA-сертификат RADIUS-сервера на клиент, например:

```text
/etc/wpa_supplicant/radius-ca.pem
```

Создайте:

```bash
sudo nano /etc/wpa_supplicant/wired.conf
```

Содержимое:

```text
ctrl_interface=/run/wpa_supplicant
ap_scan=0

network={
    key_mgmt=IEEE8021X
    eap=PEAP
    identity="student1"
    password="NetLab2026!"
    ca_cert="/etc/wpa_supplicant/radius-ca.pem"
    phase2="auth=MSCHAPV2"
    eapol_flags=0
}
```

Запустите:

```bash
sudo wpa_supplicant \
  -D wired \
  -i <INTERFACE> \
  -c /etc/wpa_supplicant/wired.conf \
  -B
```

После успешной авторизации получите DHCP-адрес:

```bash
sudo dhclient <INTERFACE>
```

Не используйте отключение проверки сертификата в постоянной конфигурации.

---

<a id="cleanup"></a>

## 13. Удаление конфигурации

### 13.1. MikroTik

Удаляйте в следующем порядке:

1. отключите Dot1X Server на клиентском порту;
2. удалите Dot1X Server entry;
3. удалите RADIUS entry;
4. удалите статические Bridge Filter, созданные для MAC;
5. удалите динамические или временные VLAN-настройки;
6. удалите Guest, Reject и Server-Fail VLAN, если они больше не нужны;
7. верните клиентский порт в обычную access VLAN;
8. сохраните management-доступ.

Не удаляйте trunk и общие VLAN, если они используются другими заданиями.

### 13.2. Rapier

Отключить 802.1X на порту:

```text
disable portauth=8021x port=2
```

Отключить глобально, если других портов нет:

```text
disable portauth=8021x
```

Отключить MAC-auth:

```text
disable portauth=macbased port=2
disable portauth=macbased
```

Удалить RADIUS-сервер:

```text
delete radius server=192.168.100.10
```

Верните порт в исходную статическую VLAN.

### 13.3. Windows-клиент

Откройте свойства адаптера и отключите:

```text
IEEE 802.1X authentication
```

Остановите Wired AutoConfig, если она до лабораторной была отключена.

Верните DHCP или исходный статический IP.

### 13.4. IAS

В IAS удалите:

* созданные Remote Access Policies;
* Connection Request Policies;
* RADIUS Client;
* временные группы и пользователей.

Остановите IAS, если сервер использовался только для лаборатории.

Верните исходное значение политики:

```text
Хранить пароли с использованием обратимого шифрования
```

в состояние `Отключено`.

---

<a id="mapping"></a>

## 14. Краткое соответствие параметров

| Задача                | MikroTik                  | Rapier                   | IAS/NPS            |
| --------------------- | ------------------------- | ------------------------ | ------------------ |
| Добавить RADIUS       | `RADIUS → Add`            | `add radius server=...`  | New RADIUS Client  |
| 802.1X authenticator  | Dot1X Server              | `portauth=8021x`         | Ethernet policy    |
| MAC-auth              | `auth-types=mac-auth`     | `portauth=macbased`      | Calling-Station-Id |
| Dynamic VLAN          | Tunnel-атрибуты           | `vlanassignment=enabled` | Tunnel attributes  |
| Guest VLAN            | `guest-vlan-id`           | `guestvlan=`             | не требуется       |
| Access-Reject VLAN    | `reject-vlan-id`          | базово блокировка        | Access-Reject      |
| RADIUS fail VLAN      | `server-fail-vlan-id`     | critical port для MAC    | сервер не отвечает |
| Повторная авторизация | `reauth-timeout`          | `reauthenabled`          | обычная политика   |
| Piggybacking          | по умолчанию порт целиком | `piggyback=true`         | —                  |
| Только один MAC       | Bridge Filter             | `piggyback=false`        | —                  |
| Несколько клиентов    | ограниченно               | `mode=multi`             | отдельные сессии   |
| Состояние             | Dot1X Active/State        | `show portauth`          | IAS event log      |

---

<a id="minimal-algorithms"></a>

## 15. Минимальные алгоритмы

### 15.1. 802.1X с именем и паролем

```text
1. Настроить management-IP IAS и коммутатора.
2. Добавить коммутатор как RADIUS Client в IAS.
3. Создать пользователя и группу.
4. Создать Ethernet Remote Access Policy.
5. Выбрать EAP-MD5 для старой лаборатории.
6. Добавить IAS в RADIUS MikroTik/Rapier.
7. Включить authenticator на клиентском порту.
8. Включить 802.1X на Windows-клиенте.
9. Ввести логин и пароль.
10. Проверить состояние Authorized.
```

### 15.2. MAC-аутентификация

```text
1. Узнать MAC клиента.
2. Выбрать единый формат MAC.
3. Создать IAS policy с Calling-Station-Id.
4. Включить Accept users without validating credentials.
5. Включить mac-auth на клиентском порту.
6. Подключить разрешённое устройство.
7. Проверить Authorized.
8. Подключить неизвестное устройство.
9. Проверить блокировку или Reject VLAN.
```

### 15.3. Динамическая VLAN

```text
1. Создать рабочие VLAN.
2. Разрешить их на trunk.
3. Не добавлять клиентский порт статически.
4. Создать группы пользователей.
5. Создать отдельную RADIUS policy для каждой группы.
6. Вернуть Tunnel-Type, Tunnel-Medium-Type и Tunnel-Pvt-Group-ID.
7. Авторизовать клиента.
8. Проверить динамический VLAN ID.
9. Обновить DHCP-адрес клиента.
```

### 15.4. Карантин и отказ сервера

```text
Access-Accept → рабочая VLAN
Access-Reject → Reject VLAN
Нет EAPOL → Guest VLAN
Нет ответа RADIUS → Server-Fail VLAN
```

Для безопасной конфигурации Server-Fail VLAN можно не назначать: тогда при отказе RADIUS порт останется закрытым.
