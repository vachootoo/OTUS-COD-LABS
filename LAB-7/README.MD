# VXLAN. Multihoming

## 1. Цель работы

Настроить EVPN VXLAN Multihoming в режиме All-Active поверх ранее построенной фабрики VXLAN EVPN с использованием Symmetric IRB.

В качестве исходной конфигурации используется существующая L3 VXLAN фабрика.

Необходимо подключить один хост одновременно к двум Leaf-коммутаторам:

Подключение хоста выполняется с использованием LACP Port-Channel.

В результате оба Leaf должны одновременно обслуживать один Ethernet Segment, обеспечивая:

- отказоустойчивое подключение хоста;

- использование обоих физических соединений;

- отсутствие MLAG peer-link;

- автоматическое переключение при отказе одного из Leaf или одного из линков.

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

### Пользовательские сети и VNI

| VLAN | L2VNI | Подсеть | Gateway |
|---|---|---|---|
| `VLAN 10` | `VNI 10010` | `172.16.10.0/24` | `172.16.10.254` |
| `VLAN 20` | `VNI 10020` | `172.16.20.0/24` | `172.16.20.254` |
| `VLAN 30` | `VNI 10030` | `172.16.30.0/24` | `172.16.30.254` |

Для VRF используется отдельный L3VNI:
```
L3VNI: 50000
```
### Хосты подключенные по LACP

| VLAN | Host | IP  | Gateway |
|---|---|---|---|
| `VLAN 10` | `Host-1` | `172.16.10.20/24` | `172.16.10.254` |
| `VLAN 20` | `Host-2` | `172.16.30.20/24` | `172.16.30.254` |

## 3. Схема сети
multihoming_scheme.png
## 4. EVPN Multihoming

В отличие от MLAG, Leaf-1 и Leaf-2 не образуют одну логическую систему.

Каждый Leaf имеет:

- собственный Loopback;

- собственный VTEP;

- собственный BGP AS;

- собственные EVPN маршруты.

Между Leaf отсутствует:
```
MLAG peer-link
```
Синхронизация выполняется через BGP EVPN control plane.

**Ethernet Segment**

это совокупность физических или логических Ethernet-соединений (линий связи), с помощью которых клиентское устройство подключается к одному или нескольким маршрутизаторам.

**ESI — Ethernet Segment Identifier**

Каждый multihomed Ethernet Segment получает уникальный идентификатор:
```
ESI
```
Например:
```
0000:0000:0000:0000:0010
```
ESI имеет размер 10 байт.

Одинаковый ESI необходимо настроить на обоих Leaf:
```
Leaf-1 = 0000:0000:0000:0000:0010
Leaf-2 = 0000:0000:0000:0000:0010
```
Другой Ethernet Segment должен получить другой ESI.

Например:
```
HOST-1
ESI 0000:0000:0000:0000:0010

HOST-2
ESI 0000:0000:0000:0000:0020
```
**ES-Import Route Target**

Для Ethernet Segment используется специальный:

ES-Import RT

00:00:00:00:00:10

Он используется EVPN Type-4 маршрутами и позволяет Leaf определить участников одного Ethernet Segment.

Значение должно совпадать на Leaf-1 и Leaf-2:
```
Leaf-1:
00:00:00:00:00:10

Leaf-2:
00:00:00:00:00:10
```
ES-Import RT не следует путать с обычным Route Target VLAN/VNI.

Например:
```
VLAN 10 RT:
65000:10010

ES-Import RT:
00:00:00:00:00:10
```
Это разные сущности.

LACP System-ID

HOST подключён к двум физически независимым коммутаторам:

На обоих Leaf назначается одинаковый:
```
LACP System-ID
```
Например:
```
0200.0000.0010
```
То есть:

02:00:00:00:00:10

На обоих Leaf:
```
Leaf-1: 0200.0000.0010
Leaf-2: 0200.0000.0010
```
Значение начинается с 02, чтобы использовать locally administered MAC address.

**EVPN Route Types**

После настройки Multihoming в EVPN control plane используются следующие типы маршрутов.

**Type-1 — Ethernet Auto-Discovery**

Type-1 сообщает остальным VTEP:

данный Ethernet Segment доступен через этот VTEP.

Это позволяет удалённому Leaf знать, что HOST доступен через оба VTEP.

Type-1 используется, в частности, для:

aliasing;

быстрого удаления путей при отказе Ethernet Segment;

построения ECMP к multihomed хостам.

**Type-2 — MAC/IP Advertisement**

Type-2 используется для распространения MAC/IP информации о хостах.

Например:
```
HOST:
IP  = 172.16.10.20
MAC = xx:xx:xx:xx:xx:xx
```
Type-2 маршрут для multihomed HOST содержит ESI:
```
MAC
IP
VNI
ESI
```
Благодаря ESI удалённый VTEP понимает, что данный MAC относится к multihomed Ethernet Segment.

**Type-3 — IMET**

Type-3 уже использовался до Multihoming.

Он используется для распространения информации, необходимой для обработки:
```
BUM
Broadcast
Unknown Unicast
Multicast
```
**Type-4 — Ethernet Segment Route**

После настройки Multihoming появляется новый EVPN маршрут:

Type-4 Ethernet Segment Route

Он используется для обнаружения Leaf, подключённых к одному Ethernet Segment.

Leaf-1 объявляет:
```
ESI: 0000:0000:0000:0000:0010
VTEP: 10.255.0.11
```
Leaf-2 объявляет:
```
ESI: 0000:0000:0000:0000:0010
VTEP: 10.255.0.12
```
После обмена Type-4 маршрутами Leaf понимают:

мы подключены к одному Ethernet Segment

Type-4 также используется для DF Election.

**Designated Forwarder**

Несмотря на режим All-Active, для Ethernet Segment выбирается:

DF — Designated Forwarder

DF особенно важен при передаче BUM-трафика из VXLAN к хостам.

Только DF передаёт соответствующий BUM-трафик в Ethernet Segment.

Это предотвращает получение HOST двух копий одного broadcast/multicast пакета.

Для обычного unicast-трафика режим остаётся All-Active.

## 5. Конфигурация устройств

<details>
<summary>Spine-1</summary>

```
Spine-1# sh run
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
   mtu 9000
   no switchport
   ip address 10.0.1.0/31   
!
interface Ethernet2
   mtu 9000
   no switchport
   ip address 10.0.1.2/31
!
interface Ethernet3
   mtu 9000
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
   maximum-paths 2
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor 10.0.1.1 remote-as 65101
   neighbor 10.0.1.1 bfd
   neighbor 10.0.1.1 description Leaf-1
   neighbor 10.0.1.3 remote-as 65102
   neighbor 10.0.1.3 bfd
   neighbor 10.0.1.3 description Leaf-2
   neighbor 10.0.1.5 remote-as 65103
   neighbor 10.0.1.5 bfd
   neighbor 10.0.1.5 description Leaf-3
   neighbor 10.255.0.11 peer group EVPN
   neighbor 10.255.0.11 remote-as 65101
   neighbor 10.255.0.12 peer group EVPN
   neighbor 10.255.0.12 remote-as 65102
   neighbor 10.255.0.13 peer group EVPN
   neighbor 10.255.0.13 remote-as 65103
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      no neighbor EVPN activate
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
   mtu 9000
   no switchport
   ip address 10.0.2.0/31
!
interface Ethernet2
   mtu 9000
   no switchport
   ip address 10.0.2.2/31
!
interface Ethernet3
   mtu 9000
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
   maximum-paths 2
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor 10.0.2.1 remote-as 65101
   neighbor 10.0.2.1 bfd
   neighbor 10.0.2.1 description Leaf-1
   neighbor 10.0.2.3 remote-as 65102
   neighbor 10.0.2.3 bfd
   neighbor 10.0.2.3 description Leaf-2
   neighbor 10.0.2.5 remote-as 65103
   neighbor 10.0.2.5 bfd
   neighbor 10.0.2.5 description Leaf-3
   neighbor 10.255.0.11 peer group EVPN
   neighbor 10.255.0.11 remote-as 65101
   neighbor 10.255.0.12 peer group EVPN
   neighbor 10.255.0.12 remote-as 65102
   neighbor 10.255.0.13 peer group EVPN
   neighbor 10.255.0.13 remote-as 65103
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      no neighbor EVPN activate
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
vlan 10
   name vxlan10010
!
vlan 20
   name vxlan10020
!
vlan 30
   name vxlan10030
!
vrf instance TENANT
!
interface Port-Channel6
   description Host-1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0010
      route-target import 00:00:00:00:00:10
   lacp system-id 0200.0000.0010
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
   description Host-1
   channel-group 6 mode active
!
interface Ethernet7
!
interface Ethernet8
   switchport access vlan 10
!
interface Loopback0
   ip address 10.255.0.11/32
!
interface Management1
!
interface Vlan10
   vrf TENANT
   ip address virtual 172.16.10.254/24
!
interface Vlan20
   vrf TENANT
   ip address virtual 172.16.20.254/24
!
interface Vlan30
   vrf TENANT
   ip address virtual 172.16.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf TENANT vni 50000
!
ip virtual-router mac-address 00:1c:73:00:00:01
!
ip routing
ip routing vrf TENANT
!
router bgp 65101
   router-id 10.255.0.11
   timers bgp 3 9
   maximum-paths 2
   neighbor SPINE_EVPN peer group
   neighbor SPINE_EVPN remote-as 65000
   neighbor SPINE_EVPN update-source Loopback0
   neighbor SPINE_EVPN ebgp-multihop 3
   neighbor SPINE_EVPN send-community extended
   neighbor 10.0.1.0 remote-as 65000
   neighbor 10.0.1.0 bfd
   neighbor 10.0.1.0 description Spine-1
   neighbor 10.0.2.0 remote-as 65000
   neighbor 10.0.2.0 bfd
   neighbor 10.0.2.0 description Spine-2
   neighbor 10.255.0.1 peer group SPINE_EVPN
   neighbor 10.255.0.2 peer group SPINE_EVPN
   !
   vlan 10
      rd 10.255.0.11:10010
      route-target both 65000:10010
      redistribute learned
   !
   vlan 20
      rd 10.255.0.11:10020
      route-target both 65000:10020
      redistribute learned
   !
   vlan 30
      rd 10.255.0.11:10030
      route-target both 65000:10030
      redistribute learned
   !
   address-family evpn
      neighbor SPINE_EVPN activate
   !
   address-family ipv4
      no neighbor SPINE_EVPN activate
      network 10.255.0.11/32
   !
   vrf TENANT
      rd 10.255.0.11:50000
      route-target import evpn 65000:50000
      route-target export evpn 65000:50000
      router-id 10.255.0.11
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
vlan 10
   name vxlan10010
!
vlan 20
   name vxlan10020
!
vlan 30
   name vxlan10030
!
vrf instance TENANT
!
interface Port-Channel5
   description Host-2
   switchport access vlan 30
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0030
      route-target import 00:00:00:00:00:30
   lacp system-id 0200.0000.0030
!
interface Port-Channel6
   description Host-1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0010
      route-target import 00:00:00:00:00:10
   lacp system-id 0200.0000.0010
!
interface Ethernet1
   mtu 9000
   no switchport
   ip address 10.0.1.3/31
!
interface Ethernet2
   mtu 9000
   no switchport
   ip address 10.0.2.3/31
!
interface Ethernet3
   no switchport
!
interface Ethernet4
!
interface Ethernet5
   description Host-2
   channel-group 5 mode active
!
interface Ethernet6
   description Host-1
   channel-group 6 mode active
!
interface Ethernet7
!
interface Ethernet8
   switchport access vlan 20
!
interface Loopback0
   ip address 10.255.0.12/32
!
interface Management1
!
interface Vlan10
   vrf TENANT
   ip address virtual 172.16.10.254/24
!
interface Vlan20
   vrf TENANT
   ip address virtual 172.16.20.254/24
!
interface Vlan30
   vrf TENANT
   ip address virtual 172.16.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf TENANT vni 50000
!
ip virtual-router mac-address 00:1c:73:00:00:01
!
ip routing
ip routing vrf TENANT
!
router bgp 65102
   router-id 10.255.0.12
   timers bgp 3 9
   maximum-paths 2
   neighbor SPINE_EVPN peer group
   neighbor SPINE_EVPN remote-as 65000
   neighbor SPINE_EVPN update-source Loopback0
   neighbor SPINE_EVPN ebgp-multihop 3
   neighbor SPINE_EVPN send-community extended
   neighbor 10.0.1.2 remote-as 65000
   neighbor 10.0.1.2 bfd
   neighbor 10.0.1.2 description Spine-1
   neighbor 10.0.2.2 remote-as 65000
   neighbor 10.0.2.2 bfd
   neighbor 10.0.2.2 description Spine-2
   neighbor 10.255.0.1 peer group SPINE_EVPN
   neighbor 10.255.0.2 peer group SPINE_EVPN
   !
   vlan 10
      rd 10.255.0.12:10010
      route-target both 65000:10010
      redistribute learned
   !
   vlan 20
      rd 10.255.0.12:10020
      route-target both 65000:10020
      redistribute learned
   !
   vlan 30
      rd 10.255.0.12:10030
      route-target both 65000:10030
      redistribute learned
   !
   address-family evpn
      neighbor SPINE_EVPN activate
   !
   address-family ipv4
      no neighbor SPINE_EVPN activate
      network 10.255.0.12/32
   !
   vrf TENANT
      rd 10.255.0.12:50000
      route-target import evpn 65000:50000
      route-target export evpn 65000:50000
      router-id 10.255.0.12
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
vlan 10
   name vxlan10010
!
vlan 20
   name vxlan10020
!
vlan 30
   name vxlan10030
!
vrf instance TENANT
!
interface Port-Channel5
   description Host-2
   switchport access vlan 30
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0030
      route-target import 00:00:00:00:00:30
   lacp system-id 0200.0000.0030
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
   description Host-1
   channel-group 5 mode active
!
interface Ethernet6
!
interface Ethernet7
   switchport access vlan 30
!
interface Ethernet8
   switchport access vlan 10
!
interface Loopback0
   ip address 10.255.0.13/32
!
interface Management1
!
interface Vlan10
   vrf TENANT
   ip address virtual 172.16.10.254/24
!
interface Vlan20
   vrf TENANT
   ip address virtual 172.16.20.254/24
!
interface Vlan30
   vrf TENANT
   ip address virtual 172.16.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf TENANT vni 50000
!
ip virtual-router mac-address 00:1c:73:00:00:01
!
ip routing
ip routing vrf TENANT
!
router bgp 65103
   router-id 10.255.0.13
   timers bgp 3 9
   maximum-paths 2
   neighbor SPINE_EVPN peer group
   neighbor SPINE_EVPN remote-as 65000
   neighbor SPINE_EVPN update-source Loopback0
   neighbor SPINE_EVPN ebgp-multihop 3
   neighbor SPINE_EVPN send-community extended
   neighbor 10.0.1.4 remote-as 65000
   neighbor 10.0.1.4 bfd
   neighbor 10.0.1.4 description Spine-1
   neighbor 10.0.2.4 remote-as 65000
   neighbor 10.0.2.4 bfd
   neighbor 10.0.2.4 description Spine-2
   neighbor 10.255.0.1 peer group SPINE_EVPN
   neighbor 10.255.0.2 peer group SPINE_EVPN
   !
   vlan 10
      rd 10.255.0.13:10010
      route-target both 65000:10010
      redistribute learned
   !
   vlan 20
      rd 10.255.0.13:10020
      route-target both 65000:10020
      redistribute learned
   !
   vlan 30
      rd 10.255.0.13:10030
      route-target both 65000:10030
      redistribute learned
   !
   address-family evpn
      neighbor SPINE_EVPN activate
   !
   address-family ipv4
      no neighbor SPINE_EVPN activate
      network 10.255.0.13/32
   !
   vrf TENANT
      rd 10.255.0.13:50000
      route-target import evpn 65000:50000
      route-target export evpn 65000:50000
      router-id 10.255.0.13
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

---

## 6. Проверка

show lacp peer
```
Leaf-1#  show lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel6:
 Et6     Bundled | 8000,50-83-7c-2e-ee-60        6   ALGs+CD    0x0038    32768
```
```
Leaf-2#show lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel5:
 Et5     Bundled | 8000,50-a3-41-3e-c2-90        3   ALGs+CD    0x0022    32768
Port Channel Port-Channel6:
 Et6     Bundled | 8000,50-83-7c-2e-ee-60        5   ALGs+CD    0x0038    32768
```
```
Leaf-3#show lacp peer
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel5:
 Et5     Bundled | 8000,50-a3-41-3e-c2-90        4   ALGs+CD    0x0022    32768
```
show bgp evpn instance vlan

```
Leaf-1#show bgp evpn instance vlan 10
EVPN instance: VLAN 10
  Route distinguisher: 10.255.0.11:10010
  Route target import: Route-Target-AS:65000:10010
  Route target export: Route-Target-AS:65000:10010
  Service interface: VLAN-based
  Local VXLAN IP address: 10.255.0.11
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0000:0010
      Type: 0 (administratively configured)
      Interface: Port-Channel6
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:00:10
      DF election algorithm: modulus
      Designated forwarder: 10.255.0.11
      Non-Designated forwarder: 10.255.0.12
```
```
Leaf-3#show bgp evpn instance vlan 30
EVPN instance: VLAN 30
  Route distinguisher: 10.255.0.13:10030
  Route target import: Route-Target-AS:65000:10030
  Route target export: Route-Target-AS:65000:10030
  Service interface: VLAN-based
  Local VXLAN IP address: 10.255.0.13
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0000:0030
      Type: 0 (administratively configured)
      Interface: Port-Channel5
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:00:30
      DF election algorithm: modulus
      Designated forwarder: 10.255.0.12
      Non-Designated forwarder: 10.255.0.13
```
show bgp evpn route-type ethernet-segment

```
Leaf-2#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.255.0.12, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.255.0.11:1 ethernet-segment 0000:0000:0000:0000:0010 10.255.0.11
                                 10.255.0.11           -       100     0       65000 65101 i
 *  ec    RD: 10.255.0.11:1 ethernet-segment 0000:0000:0000:0000:0010 10.255.0.11
                                 10.255.0.11           -       100     0       65000 65101 i
 * >      RD: 10.255.0.12:1 ethernet-segment 0000:0000:0000:0000:0010 10.255.0.12
                                 -                     -       -       0       i
 * >      RD: 10.255.0.12:1 ethernet-segment 0000:0000:0000:0000:0030 10.255.0.12
                                 -                     -       -       0       i
 * >Ec    RD: 10.255.0.13:1 ethernet-segment 0000:0000:0000:0000:0030 10.255.0.13
                                 10.255.0.13           -       100     0       65000 65103 i
 *  ec    RD: 10.255.0.13:1 ethernet-segment 0000:0000:0000:0000:0030 10.255.0.13
                                 10.255.0.13           -       100     0       65000 65103 i
```

Пингуем Host-1 с Host-2
```
host-2#ping 172.16.10.20
PING 172.16.10.20 (172.16.10.20) 72(100) bytes of data.
80 bytes from 172.16.10.20: icmp_seq=1 ttl=62 time=17.0 ms
80 bytes from 172.16.10.20: icmp_seq=2 ttl=62 time=10.4 ms
80 bytes from 172.16.10.20: icmp_seq=3 ttl=62 time=6.27 ms
80 bytes from 172.16.10.20: icmp_seq=4 ttl=62 time=6.67 ms
80 bytes from 172.16.10.20: icmp_seq=5 ttl=62 time=6.72 ms

--- 172.16.10.20 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 56ms
rtt min/avg/max/mdev = 6.265/9.404/16.955/4.063 ms, pipe 2, ipg/ewma 14.102/12.979 ms
```
Проверяем информацию по mac адресам

```
Leaf-2#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.255.0.12, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.255.0.11:10010 mac-ip 5083.7c2e.ee60
                                 10.255.0.11           -       100     0       65000 65101 i
 *  ec    RD: 10.255.0.11:10010 mac-ip 5083.7c2e.ee60
                                 10.255.0.11           -       100     0       65000 65101 i
 * >      RD: 10.255.0.12:10010 mac-ip 5083.7c2e.ee60
                                 -                     -       -       0       i
 * >Ec    RD: 10.255.0.11:10010 mac-ip 5083.7c2e.ee60 172.16.10.20
                                 10.255.0.11           -       100     0       65000 65101 i
 *  ec    RD: 10.255.0.11:10010 mac-ip 5083.7c2e.ee60 172.16.10.20
                                 10.255.0.11           -       100     0       65000 65101 i
 * >      RD: 10.255.0.12:10010 mac-ip 5083.7c2e.ee60 172.16.10.20
                                 -                     -       -       0       i
 * >Ec    RD: 10.255.0.13:10030 mac-ip 50a3.413e.c290
                                 10.255.0.13           -       100     0       65000 65103 i
 *  ec    RD: 10.255.0.13:10030 mac-ip 50a3.413e.c290
                                 10.255.0.13           -       100     0       65000 65103 i
 * >      RD: 10.255.0.12:10030 mac-ip 50a3.413e.c290 172.16.30.20
                                 -                     -       -       0       i
 * >Ec    RD: 10.255.0.13:10030 mac-ip 50a3.413e.c290 172.16.30.20
                                 10.255.0.13           -       100     0       65000 65103 i
 *  ec    RD: 10.255.0.13:10030 mac-ip 50a3.413e.c290 172.16.30.20
                                 10.255.0.13           -       100     0       65000 65103 i
```



