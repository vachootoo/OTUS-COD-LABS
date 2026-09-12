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
bgp bestpath as-path multi path-relax
```
для включения ECMP используем команду:
```
 maximum-paths 4
```
Обязательно используем BFD и уменьшаем таймеры(по умолчанию 60 180):

```
timers bgp 3 9
neighbor 10.0.1.1 bfd
```

## 5. Конфигурация оборудования

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
   timers bgp 3 9
   maximum-paths 4
   neighbor 10.0.1.1 remote-as 65101
   neighbor 10.0.1.1 bfd
   neighbor 10.0.1.1 description Leaf-1
   neighbor 10.0.1.3 remote-as 65102
   neighbor 10.0.1.3 bfd
   neighbor 10.0.1.3 description Leaf-2
   neighbor 10.0.1.5 remote-as 65103
   neighbor 10.0.1.5 bfd
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
   maximum-paths 4
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
<summary>Leaf-1</summary>

```
Leaf-1(config)#sh run
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
   no switchport
   ip address 10.0.1.1/31
!
interface Ethernet2
   no switchport
   ip address 10.0.2.1/31
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
!
interface Management1
!
ip routing
!
router bgp 65101
   router-id 10.255.0.11
   timers bgp 3 9
   maximum-paths 4
   neighbor 10.0.1.0 remote-as 65000
   neighbor 10.0.1.0 bfd
   neighbor 10.0.1.0 description Spine-1
   neighbor 10.0.2.0 remote-as 65000
   neighbor 10.0.2.0 bfd
   neighbor 10.0.2.0 description Spine-2
   network 10.255.0.11/32
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
<summary>Leaf-2</summary>

```
Leaf-2(config)#    sh run
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
   no switchport
   ip address 10.0.1.3/31
!
interface Ethernet2
   no switchport
   ip address 10.0.2.3/31
!
interface Ethernet3
   no switchport
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
!
interface Management1
!
ip routing
!
router bgp 65102
   router-id 10.255.0.12
   timers bgp 3 9
   maximum-paths 4
   neighbor 10.0.1.2 remote-as 65000
   neighbor 10.0.1.2 bfd
   neighbor 10.0.1.2 description Spine-1
   neighbor 10.0.2.2 remote-as 65000
   neighbor 10.0.2.2 bfd
   neighbor 10.0.2.2 description Spine-2
   network 10.255.0.12/32
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
<summary>Leaf-3</summary>

```
Leaf-3(config)#sh run
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
   no switchport
   ip address 10.0.1.5/31
!
interface Ethernet2
   no switchport
   ip address 10.0.2.5/31
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
!
interface Management1
!
ip routing
!
router bgp 65103
   router-id 10.255.0.13
   timers bgp 3 9
   maximum-paths 4
   neighbor 10.0.1.4 remote-as 65000
   neighbor 10.0.1.4 bfd
   neighbor 10.0.1.4 description Spine-1
   neighbor 10.0.2.4 remote-as 65000
   neighbor 10.0.2.4 bfd
   neighbor 10.0.2.4 description Spine-2
   network 10.255.0.13/32
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

## 6. Проверка

Spine-1

```
Spine-1(config)# sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.255.0.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  Leaf-1                   10.0.1.1 4 65101          13692     13692    0    0 00:20:04 Estab   1      1
  Leaf-2                   10.0.1.3 4 65102          16577     16570    0    0 11:44:40 Estab   1      1
  Leaf-3                   10.0.1.5 4 65103           8965      8939    0    0 00:15:50 Estab   1      1
```
Spine-2

```
Spine-2#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.255.0.2, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  Leaf-1                   10.0.2.1 4 65101          26890     26913    0    0 02:18:57 Estab   1      1
  Leaf-2                   10.0.2.3 4 65102          27165     27178    0    0 11:14:35 Estab   1      1
  Leaf-3                   10.0.2.5 4 65103          27052     27004    0    0 05:53:20 Estab   1      1
```
**Проверим ECMP:**

Leaf-1:
```
Leaf-1(config-router-bgp)#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.255.0.11, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.255.0.1/32          10.0.1.0              0       -          100     0       65000 i
 * >      10.255.0.2/32          10.0.2.0              0       -          100     0       65000 i
 * >      10.255.0.11/32         -                     -       -          -       0       i
 * >Ec    10.255.0.12/32         10.0.1.0              0       -          100     0       65000 65102 i
 *  ec    10.255.0.12/32         10.0.2.0              0       -          100     0       65000 65102 i
 * >Ec    10.255.0.13/32         10.0.1.0              0       -          100     0       65000 65103 i
 *  ec    10.255.0.13/32         10.0.2.0
```
```
Leaf-1(config)#sh ip route

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort is not set

 C        10.0.1.0/31
           directly connected, Ethernet1
 C        10.0.2.0/31
           directly connected, Ethernet2
 B E      10.255.0.1/32 [200/0]
           via 10.0.1.0, Ethernet1
 B E      10.255.0.2/32 [200/0]
           via 10.0.2.0, Ethernet2
 C        10.255.0.11/32
           directly connected, Loopback0
 B E      10.255.0.12/32 [200/0]
           via 10.0.1.0, Ethernet1
           via 10.0.2.0, Ethernet2
 B E      10.255.0.13/32 [200/0]
           via 10.0.1.0, Ethernet1
           via 10.0.2.0, Ethernet2
```
***Проверим BFD:***

Например на  Spine-2:
```
Spine-2#sh bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.0.2.1   477553134    42040934        Ethernet1(17)  normal   09/12/26 09:43
10.0.2.3   536474471    80937991        Ethernet2(16)  normal   09/11/26 22:50
10.0.2.5  2580779609  1022682620        Ethernet3(18)  normal   09/12/26 09:47

         LastDown            LastDiag    State
-------------------- ------------------- -----
   09/12/26 07:34       No Diagnostic       Up
   09/11/26 22:50       No Diagnostic       Up
   09/12/26 04:06       No Diagnostic       Up
```
**Проверка доступности Loopback:**
```
Spine-1#ping 10.255.0.11
PING 10.255.0.11 (10.255.0.11) 72(100) bytes of data.
80 bytes from 10.255.0.11: icmp_seq=1 ttl=64 time=2.96 ms
80 bytes from 10.255.0.11: icmp_seq=2 ttl=64 time=1.71 ms
80 bytes from 10.255.0.11: icmp_seq=3 ttl=64 time=1.27 ms
80 bytes from 10.255.0.11: icmp_seq=4 ttl=64 time=1.88 ms
80 bytes from 10.255.0.11: icmp_seq=5 ttl=64 time=1.58 ms

--- 10.255.0.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 16ms
rtt min/avg/max/mdev = 1.272/1.880/2.960/0.575 ms, ipg/ewma 3.952/2.403 ms
Spine-1#ping 10.255.0.12
PING 10.255.0.12 (10.255.0.12) 72(100) bytes of data.
80 bytes from 10.255.0.12: icmp_seq=1 ttl=64 time=2.79 ms
80 bytes from 10.255.0.12: icmp_seq=2 ttl=64 time=2.24 ms
80 bytes from 10.255.0.12: icmp_seq=3 ttl=64 time=1.61 ms
80 bytes from 10.255.0.12: icmp_seq=4 ttl=64 time=1.92 ms
80 bytes from 10.255.0.12: icmp_seq=5 ttl=64 time=1.63 ms

--- 10.255.0.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 12ms
rtt min/avg/max/mdev = 1.610/2.037/2.793/0.441 ms, ipg/ewma 3.000/2.392 ms
Spine-1#ping 10.255.0.13
PING 10.255.0.13 (10.255.0.13) 72(100) bytes of data.
80 bytes from 10.255.0.13: icmp_seq=1 ttl=64 time=4.03 ms
80 bytes from 10.255.0.13: icmp_seq=2 ttl=64 time=1.28 ms
80 bytes from 10.255.0.13: icmp_seq=3 ttl=64 time=1.55 ms
80 bytes from 10.255.0.13: icmp_seq=4 ttl=64 time=1.35 ms
80 bytes from 10.255.0.13: icmp_seq=5 ttl=64 time=1.62 ms

--- 10.255.0.13 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 14ms
rtt min/avg/max/mdev = 1.281/1.966/4.028/1.038 ms, ipg/ewma 3.542/2.967 ms
```

## 7. Полезные команды для troubleshooting

```
show ip interface brief
show ip bgp summary
show ip bgp
show ip route bgp
show ip bgp neighbors
show ip bgp neighbors <neighbor-ip> received-routes
show ip bgp neighbors <neighbor-ip> advertised-routes
show ip route <loopback-ip>
show ip bgp <loopback-ip>/32
ping <remote-loopback> source <local-loopback>
traceroute <remote-loopback> source <local-loopback>
При использовании BFD:
show bfd peers
```
