# Построение Underlay сети (OSPF)

## 1. План работы

В качестве Underlay используется **OSPF**.

Основные параметры:

- 2 × Spine
- 3 × Leaf
- все Spine ↔ Leaf соединения работают как L3
- используется `OSPF Area 0`
- `Loopback0` используется как Router-ID
- физические интерфейсы работают как `point-to-point`
- для P2P используются сети `/31`

### Основные этапы

1. Включить маршрутизацию:

```text
ip routing
```

2. Настроить OSPF:

```text
router ospf 1
   router-id 10.255.0.1
```

3. На физических интерфейсах включить OSPF:

```text
interface Ethernet1
   no switchport
   ip address 10.0.1.0/31
   ip ospf area 0
   ip ospf network point-to-point
```

---

## 2. Адресное пространство

| Назначение | Сеть |
|---|---|
| SPINE1 P2P | `10.0.1.0/24` |
| SPINE2 P2P | `10.0.2.0/24` |
| Loopback0 | `10.255.0.0/24` |

---
**Loopback 0**

| Device | IP Address |
|---|---|
| SPINE1 | `10.255.0.1/32` |
| SPINE2 | `10.255.0.2/32` |
| LEAF1 | `10.255.0.11/32` |
| LEAF2 | `10.255.0.12/32` |
| LEAF3 | `10.255.0.13/32` |

---
**Spine-1 P2P**

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
**Spine-2 P2P**

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

## 3. Конфигурация оборудования
<details>
<summary>Spine-1</summary>

```
Spine-1(config)#sh run
! Command: show running-config
! device: Spine-1 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Spine-1
!
spanning-tree mode mstp
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   no switchport
   ip address 10.0.1.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   no switchport
   ip address 10.0.1.2/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
   no switchport
   ip address 10.0.1.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.255.0.1/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
router ospf 1
   router-id 10.255.0.1
   max-lsa 12000
!
end

```

</details>

<details>
<summary>Spine-2</summary>

```
Spine-2#sh run
! Command: show running-config
! device: Spine-2 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Spine-2
!
spanning-tree mode mstp
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   no switchport
   ip address 10.0.2.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   no switchport
   ip address 10.0.2.2/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
   no switchport
   ip address 10.0.2.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.255.0.2/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
router ospf 1
   router-id 10.255.0.2
   max-lsa 12000
!
end


```

</details>

<details>
<summary>Leaf-1</summary>

```
Leaf-1#sh run
! Command: show running-config
! device: Leaf-1 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf-1
!
spanning-tree mode mstp
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.0.1.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.0.2.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.255.0.11/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
router ospf 1
   router-id 10.255.0.11
   max-lsa 12000
!
end


```

</details>

<details>
<summary>Leaf-2</summary>

```
Leaf-2#sh run
! Command: show running-config
! device: Leaf-2 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf-2
!
spanning-tree mode mstp
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.0.1.3/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   description to Spine-2
   no switchport
   ip address 10.0.2.3/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.255.0.12/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
router ospf 1
   router-id 10.255.0.12
   max-lsa 12000
!
end


```

</details>

<details>
<summary>Leaf-3</summary>

```
Leaf-3#sh run
! Command: show running-config
! device: Leaf-3 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf-3
!
spanning-tree mode mstp
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   description to Spine-1
   no switchport
   ip address 10.0.1.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   no switchport
   ip address 10.0.2.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.255.0.13/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
router ospf 1
   router-id 10.255.0.13
   max-lsa 12000
!
end


```

</details>

## 4. Проверка

SPINE1
LEAF1    Router-ID 10.255.0.11    Ethernet1
LEAF2    Router-ID 10.255.0.12    Ethernet2
LEAF3    Router-ID 10.255.0.13    Ethernet3
SPINE2
LEAF1    Router-ID 10.255.0.11    Ethernet1
LEAF2    Router-ID 10.255.0.12    Ethernet2
LEAF3    Router-ID 10.255.0.13    Ethernet3

Каждый Leaf должен иметь двух OSPF-соседей:

SPINE1
SPINE2

Например для LEAF1:

10.255.0.1    FULL    Ethernet1
10.255.0.2    FULL    Ethernet2
