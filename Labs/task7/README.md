# Отчёт по настройке ISIS в офисе «Триада»

## Цель
- Настроить протокол маршрутизации **IS-IS** в офисе «Триада» на маршрутизаторах **R23**, **R24**, **R25** и **R26**.  
- Обеспечить маршрутизацию **одновременно для IPv4 и IPv6**.  

---

## Топология сети

![На изображении схема лаборатной сети](/Labs/task7/pictures/schema.JPG)


В данной схеме зональная принадлежность маршрутизаторов следующая:
- **R23** — зона 2222  
- **R24** — зона 24  
- **R25** — зона 2222  
- **R26** — зона 26  

---

## План адресации

### Loopback интерфейсы
Loopback интерфейсы необходимы для уникальной идентификации маршрутизаторов в сети IS-IS и как стабильные адреса для маршрутизации.  

| Узел | IPv4        | Маска           | NET (NSAP)                |
|------|-------------|-----------------|---------------------------|
| R23  | 10.128.0.23 | 255.255.255.255 | 49.2222.0101.2800.0023.00 |
| R24  | 10.128.0.24 | 255.255.255.255 | 49.0024.0101.2800.0024.00 |
| R25  | 10.128.0.25 | 255.255.255.255 | 49.2222.0101.2800.0025.00 |
| R26  | 10.128.0.26 | 255.255.255.255 | 49.0026.0101.2800.0026.00 |

---

## Пошаговая инструкция выполнения

### Выполняем базовую настройку IS-IS для всех маршрутизаторов
Для этого включаем поддержку IPv6, настраиваем протокол IS-IS, указываем, что Loopback интерфейсы должны быть пассивными (только анонсируются, соседства по ним не формируются).  

``` bash
conf t
 ipv6 unicast-routing
!
router isis TRIADA
 net 49.XXXX.XXXX.XXXX.XXXX.00
 metric-style wide             # Включение поддержки широких метрик ( обязательный для IPv6).
 log-adjacency-changes         # Позволяет видеть в логах, когда поднимается или падает соседство IS-IS.
 passive-interface Loopback0   # Loopback интерфейс не участвует в установлении соседств, но его адрес анонсируется в IS-IS
```

Настройка IS-IS на маршрутизаторе **R23**:

```bash
!
interface Ethernet0/1
 description R23-to-R25
 ip address 172.20.12.5 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
interface Ethernet0/2
 description R23-to-R24
 ip address 172.20.12.1 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
router isis TRIADA
 net 49.2222.0101.2800.0023.00
 metric-style wide
 log-adjacency-changes
 passive-interface Loopback0
```

Настройка IS-IS на маршрутизаторе **R24**: 

```bash
interface Ethernet0/1
 description R24-to-R26
 ip address 172.20.12.9 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
interface Ethernet0/2
 description R24-to-R23
 ip address 172.20.12.2 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
router isis TRIADA
 net 49.0024.0101.2800.0024.00
 metric-style wide
 log-adjacency-changes
 passive-interface Loopback0
```

Настройка IS-IS на маршрутизаторе **R25**: 

```bash
interface Ethernet0/0
 description R25-to-R23
 ip address 172.20.12.6 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
interface Ethernet0/2
 description R25-to-R26
 ip address 172.20.12.13 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
router isis TRIADA
 net 49.2222.0101.2800.0025.00
 metric-style wide
 log-adjacency-changes
 passive-interface Loopback0
```

Настройка IS-IS на маршрутизаторе **R26**: 

```bash
interface Ethernet0/0
 description R26-to-R24
 ip address 172.20.12.10 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
interface Ethernet0/2
 description R26-to-R25
 ip address 172.20.12.14 255.255.255.252
 ip router isis TRIADA
 ipv6 enable
 ipv6 router isis TRIADA
 isis network point-to-point
!
router isis TRIADA
 net 49.0026.0101.2800.0026.00
 metric-style wide
 log-adjacency-changes
 passive-interface Loopback0
```

---

## Проверка работоспособности

### Проверяем соседства IS-IS убеждаясь, что маршрутизаторы установили соседские отношения (adjacency).  

```bash
show isis neighbors
```

![На изображении вывод конфигурации](/Labs/task7/pictures/R23_sh_neighbors.JPG)

![На изображении вывод конфигурации](/Labs/task7/pictures/R25_sh_neighbors.JPG)


### Проверяем базу LSP и смотрим, какие маршрутизаторы известны в каждом сегменте.  

```bash
show isis database
```

![На изображении вывод конфигурации](/Labs/task7/pictures/R23_sh_database.JPG)

![На изображении вывод конфигурации](/Labs/task7/pictures/R25_sh_database.JPG)


### Проверяем таблицу маршрутизации


```bash
show ip route isis
```

![На изображении вывод конфигурации](/Labs/task7/pictures/R23_sh_ip_route.JPG)

![На изображении вывод конфигурации](/Labs/task7/pictures/R25_sh_ip_route.JPG)

В результате вывода видим, что маршруты loopback интерфейсов соседей появились в таблице маршрутизации через IS-IS.


## Выводы
- Настроен протокол **IS-IS** на всех четырёх маршрутизаторах.  
- Конфигурация выполнена **для IPv4 и IPv6 одновременно**.  
- Установлены соседства на интерфейсах **R23–R24** и **R25–R26**.  
- Внутри каждого сегмента маршрутизация работает корректно: loopback-адреса доступны.

Файлы конфигурации оборудования приведены [здесь](/Labs/task7/config/).
