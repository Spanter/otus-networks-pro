
# Реализация DHCPv6

---

## Цели

- Настроить IPv6 на маршрутизаторах
- Проверить автоматическую настройку адреса (SLAAC)
- Настроить Stateless и Stateful DHCPv6
- Реализовать DHCPv6 Relay
- Проверить связность между устройствами

---

## Топология


![На изображении схема лаборатной сети](/Labs/task3/DHCPv6/pictures/schema.PNG)


---

## Таблица адресации

| Устройство | Интерфейс | IPv6-адрес                    |
|------------|-----------|-------------------------------|
| R1         | Eth0/0    | 2001:db8:acad:2::1/64         |
| R1         | Eth0/1    | 2001:db8:acad:1::1/64         |
| R2         | Eth0/0    | 2001:db8:acad:2::2/64         |
| R2         | Eth0/1    | 2001:db8:acad:3::1/64         | 
| PC-A       | NIC       | SLAAC / Stateless DHCPv6      |
| PC-B       | NIC       | Stateful DHCPv6 (через Relay) |

---

## Шаг 1: Базовая настройка маршрутизаторов

### Делаем базовую настройку:

```bash
hostname R1
no ip domain-lookup
enable secret class
ipv6 unicast-routing
line con 0
 password cisco
 login
line vty 0 4
 password cisco
 login
service password-encryption
banner motd ^
*********************************************************************
*                                                                   * 
*   Attention! Unauthorized access to the equipment is forbidden!   *
*                                                                   *
*********************************************************************
^

```

И аналогично на R2 (`hostname R2` и т.д.)

### Настройка интерфейсов R1:

```bash
interface Eth0/0
 ipv6 address 2001:db8:acad:2::1/64
 ipv6 address fe80::1 link-local
 no shutdown

interface Eth0/1
 ipv6 address 2001:db8:acad:1::1/64
 ipv6 address fe80::1 link-local
 no shutdown
```

### Настройка интерфейсов R2:

```bash
interface Eth0/0
 ipv6 address 2001:db8:acad:2::2/64
 ipv6 address fe80::2 link-local
 no shutdown

interface Eth0/1
 ipv6 address 2001:db8:acad:3::1/64
 ipv6 address fe80::1 link-local
 no shutdown
```

### Прописываем статическую маршрутизацию:

```bash
R1(config)# ipv6 route 2001:db8:acad:3::/64 2001:db8:acad:2::2
R2(config)# ipv6 route 2001:db8:acad:1::/64 2001:db8:acad:2::1
```

---

## Шаг 2: Проверка SLAAC на PC-A

На PC-A включаем автонастройка получения адреса. Проверяем:

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/step3%20-%20sh_ip_dhcp_pool.PNG)

---

## Шаг 3: Stateless DHCPv6 на R1 -> PC-A

Создаём DHCP-пул:

```bash
ipv6 dhcp pool R1-STATELESS
 dns-server 2001:db8:acad::254
 domain-name stateless.com
```

Делаем привязку к интерфейсу:

```bash
interface Eth0/1
 ipv6 nd other-config-flag
 ipv6 dhcp server R1-STATELESS
```

---

## Шаг 4: Stateful DHCPv6 на R1 -> PC-B через R2

Создание пула:

```bash
ipv6 dhcp pool R2-STATEFUL
 address prefix 2001:db8:acad:3:aaa::/80
 dns-server 2001:db8:acad::254
 domain-name stateful.com
```

Делаем привязку на R1:

```bash
interface Eth0/0
 ipv6 dhcp server R2-STATEFUL
```

---

## Шаг 5: DHCPv6 Relay (R2 → R1)

Настраиваем на R2:

```bash
interface Eth0/1
 ipv6 nd managed-config-flag
 ipv6 dhcp relay destination 2001:db8:acad:2::1 Eth0/0
```

---

## Проверка работы


### На PC-A:

![На изображении результат вывода команды](/Labs/task3/DHCPv6/pictures/PC-A%20ipv6.PNG)

![На изображении результат вывода команды](/Labs/task3/DHCPv6/pictures/PC-A%20ping%20R1.PNG)


### На PC-B:

![На изображении результат вывода команды](/Labs/task3/DHCPv6/pictures/PC-B%20ipv6.PNG)

![На изображении результат вывода команды](/Labs/task3/DHCPv6/pictures/PC-B%20ping%20R2.PNG)

---

## Результаты

- DHCPv6 работает в трёх режимах: SLAAC, Stateless, Stateful
- Stateless передаёт DNS и доменное имя, но не адрес
- Stateful выдаёт IP-адрес полностью
- Relay позволяет централизованную выдачу IP из удалённого DHCP-сервера
- Связность между всеми сетями подтверждена пингом

Файлы конфигурации оборудования приведены [здесь](/Labs/task3/DHCPv6/config/).
---