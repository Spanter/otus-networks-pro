# Настройка DHCPv4

## Цели

- Настроить сеть и выполнить базовую конфигурацию устройств
- Настроить DHCPv4-сервер на R1 для двух VLAN
- Настроить ретранслятор DHCP (DHCP Relay) на R2
- Проверить получение IP-адресов по DHCP на ПК

---

## Топология сети

![На изображении схема лаборатной сети](/Labs/task3/DHCPv4/pictures/schema.PNG)

## Таблица IP-адресации (пример после планирования)

| Устройство | Интерфейс  | IP-адрес     | Маска подсети   | Шлюз по умолчанию |
|------------|------------|--------------|-----------------|-------------------|
| R1         | Eth0/0     | 10.0.0.1     | 255.255.255.252 | N/A               |
| R1         | Eth0/1.100 | 192.168.1.1  | 255.255.255.192 | N/A               |
| R1         | Eth0/1.200 | 192.168.1.65 | 255.255.255.224 | N/A               |
| R2         | Eth0/0     | 10.0.0.2     | 255.255.255.252 | N/A               |
| R2         | Eth0/1     | 192.168.1.97 | 255.255.255.240 | N/A               |
| S1         | VLAN 200   | 192.168.1.66 | 255.255.255.224 | 192.168.1.65      |
| S2         | VLAN 1     | 192.168.1.98 | 255.255.255.240 | 192.168.1.97      |
| PC-A       | NIC        | DHCP         | DHCP            | DHCP              |
| PC-B       | NIC        | DHCP         | DHCP            | DHCP              |

## Таблица VLAN

| VLAN | Наименование | Назначенные интерфейсы                                         | 
|------|--------------|----------------------------------------------------------------|
| 1    | N/A          | S2: Eth4/1                                                     |
| 100  | Clients      | S1: Eth1/1                                                     |
| 200  | Management   | S1: VLAN 200                                                   |
| 999  | Parking_Lot  | S1: Eth0/0-3, Eth1/2-3, Eth2/0-3, Eth3/0-3, Eth4/0-3, Eth5/0-3 |
| 1000 | Native       | N/A                                                            |


---

## Часть 1. Сборка сети и базовая настройка

### Конфигурация R1

```bash
hostname R1
no ip domain-lookup
enable secret class
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
clock set 20:05:00 25 June 2025
interface Eth0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
interface Eth0/1
 no shutdown
interface Eth0/1.100
 description Clients
 encapsulation dot1Q 100
 ip address 192.168.1.1 255.255.255.192
interface Eth0/1.200
 description Management
 encapsulation dot1Q 200
 ip address 192.168.1.65 255.255.255.224
interface Eth0/1.1000
 encapsulation dot1Q 1000 native
```

---

### Конфигурация R2

```bash
hostname R2
no ip domain-lookup
enable secret class
...
interface Eth0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
interface Eth0/1
 ip address 192.168.1.97 255.255.255.240
 no shutdown

```

### Конфигурация статической маршрутизации на R1 и R2

#### R1

```bash
ip route 192.168.1.96 255.255.255.240 10.0.0.2
```

#### R2

```bash
ip route 0.0.0.0 0.0.0.0 10.0.0.1
```

---

## Часть 2. Настройка DHCPv4 на R1

### DHCP-исключения

```bash
ip dhcp excluded-address 192.168.1.1 192.168.1.5
ip dhcp excluded-address 192.168.30.97 192.168.30.101
```

### DHCP-пул для Subnet A

```bash
ip dhcp pool subnet_A
 network 192.168.1.0 255.255.255.192
 default-router 192.168.1.1
 domain-name ccna-lab.com
 lease 2 12 30
```

### DHCP-пул для Subnet C

```bash
ip dhcp pool R2_Client_LAN
 network 192.168.1.96 255.255.255.240
 default-router 192.168.1.97
 domain-name ccna-lab.com
 lease 2 12 30
```

### Результат проверки конфигурации DHCPv4 сервера

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/step3%20-%20sh_ip_dhcp_pool.PNG)

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/step3%20-%20sh_ip_dhcp_binding.PNG)

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/step3%20-%20sh_ip_dhcp_server_statistics.PNG)


### Результат получения IP-адреса от DHCP на PC-A

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p2-step4.PNG)

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p2-st4-ping_R1.PNG)

---

## Часть 3. Настройка DHCP Relay на R2

```bash
interface Eth0/1
 ip helper-address 10.0.0.1
```

---

## Проверка DHCP

### Получение адреса от DHCP на PC-B

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p4-s2-sh_ip.PNG)


### На R1 и R2

```bash
show ip dhcp binding
```
![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p3-step2-sh_ip_dhcp_binding.PNG)

```bash
show ip dhcp server statistics
```
![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p3-step2%20-%20sh_ip_dhcp_server_statistics(R1).PNG)

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p3-step2%20-%20sh_ip_dhcp_server_statistics(R2).PNG)


### На PC-B проверка эхо-запроса до R1

![На изображении результат вывода команды](/Labs/task3/DHCPv4/pictures/p3-step2-ping_R1.PNG)


---

## Результаты

- PC-A успешно получает IP из пула `192.168.1.0/26`
- PC-B получает IP из пула `192.168.1.96/28` через ретрансляцию
- DHCP-сервер работает на R1
- R2 успешно пересылает DHCP-запросы (Relay Agent)
- Связность между всеми хостами обеспечена

Файлы конфигурации оборудования приведены [здесь](/Labs/task3/DHCPv4/config/).

---