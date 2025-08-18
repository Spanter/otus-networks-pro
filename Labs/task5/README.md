# Отчёт по настройке PBR и маршрутизации

## Цель

-   Настроить **политику маршрутизации (PBR)** в офисе **Чокурдах**.
-   Распределить трафик между двумя линками к провайдеру (через **R25**
    и **R26**).
-   Настроить отслеживание доступности линков с помощью **IP SLA**
    (IPv4).
-   Для офиса **Лабытнанги** --- настроить маршрут по умолчанию.

---

## Топология сети

![На изображении схема лаборатной сети](/Labs/task5/pictures/schema.JPG`)

-   **AS520 (Триада):** ядро провайдера.
-   **R28 (Чокурдах):** офисный маршрутизатор, подключён двумя линками к
    провайдеру:
    -   через **R25** (10.10.1.29/30)
    -   через **R26** (10.10.1.33/30)
-   **VLAN30 (192.168.30.0/24):** клиенты (VPC30).
-   **VLAN31 (192.168.31.0/24):** клиенты (VPC31).
-   **SW29:** L2-коммутатор для локальных VLAN.
-   **R27 (Лабытнанги):** удалённый офис, подключён через R25.

---

## Настройка PBR (Policy-Based Routing) на R28

1. Создаем **ACL**, которые разделяют пользователей по VLAN:

```bash
ip access-list extended ACL-VLAN30
 permit ip 192.168.30.0 0.0.0.255 any
ip access-list extended ACL-VLAN31
 permit ip 192.168.31.0 0.0.0.255 any
```

2. Настраиваем **маршрутная карта** для PBR:

-   VLAN30 → основной аплинк через **R25**, резерв через **R26**.\
-   VLAN31 → основной аплинк через **R26**, резерв через **R25**.

```bash
route-map PBR-CHOKURDAKH permit 10
 match ip address ACL-VLAN30
 set ip next-hop verify-availability 10.10.1.29 1 track 10
 set ip next-hop verify-availability 10.10.1.33 2 track 20

route-map PBR-CHOKURDAKH permit 20
 match ip address ACL-VLAN31
 set ip next-hop verify-availability 10.10.1.33 1 track 20
 set ip next-hop verify-availability 10.10.1.29 2 track 10
```

3. Делаем PBR к интерфейсам VLAN:

```bash
interface Ethernet0/2.30
 ip policy route-map PBR-CHOKURDAKH
!
interface Ethernet0/2.31
 ip policy route-map PBR-CHOKURDAKH
```

---

## Настройка IP SLA и Track

Для проверки доступности интерфейсов к провайдеру прописываем следующую конфигурацию:

```bash
ip sla 10
 icmp-echo 10.10.1.29 source-interface Ethernet0/1
 frequency 5
ip sla schedule 10 life forever start-time now

ip sla 20
 icmp-echo 10.10.1.33 source-interface Ethernet0/0
 frequency 5
ip sla schedule 20 life forever start-time now
```

Далее создаем **track-объекты** для SLA:

```bash
track 10 ip sla 10 reachability
 delay down 10 up 5

track 20 ip sla 20 reachability
 delay down 10 up 5
```

В данной конифгурации:
   -   Проверяем выполнение каждые **5 секунд**.
   -   Настраиваем переключение на резерв -- через \~**10 секунд** после обрыва.
   -   Настраиваем возврат на основной -- через \~**10 секунд** после восстановления.

---

## Настройка маршрутов по умолчанию

Для маршрутмзатора R28 настраиваем следующий маршруты:

```bash
ip route 0.0.0.0 0.0.0.0 10.10.1.29 name ISP-to-R25 track 10
ip route 0.0.0.0 0.0.0.0 10.10.1.33 name ISP-to-R26 track 20
```

У филиала **Лабытнанги (R27)** имеется только один внешний интерфейс. Поэтому достаточно одного default:

```bash
ip route 0.0.0.0 0.0.0.0 10.10.1.25
```

---

## Проверка

1. **CEF тест маршрута для клиента из VLAN30:**

    ![На изображении вывод команды](/Labs/task5/pictures/sh_CEF_VLAN30.JPG)

    Трафик идёт через маршрутизатор R26.

2. **CEF тест маршрута для клиента из VLAN31:**

    ![На изображении вывод команды](/Labs/task5/pictures/sh_CEF_VLAN31.JPG)

    Трафик идёт через маршрутизатор R25.

3. При падении внешнего интерфейса, SLA переводит трафик на резерв в течение **\~10
    секунд**.

---

## Вывод

- PBR на R28 успешно распределяет трафик по двум интерфейсам:
  -   VLAN30 -> Основной -- R26. Резервный -- R25
  -   VLAN31 -> Основной -- R25. Резервный -- R26.
- IP SLA и track обеспечивают автоматический **фейловер и возврат**.
- Для маршрутизатора R27 (Лабытнанги) настроен маршрут по умолчанию.


Файлы конфигурации оборудования приведены [здесь](/Labs/task5/config/).

---