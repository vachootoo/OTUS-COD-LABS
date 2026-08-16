# IP Адрес план

## 1. Проектирование адресного пространства

IP-план предназначен для Leaf-Spine фабрики

Топология:

- 2 × Spine
- 3 × Leaf
- L3 Point-to-Point соединения между каждым Leaf и каждым Spine
---
## 2. Топология
![Архитектура сети](clos_scheme.JPG)

## 3. Адресное пространство

Используется адресное пространство:

```text
10.0.0.0/8
```

| Назначение | Подсеть |
|---|---|
| Spine-1 P2P | `10.0.1.0/24` |
| Spine-2 P2P | `10.0.2.0/24` |
| Loopback0 | `10.255.0.0/24` |

Loopback

| Device | Interface | IP Address |
|---|---|---|
| Spine-1 | Loopback0 | `10.255.0.1/32` | 
| Spine-2 | Loopback0 | `10.255.0.2/32` | 
| Leaf-1 | Loopback0 | `10.255.0.11/32` | 
| Leaf-2 | Loopback0 | `10.255.0.12/32` | 
| Leaf-3 | Loopback0 | `10.255.0.13/32` | 


Сети Point-to-Point

Для соединений Spine ↔ Leaf используются сети `/31`.

Spine-1 P2P

Адресный пул:

```text
10.0.1.0/24
```

| Link | Network | Spine IP | Leaf IP |
|---|---|---|---|
| Spine-1 ↔ Leaf-1 | `10.0.1.0/31` | `10.0.1.0` | `10.0.1.1` |
| Spine-1 ↔ Leaf-2 | `10.0.1.2/31` | `10.0.1.2` | `10.0.1.3` |
| Spine-1 ↔ Leaf-3 | `10.0.1.4/31` | `10.0.1.4` | `10.0.1.5` |

```text
10.0.1.0/31
├── 10.0.1.0  Spine-1
└── 10.0.1.1  Leaf-1

10.0.1.2/31
├── 10.0.1.2  Spine-1
└── 10.0.1.3  Leaf-2

10.0.1.4/31
├── 10.0.1.4  Spine-1
└── 10.0.1.5  Leaf-3
```
Spine-2 P2P

Адресный пул:

```text
10.0.2.0/24
```

| Link | Network | Spine IP | Leaf IP |
|---|---|---|---|
| Spine-2 ↔ Leaf-1 | `10.0.2.0/31` | `10.0.2.0` | `10.0.2.1` |
| Spine-2 ↔ Leaf-2 | `10.0.2.2/31` | `10.0.2.2` | `10.0.2.3` |
| Spine-2 ↔ Leaf-3 | `10.0.2.4/31` | `10.0.2.4` | `10.0.2.5` |

```text
10.0.2.0/31
├── 10.0.2.0  Spine-2
└── 10.0.2.1  Leaf-1

10.0.2.2/31
├── 10.0.2.2  Spine-2
└── 10.0.2.3  Leaf-2

10.0.2.4/31
├── 10.0.2.4  Spine-2
└── 10.0.2.5  Leaf-3
```
## 3. Проверка
```
Spine-1#ping 10.0.1.1
PING 10.0.1.1 (10.0.1.1) 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=12.5 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=2.68 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=1.98 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=1.81 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=2.06 ms

--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 43ms
rtt min/avg/max/mdev = 1.814/4.210/12.514/4.162 ms, pipe 2, ipg/ewma 10.639/8.205 ms
```
