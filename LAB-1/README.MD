# IP Addressing Plan

## 1. Проектирование адресного пространства

IP-план предназначен для Leaf-Spine фабрики

Топология:

- 2 × Spine
- 3 × Leaf
- L3 Point-to-Point соединения между каждым Leaf и каждым Spine
---
## 2. Topology

## 3. Адресное пространство

Используется адресное пространство:

```text
10.0.0.0/8
```

| Назначение | Подсеть |
|---|---|
| Spine1 P2P | `10.0.1.0/24` |
| Spine2 P2P | `10.0.2.0/24` |
| Loopback0 | `10.255.0.0/24` |

| Device | Interface | IP Address | ASN |
|---|---|---|---:|
| SPINE1 | Loopback0 | `10.255.0.1/32` | 
| SPINE2 | Loopback0 | `10.255.0.2/32` | 
| LEAF1 | Loopback0 | `10.255.0.11/32` | 
| LEAF2 | Loopback0 | `10.255.0.12/32` | 
| LEAF3 | Loopback0 | `10.255.0.13/32` | 
