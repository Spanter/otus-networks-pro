# Отчёт по настройки основные протоколы сети Интернет

## Цель
- Настроить **DHCP** в офисе **Москва**  
- Настроить синхронизацию времени в офисе **Москва**  
- Настроить **NAT** в офисах **Москва**, **Санкт-Петербург** и **Чокурдах** 

---

## Топология сети

![На изображении схема лабораторной сети](/Labs/task12/pictures/schema.JPG) 

---

## Пошаговая настройка

### 1. Настройка NAT(PAT) на R14 и R15.

Настраиваем NAT (PAT) на интерфейсах, выходящих в автономную систему *AS1001*. Для начала определяем внутренние и внешние интерфейсы.

```bash
interface Ethernet0/0
 description R14-to-R12
 ip nat inside

interface Ethernet0/1
 description R14-to-R13
 ip nat inside

interface Ethernet0/2
 description R14-to-R22
 ip nat outside

interface Ethernet0/3
 description R14-to-R19
 ip nat inside
```


Для трансляции внутренних сетей на маршрутизаторах, используем список доступа которые собираемся транслировать.

```bash
access-list 1 permit 192.168.0.0 0.0.255.255
access-list 1 permit 172.20.10.0 0.0.0.255
```

Преобразование адреса в публичный выполняем через внешний интерфейс с перегрузкой(*overload*).

```bash
ip nat inside source list 1 interface Ethernet0/2 overload
```

**Результат:** 

Для проверки выполняем пинг c маршрутизатора *R12* и проверяем трансляцию и статистику на *R14*

![На изображении вывод конфигурации](/Labs/task12/pictures/R12_ping.JPG) 

![На изображении вывод конфигурации](/Labs/task12/pictures/R14_NAT.JPG)

*Аналогичную настройку и проверку делаем для маршрутизатора R15.*

---

### 2. Настройка NAT(PAT) на R18

Определяем внутренние и внешние интерфейсы на маршрутизаторе *R18*.

```bash
interface Ethernet0/0
 description R18-to-R16
 ip nat inside

interface Ethernet0/1
 description R18-to-R17
 ip nat inside

interface Ethernet0/2
 description R18-to-R24
 ip nat outside

interface Ethernet0/3
 description R18-to-R26
 ip nat outside
```

Настраиваем NAT через пул из 5 адресов (*10.10.1.200-10.10.1.204*)

```bash
ip nat pool AS2042-POOL 10.10.1.200 10.10.1.204 netmask 255.255.255.248
```
Для трансляции внутренних сетей на маршрутизаторах, используем список доступа которые собираемся транслировать.

```bash
ip access-list standard NAT-INSIDE-R18
 permit 192.168.80.0 0.0.0.255
 permit 192.168.100.0 0.0.0.255
 permit 172.20.11.0 0.0.0.255
```

Преобразование адреса в публичный выполняем через внешний интерфейс с перегрузкой(*overload*).

```bash
ip nat inside source list NAT-INSIDE-R18 pool AS2042-POOL overload
```

**Результат:**

Для проверки выполняем пинг c маршрутизаторов *R16*, *R17* и *R32*. Проверяем трансляцию и статистику на *R18*

![На изображении вывод конфигурации](/Labs/task12/pictures/R18_NAT.JPG) 

---

### 3. Настройка статический NAT для R20

Настроим на R15 статическую трансляцию, чтобы R20 был доступен по внешнему IP адресу. Добавляем ещё один IP-адрес на исходящий интерфейс, который будет использовать для статического NAT:

```bash
interface Ethernet0/2
 description R15-toR21
 ip address 10.10.1.150 255.255.255.252 secondary
```

Прописываем исключение адреса *R20* из списка адресов для NAT:

```bash
access-list 1 deny   172.20.10.22
access-list 1 permit 192.168.0.0 0.0.255.255
access-list 1 permit 172.20.10.0 0.0.0.25
```

Прописываем статическую трансляцию на *R15*:

```bash
ip nat inside source static 172.20.10.22 10.10.1.150
```

**Результат:**

Для проверки выполняем пинг c маршрутизаторов *R20*. Проверяем трансляцию на *R15*

![На изображении вывод конфигурации](/Labs/task12/pictures/R15_NAT_tr.JPG)

---

### 4. Настройка NAT для удалённого управления R19

Поскольку на маршрутизаторах *R14* и *R15* уже настроен NAT и доступны по интерфейсу Loopback, то на *R19* прописываем настройки telnet.

```bash
line vty 0 4
 password 7 0822455D0A16
 no login
 transport input telnet
```

**Результат:**

Для проверки подключимся с *R28* на *R19* по telnet:

![На изображении вывод конфигурации](/Labs/task12/pictures/R19_telnet.JPG)

---

### 5. Настройка статического NAT(PAT) в офисе Чокурдах (R28)

В филиале Чокурдах настраиваем статический NAT для двух VLAN (*VLAN30 и VLAN31*). В качестве внешнего адреса используем сети *10.10.2.0/24* и*10.10.3.0/24*

```bash
ip nat inside source static network 192.168.30.0 10.10.2.0 /24
ip nat inside source static network 192.168.31.0 10.10.3.0 /24
```

Далее определяем внутренние и внешние интерфейсы на маршрутизаторе *R28*.

```bash
interface Ethernet0/0
 description R28-to-R26
 ip nat outside

interface Ethernet0/1
 description R28-to-R25
 ip nat outside

interface Ethernet0/2.30
 description Clients1
 ip nat inside

interface Ethernet0/2.31
 description Clients2
 ip nat inside
```

**Результат:**

Для проверки выполняем проверку трансляции адресов и статистику на *R28*

![На изображении вывод конфигурации](/Labs/task12/pictures/R28_NAT.JPG)

---

### 6. Настройка IPv4 DHCP-сервера в офисе Москва на маршрутизаторах R12 и R13

На маршрутизаторах *R12* и *R13* настраиваем DHCP-сервис для *VLAN10* и *VLAN70*. Для этого создаем и настраиваем два пула. В пуле указываем шлюз и DNS-сервер:

```bash
ip dhcp pool Clients-VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 lease 7

ip dhcp pool Clinets-VLAN70
 network 192.168.70.0 255.255.255.0
 default-router 192.168.70.1
 dns-server 8.8.8.8
 lease 7
```

Поскольку на *R12* и *R13* настроен HSRP, резервируем первые адреса для сетевых устройств:

```bash
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.70.1 192.168.70.10
```

**Результат:** 

На хостах *VPC1* и *VPC7* проверяем получение IP-адреса автоматически

![На изображении вывод конфигурации](/Labs/task12/pictures/VPC1_DHCP.JPG)

![На изображении вывод конфигурации](/Labs/task12/pictures/VPC7_DHCP.JPG)

---

### 7. Настройка NTP сервер на R12 и R13

Для синхронизации сетевых устройств в офисе Москва, настроим NTP сервер на маршрутизаторах *R12* и *R13*. . 

На *R12* настраиваем основной NTP-сервер (*ntp master 1*):

```bash
ntp source Loopback0
ntp master 1
ntp update-calendar
```

На *R13* настраиваем резервный NTP-сервер (*ntp master 2*) и настраиваем синхронизацию от *R12*:

```bash
ntp source Loopback0
ntp master 2
ntp update-calendar
ntp server 10.128.0.12 prefer
```

На остальных устройствах прописываем основной и резервный NTP-серверва. Выставляем предпочтение синхронизацию с *R12*:

```bash
ntp server 10.128.0.12 prefer
ntp server 10.128.0.13
```

**Результат:** 

Все устройства офиса синхронизированы по времени с *R12* и *R13*. При отказе *R12* время продолжает корректироваться с *R13*

![На изображении вывод конфигурации](/Labs/task12/pictures/R12_ntp_srv.JPG)

![На изображении вывод конфигурации](/Labs/task12/pictures/R13_ntp_srv.JPG)

![На изображении вывод конфигурации](/Labs/task12/pictures/R19_ntp_cl.JPG)

---

### 8. Проверка IP-связности

Убеждаемся что все офисы имеют IP-связность и что NAT не блокирует обмен

![На изображении вывод конфигурации](/Labs/task12/pictures/VPC_ping.JPG)

![На изображении вывод конфигурации](/Labs/task12/pictures/VPC8_ping.JPG)

![На изображении вывод конфигурации](/Labs/task12/pictures/VPC31_ping.JPG)

## Выводы

- Реализована трансляция адресов в офисах (NAT/PAT).
- Настроена автоматическая выдача адресов (DHCP) в офисе Москва.
- Реализована синхронизация времени (NTP) в офисе Москва.
- Обеспечена полная IP-связность между офисами.

Файлы конфигурации оборудования приведены [здесь](/Labs/task12/config/).