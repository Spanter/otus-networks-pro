# Настройка и маршрутизация между VLAN по технологии Router-on-a-Stick

---


## Описание работы

В данной лабораторной работе выполняется пошаговая настройка оборудования:
- создание VLAN'ов и привязка портов к ним;
- настройка транковых соединений;
- конфигурация подинтерфейсов маршрутизатора;
- проверка работоспособности маршрутизации между VLAN

---

## Цель работы

1. Сконфигурировать базовые настройки устройств.
2. Создать VLAN'ы. Назначить им порты на коммутаторах.
3. Настроить транки между оборудованиями.
4. Настроить маршрутизацию между VLAN на маршрутизаторе.
5. Проверить работоспособность сети.

---

## Топология сети

![На изображении схема лаборатной сети](/Labs/task1/Schema.PNG)

---

Таблица адресации

| Устройство | Интерфейс | IP-адрес | Маска  подсети | Шлюз по умолчанию |
|------------|------------|------------|------------|------------|
| R1 | e0/0.3 | 192.168.3.1 | 255.255.255.0 | - |
| R1 | e0/0.4 | 192.168.4.1 | 255.255.255.0 | - |
| R1 | e0/0.8 | - | - | - |
| S1 | VLAN3 | 192.168.3.11 | 255.255.255.0 | 192.168.3.1 |
| S2 | VLAN3 | 192.168.3.12 | 255.255.255.0 | 192.168.3.1 |
| PC-A | NIC | 192.168.3.3 | 255.255.255.0 | 192.168.3.1 |
| PC-B | NIC | 192.168.4.3 | 255.255.255.0 | 192.168.4.1 |

---

## Часть 1. Настройка базовых параметров

### Маршрутизатор R1

```bash
enable
configure terminal
hostname R1
no ip domain-lookup
enable secret class
line console 0
 password cisco
 login
exit
line vty 0 4
 password cisco
 login
exit
service password-encryption
banner motd # 
*********************************************************************
***                                                               *** 
*** Attention! Unauthorized access to the equipment is forbidden! ***
***                                                               ***
*********************************************************************
 #
clock set 15:00:00 8 June 2025
end
copy running-config startup-config
```

---

### Коммутатор S1

```bash
enable
configure terminal
hostname S1
no ip domain-lookup
enable secret class
line console 0
 password cisco
 login
exit
line vty 0 4
 password cisco
 login
exit
service password-encryption
banner motd # 
*********************************************************************
***                                                               *** 
*** Attention! Unauthorized access to the equipment is forbidden! ***
***                                                               ***
*********************************************************************
 #
clock set 15:05:00 8 June 2025
end
write memory
```

*(Аналогичная настройка для коммутатора S2)*

---

## Часть 2. Создание VLAN и назначение портов

### На S1 и S2

```bash
vlan 3
 name Management
vlan 4
 name Operations
vlan 7
 name ParkingLot
vlan 8
 name Native
```

#### Настройка интерфейса управления:
На S1:
```bash
interface vlan 3
 ip address 192.168.3.11 255.255.255.0
 no shutdown
ip default-gateway 192.168.3.1
```

На S2
```bash
interface vlan 3
 ip address 192.168.3.12 255.255.255.0
 no shutdown
ip default-gateway 192.168.3.1
```

#### Назначение портов:
НА S1
```bash
interface range eth0/1 - 3, eth1/2 - 3, eth2/0 - 3, eth3/0 - 3, eth4/0 - 3, eth5/0 - 3
 switchport mode access
 switchport access vlan 7

interface eth1/1
 switchport mode access
 switchport access vlan 3
```
![На изображении результат вывод таблицы vlan S1](/Labs/task1/S1-vl-br.PNG)

На S2
```bash
interface range eth0/1 - 3, eth1/0 - 3, eth2/0 - 3, eth3/0 - 3, eth4/2 - 3, eth5/0 - 3
 switchport mode access
 switchport access vlan 7

interface eth4/1
 switchport mode access
 switchport access vlan 4
```
![На изображении результат вывод таблицы vlan S2](/Labs/task1/S2-vl-br.PNG)

---

## Часть 3. Настройка транков

### На S1:
```bash
interface eth0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 8
 switchport trunk allowed vlan 3,4,8

interface f1/5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 8
 switchport trunk allowed vlan 3,4,8
```
### На S2:
```bash
interface f0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 8
 switchport trunk allowed vlan 3,4,8
```

---

## Часть 4. Настройка маршрутизации между VLAN на R1

```bash
interface eth0/0
 no shutdown

interface eth0/0.3
 description Management VLAN
 encapsulation dot1Q 3
 ip address 192.168.3.1 255.255.255.0

interface eth0/0.4
 description Operations VLAN
 encapsulation dot1Q 4
 ip address 192.168.4.1 255.255.255.0

interface eth0/0.8
 description Native VLAN
 encapsulation dot1Q 8 native
```

---

## Часть 5. Проверка маршрутизации со стороны PC

### PC-A:


![На изображении результат ping до шлюза](/Labs/task1/PC-A_ping_gateway.PNG)

![На изображении результат ping до хоста PC-B](/Labs/task1/PC-A_ping_PC-B.PNG)

![На изображении результат ping до коммутатора S2](/Labs/task1/PC-A_ping_S2.PNG)


### PC-B:

![На изображении результат маршрута до хоста PC-A](/Labs/task1/PC-B_trace_PC-A.PNG)

----

Файлы конфигурации оборудования приведены [здесь](/Labs/task1/config/).