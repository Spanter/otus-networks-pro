# Отчёт по настройке iBGP

## Цель
- Настроить iBGP в офисе Москва (*R14*, *R15*).  
- Настроить iBGP в сети провайдера Триада с использованием Route Reflector.
- Настройка приоритета провайдера Ламас для офиса Москва
- Настройка балансировки трафика в Санкт-Петербурге (*R18*)
- Обеспечить полную IP-связность всех сетей лабораторной топологии.  

---

## Топология сети

![Схема лабораторной сети](/Labs/task10/pictures/schema.JPG)

### Автономные системы (AS)
| Узел            | AS   | Устройство |
|-----------------|------|------------|
| Москва          | 1001 | R14, R15   |
| Санкт-Петербург | 2042 | R18        |
| Киторн          | 101  | R22        |
| Ламас           | 301  | R21        |
| Триада          | 520  | R24, R26   |

### Loopback-интерфейсы (/32)
| Узел                  | Loopback IP |
|-----------------------|-------------|
| R14 (Москва)          | 10.128.0.14 |
| R15 (Москва)          | 10.128.0.15 |
| R18 (Санкт-Петербург) | 10.128.0.18 |
| R21 (Ламас)           | 10.128.0.21 |
| R23 (Триада)          | 10.128.0.23 |
| R24 (Триада)          | 10.128.0.24 |
| R25 (Триада)          | 10.128.0.25 |
| R26 (Триада)          | 10.128.0.26 |

---

## Пошаговое выполнение

### Настройка iBGP в офисе Москва (R14 и R15)

Для стабильности, создаём iBGP-сессии между *R14* и *R15* через loopback-интерфейсы. Далее Анонсируем свои loopback и P2P-сети.

**R14**
```bash
router bgp 1001
 bgp router-id 10.128.0.14
 bgp log-neighbor-changes
 network 10.128.0.14 mask 255.255.255.255
 network 10.10.1.0 mask 255.255.255.252
 neighbor 10.128.0.15 remote-as 1001
 neighbor 10.128.0.15 update-source Loopback0
```

**R15**
```bash
router bgp 1001
 bgp router-id 10.128.0.15
 bgp log-neighbor-changes
 network 10.128.0.15 mask 255.255.255.255
 network 10.10.1.4 mask 255.255.255.252
 neighbor 10.128.0.14 remote-as 1001
 neighbor 10.128.0.14 description R14
 neighbor 10.128.0.14 update-source Loopback0
```

Выполняем проверку:

![На изображении вывод конфигурации](/Labs/task10/pictures/R14_ip_bgp_summ.JPG)

По выводу мы видим, что состояние указано `Established`.  

---

### Настройка iBGP в провайдере Триада с использованием Route Reflector

На данной схеме *R23* выполняет роль RR. *R24*, *R25* и *R26* является клиентом iBGP.  

Настраиваем RR на *R23*:

**R23 (RR)**
```bash
router bgp 520
 bgp router-id 10.128.0.23
 bgp log-neighbor-changes
 redistribute connected
 neighbor 10.128.0.24 remote-as 520
 neighbor 10.128.0.24 update-source Loopback0
 neighbor 10.128.0.24 route-reflector-client
 neighbor 10.128.0.24 next-hop-self
 neighbor 10.128.0.25 remote-as 520
 neighbor 10.128.0.25 update-source Loopback0
 neighbor 10.128.0.25 route-reflector-client
 neighbor 10.128.0.25 next-hop-self
 neighbor 10.128.0.26 remote-as 520
 neighbor 10.128.0.26 update-source Loopback0
 neighbor 10.128.0.26 route-reflector-client
 neighbor 10.128.0.26 next-hop-self
```

**R24 (iBGP-клиент)**
```bash
router bgp 520
 bgp router-id 10.128.0.24
 bgp log-neighbor-changes
 redistribute connected
 neighbor 10.128.0.23 remote-as 520
 neighbor 10.128.0.23 update-source Loopback0
```
*Аналогичную настройку и делаем для маршрутизаторах R25 и R26.*


Выполняем проверку маршрутов iBGP:

![На изображении вывод конфигурации](/Labs/task10/pictures/R23_sh_ip_bgp.JPG)

![На изображении вывод конфигурации](/Labs/task10/pictures/R23_ip_route_bgp.JPG)

---

### Настройка приоритета провайдера Ламас для офиса Москва

В офисе Москва *(R15)* настраиваем **local-preference 200** для маршрутов через Ламас *(R21)*:  

```bash
route-map PREFER-LAMAS permit 10
 set local-preference 200
```

```bash
router bgp 1001
 neighbor 10.10.1.6 remote-as 301
 neighbor 10.10.1.6 description To_Lamas
 neighbor 10.10.1.6 route-map PREFER-LAMAS in
```

Остальные провайдеры имеют стандартное значение **local-preference 100**, поэтому маршруты через Ламас будут предпочтительны.  

---

### Настройка балансировки трафика в Санкт-Петербурге (R18)

Для балансировки трафика маршрутов до офисов, включаем **BGP мультипуть**, чтобы трафик распределялся по двум линкам одновременно:

```bash
router bgp 2042
 bgp router-id 10.128.0.18
 bgp log-neighbor-changes
 bgp bestpath as-path multipath-relax
 network 10.128.0.18 mask 255.255.255.255
 network 10.10.1.8 mask 255.255.255.252
 network 10.10.1.12 mask 255.255.255.252
 neighbor 10.10.1.10 remote-as 520
 neighbor 10.10.1.14 remote-as 520
 maximum-paths 2
```

Данная конфигурация позволяет распределять трафик до любого офиса по двум линкам одновременно.

---

### Проверка IP-связности
Для обспечения IP-связности между различными сетями, включаем редистрибьюцию маршрутов между различными протоколами маршрутизации.

Для настройки офиса Москва на *R14* и *R15* прописывем:

```bash
router bgp 1001
 redistribute ospf 1 route-map OSPF-TO-BGP

ip prefix-list MOSCOW-TO-BGP seq 5 permit 10.128.0.0/24 le 32
ip prefix-list MOSCOW-TO-BGP seq 10 permit 10.128.1.0/24 le 32
ip prefix-list MOSCOW-TO-BGP seq 15 permit 192.168.10.0/24
ip prefix-list MOSCOW-TO-BGP seq 20 permit 192.168.70.0/24
!
route-map OSPF-TO-BGP permit 10
 match ip address prefix-list MOSCOW-TO-BGP
```

Для настройки офиса Санкт-Петерербург на *R18* прописывем:

```bash
router bgp 2042
 redistribute eigrp 2042 route-map EIGRP-TO-BGP

ip prefix-list SPB-TO-BGP seq 5 permit 10.128.0.0/24 le 32
ip prefix-list SPB-TO-BGP seq 10 permit 192.168.80.0/24
ip prefix-list SPB-TO-BGP seq 15 permit 192.168.100.0/24
!
route-map EIGRP-TO-BGP permit 10
 match ip address prefix-list SPB-TO-BGP
```

1. Проверяем доступность всех loopback и P2P интерфейсов:

![На изображении вывод конфигурации](/Labs/task10/pictures/R14_ping_R15.JPG)

![На изображении вывод конфигурации](/Labs/task10/pictures/R14_ping_R27.JPG)

![На изображении вывод конфигурации](/Labs/task10/pictures/R15_ping_R18.JPG)


2. Проверяем доступность до офисов:

![На изображении вывод конфигурации](/Labs/task10/pictures/VPC1_ping_VPC8.JPG)

![На изображении вывод конфигурации](/Labs/task10/pictures/VPC1_trace_VPC8.JPG)

Все сети имеют полную IP-связность, маршруты проходят корректно.  

---

## Выводы

- Настроен iBGP между маршрутизаторами офиса Москва (*R14* и *R15*).  
- Настроен iBGP в провайдере Триада с использованием **Route Reflector** (*R24*).  
- Настроен приоритет провайдера Ламас для офиса Москва через **local-preference**.  
- В офисе Санкт-Петербург (*R18*) организована балансировка трафика по двум линкам с использованием мультипуть BGP.  
- Полная IP-связность всех сетей подтверждена проверкой ping и traceroute.  

Файлы конфигурации оборудования приведены [здесь](/Labs/task10/config/).