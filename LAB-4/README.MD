# Построение Underlay сети (eBGP)

## 1. Цель работы

Настроить eBGP в качестве Underlay протокола для Leaf-Spine фабрики на оборудовании Arista EOS.
Топология состоит из:
- 2 Spine
- 3 Leaf
- каждый Leaf подключён к каждому Spine
- все соединения Spine ↔ Leaf являются L3 point-to-point
- для P2P используются сети /31
- каждый коммутатор имеет Loopback0 /32
- BGP-соседства устанавливаются непосредственно между IP-адресами P2P-интерфейсов
- через BGP распространяются Loopback0
- между Leaf используется ECMP через оба Spine

---

## 2. Адресное пространство

### Loopback адреса

Каждому устройству назначается уникальный `/32` адрес на интерфейсе `Loopback0`. Так же назначаем номер автономной системы ASN.

| Device | Interface | IP address | ASN |
|---|---|---|---|
| Spine-1 | Loopback0 | `10.255.0.1/32` | ASN 65000 |
| Spine-2 | Loopback0 | `10.255.0.2/32` | ASN 65000 |
| Leaf-1 | Loopback0 | `10.255.0.11/32` | ASN 65101 |
| Leaf-2 | Loopback0 | `10.255.0.12/32` | ASN 65102 |
| Leaf-3 | Loopback0 | `10.255.0.13/32` | ASN 65103 |

---

### Point-to-Point сети

Для соединений между Spine и Leaf используются сети `/31`.

Это позволяет использовать только два IP-адреса для каждого point-to-point соединения.

| Соединение | Network | Spine IP | Leaf IP |
|---|---|---|---|
| Spine-1 ↔ Leaf-1 | `10.0.1.0/31` | `10.0.1.0` | `10.0.1.1` |
| Spine-1 ↔ Leaf-2 | `10.0.1.2/31` | `10.0.1.2` | `10.0.1.3` |
| Spine-1 ↔ Leaf-3 | `10.0.1.4/31` | `10.0.1.4` | `10.0.1.5` |
| Spine-2 ↔ Leaf-1 | `10.0.2.0/31` | `10.0.2.0` | `10.0.2.1` |
| Spine-2 ↔ Leaf-2 | `10.0.2.2/31` | `10.0.2.2` | `10.0.2.3` |
| Spine-2 ↔ Leaf-3 | `10.0.2.4/31` | `10.0.2.4` | `10.0.2.5` |

---

## 3. Схема сети

## 4. Настройка eBGP

В отличие от OSPF или IS-IS, BGP не обнаруживает соседей автоматически.
Каждый сосед задаётся явно:

```
neighbor <IP> remote-as <ASN>
```
Через BGP будем распространять только Loopback0:
```
network 10.255.0.1/32
```
Для исключения path-hunting для Spine используются **одинаковые номера ASN**.

Если назначить:
```
Spine-1 AS65001
Spine-2 AS65002
```
то Leaf-1 получит маршрут Leaf-2 как:

через Spine-1:
```
65001 65102
```
через Spine-2:
```
65002 65102
```
AS-PATH будут отличаться.

Если использовать разные номера ASN, то для BGP ECMP тогда может потребоваться дополнительная настройка:
```
bgp bestpath as-path multipath-relax
```
## 4. Конфигурация оборудования

<details>
<summary>Spine-1</summary>

```
Spine-1#sh run
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
!
interface Ethernet2
   no switchport
   ip address 10.0.1.2/31
!
interface Ethernet3
   no switchport
   ip address 10.0.1.4/31
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
!
interface Management1
!
ip routing
!
router bgp 65000
   router-id 10.255.0.1
   neighbor 10.0.1.1 remote-as 65101
   neighbor 10.0.1.1 description Leaf-1
   neighbor 10.0.1.3 remote-as 65102
   neighbor 10.0.1.3 description Leaf-2
   neighbor 10.0.1.5 remote-as 65103
   neighbor 10.0.1.5 description Leaf-3
   network 10.255.0.1/32
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
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
!
interface Ethernet2
   no switchport
   ip address 10.0.2.2/31
!
interface Ethernet3
   no switchport
   ip address 10.0.2.4/31
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
!
interface Management1
!
ip routing
!
router bgp 65000
   router-id 10.255.0.2
   timers bgp 3 9
   neighbor 10.0.2.1 remote-as 65101
   neighbor 10.0.2.1 bfd
   neighbor 10.0.2.1 description Leaf-1
   neighbor 10.0.2.3 remote-as 65102
   neighbor 10.0.2.3 bfd
   neighbor 10.0.2.3 description Leaf-2
   neighbor 10.0.2.5 remote-as 65103
   neighbor 10.0.2.5 bfd
   neighbor 10.0.2.5 description Leaf-3
   network 10.255.0.2/32
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
end
```

</details>

<details>
<summary>Spine-1</summary>

```

```

</details>

<details>
<summary>Spine-1</summary>

```

```

</details>

<details>
<summary>Spine-1</summary>

```

```

</details>


