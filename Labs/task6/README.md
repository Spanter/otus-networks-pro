# Отчёт по настройке OSPF

## Цель
1. Настроить OSPF в офисе «Москва».
2. Разделить сеть на зоны.
3. Настроить фильтрацию маршрутов между зонами.

---

## Топология сети

![На изображении схема лаборатной сети](/Labs/task6/pictures/schema.JPG)

Выполняем разделение сети на следующие зоны:

- **Area 0 (Backbone)** - R14, R15
- **Area 10** - R12, R13
- **Area 101 (Stub)** → R19 (получает **только маршрут по умолчанию**)
- **Area 102** → R20 (получает все маршруты, **кроме сетей area 101**)

---

## Таблица адресации

### Транзитные линк-сети (IPv4)
| Устройства  | Сеть            |
|-------------|-----------------|
| R14–R19     | 172.20.10.0/30  |
| R14–R12     | 172.20.10.4/30  |
| R14–R13     | 172.20.10.8/30  |
| R15–R12     | 172.20.10.12/30 |
| R15–R13     | 172.20.10.16/30 |
| R15–R20     | 172.20.10.20/30 | 
| R12–R13     | 172.20.10.24/30 | 

### Loopback (IPv4)
| Устройство | Loopback IP | Маска           |
|------------|-------------|-----------------|
| R12        | 10.128.0.12 | 255.255.255.255 | 
| R13        | 10.128.0.13 | 255.255.255.255 |
| R14        | 10.128.0.14 | 255.255.255.255 |
| R15        | 10.128.0.15 | 255.255.255.255 |
| R19        | 10.128.0.19 | 255.255.255.255 |
| R20        | 10.128.0.20 | 255.255.255.255 |

### Пользовательские VLAN
|VLAN    | Адресация       |
|--------|-----------------|
| VLAN10 | 192.168.10.0/24 | 
| VLAN70 | 192.168.70.0/24 |  
| VLAN99 | 192.168.99.0/24 |  

---

## План работ

1) **Выполняем настройку OSPF по интерфейсам (интерфейсный стиль)**

При настройке интерфейсов, прописываем `passive-interface default` для безопасности. Затем разрешаем OSPF только на транзитных интерфейсах.

```bash
#Пример настройки R12

router ospf 1
 router-id 10.128.0.12
 passive-interface default
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2
 no passive-interface Ethernet0/3
 default-information originate
```

2) **Настраиваем маршрутизаторы R12 и R13 для зоны Area 10**
   
Прописываем **Area 10** на транзитных интерфейсах и для пользовательских интерфейсов:

```bash
router ospf 1
 router-id 10.128.0.12
 passive-interface default
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2
 no passive-interface Ethernet0/3
 default-information originate

interface Loopback0
 ip address 10.128.0.12 255.255.255.255
 ip ospf 1 area 10

 interface Ethernet0/0.10
 description Clients_VLAN_10
 ip ospf 1 area 10

interface Ethernet0/0.70
 description Clients_VLAN_70
 ip ospf 1 area 10

interface Ethernet0/0.99
 description Management
 ip ospf 1 area 10

interface Ethernet0/1
 description R12-to-R13
 ip address 172.20.10.25 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 10
```
Аналогичную настройку делаем на маршрутизаторе R13.

3) **Настраиваем маршрутизатор R19 для зоны Area 101**
  
Выполняем настройку маршрутизатора как *stub*, чтобы он видел только default route.

```bash
router ospf 1
 router-id 10.128.0.19
 area 101 stub
 passive-interface default
 no passive-interface Ethernet0/0

interface Loopback0
 ip address 10.128.0.19 255.255.255.255
 ip ospf 1 area 101

interface Ethernet0/0
 description R19-to-R14
 ip address 172.20.10.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 101
```

4) **Настраиваем маршрутизатор R20 для зоны Area 102**

Настраиваем *Area 102* на маршрутизаторе как *stub*.

```bash
router ospf 1
 router-id 10.128.0.20
 area 102 stub
 passive-interface default
 no passive-interface Ethernet0/0

interface Loopback0
 ip address 10.128.0.20 255.255.255.255
 ip ospf 1 area 102

interface Ethernet0/0
 description R20-to-R15
 ip address 172.20.10.22 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 102
```

5) **Выполняем настройку маршрутизаторов R14 и R15 (ABR)**

Выполняем подключение маршрутизатора R14 к *backbone* (area 0). На границе c *Area 10* прописываем **default-information originate**. На границе  с *Area 101 выполняем настройку как *stub no-summary*:

```bash
router ospf 1
 router-id 10.128.0.14
 area 101 stub no-summary
 passive-interface default
 no passive-interface Ethernet0/0
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/3
 default-information originate

interface Loopback0
 ip address 10.128.0.14 255.255.255.255
 ip ospf 1 area 0

interface Ethernet0/0
 description R14-to-R12
 ip address 172.20.10.5 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Ethernet0/1
 description R14-to-R13
 ip address 172.20.10.9 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Ethernet0/3
 description R14-to-R19
 ip address 172.20.10.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 101
```

Выполняем такое же подключение маршрутизатора R15 к *backbone* (area 0). На границе c *Area 10* прописываем `default-information originate`. На границе  с *Area 102* выполняем настройку *stub* и фильтрацию *prefix-list* чтобы маршрутизатор R20 не принимал сети area 101:

```bash
router ospf 1
 router-id 10.128.0.15
 area 102 stub
 area 102 filter-list prefix OSPF-101 in
 passive-interface default
 no passive-interface Ethernet0/0
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/3
 default-information originate

interface Loopback0
 ip address 10.128.0.15 255.255.255.255
 ip ospf 1 area 0
!
interface Ethernet0/0
 description R15-to-R13
 ip address 172.20.10.17 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0
!
interface Ethernet0/1
 description R15-to-R12
 ip address 172.20.10.13 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Ethernet0/3
 description R15-to-R20
 ip address 172.20.10.21 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 102

ip prefix-list OSPF-101 seq 5 deny 10.128.0.19/32
ip prefix-list OSPF-101 seq 10 permit 0.0.0.0/0 le 32
```

---

## Выполняем проверку

- `show ip ospf neighbor`

![На изображении вывод конфигурации](/Labs/task6/pictures/R14_sh_ospf_neighbors.JPG)

- `show ip route ospf`

![На изображении вывод конфигурации](/Labs/task6/pictures/R14_sh_ospf_route.JPG)

- `show ip prefix-list`  

![На изображении вывод конфигурации](/Labs/task6/pictures/R15_sh_prefix-list.JPG)

---

## Выводы

- Сеть разделена на зоны OSPF.  
- Маршрутизаторы R12 и R13 (area 10) получают дефолт.
- Маршрутизатор R19 (area 101) получает только дефолт.
- Маршрутизатор R20 (area 102) получает все маршруты кроме area 101.
- VLAN сети работают через passive-interface.

Файлы конфигурации оборудования приведены [здесь](/Labs/task6/config/).