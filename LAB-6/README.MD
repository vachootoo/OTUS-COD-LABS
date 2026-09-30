# VxLAN. EVPN L3

## 1. Цель работы

Расширить существующую VXLAN EVPN фабрику поддержкой маршрутизации между пользовательскими сетями.

Для организации L3 VXLAN будем использовать модель Symmetric IRB (Integrated Routing and Bridging).

В рамках работы необходимо:

- создать пользовательский VRF TENANT;
- добавить пользовательские VLAN;
- назначить каждому VLAN отдельный L2VNI;
- настроить Anycast Gateway;
- создать отдельный L3VNI для VRF;
- настроить распространение L2 и L3 маршрутов через MP-BGP EVPN;
- проверить маршрутизацию между VLAN, расположенными на разных Leaf;
- проверить EVPN Type-2 и Type-5 маршруты

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

## 3. Архитектура
Фабрика состоит из двух логических уровней:

```
UNDERLAY
   
   - eBGP IPv4
   - IP transport
   - ECMP
   - доставка пакетов между VTEP

OVERLAY
   
   - MP-BGP EVPN Control Plane
   - VXLAN Data Plane
```

## 4. Схема сети

![схема сети](vxlan_l2_scheme.png)

---

## 5. Настройка VxLAN. EVPN L3

При использовании Symmetric IRB маршрутизация выполняется как на ingress VTEP, так и на egress VTEP.

На всех Leaf создаём:
```
vrf instance TENANT - создаем vhf

ip routing vrf TENANT - включаем маршрутизаию в этом vrf
```
**Задаем Anycast gateway**

На всех Leaf используется одинаковый virtual-router MAC:
```
ip virtual-router mac-address 00:1c:73:00:00:01
```

Создаём SVI.

```
interface Vlan10
   description USERS_VLAN10_GATEWAY
   vrf TENANT
   ip address virtual 172.16.10.254/24
!
interface Vlan20
   description USERS_VLAN20_GATEWAY
   vrf TENANT
   ip address virtual 172.16.20.254/24
!
interface Vlan30
   description USERS_VLAN30_GATEWAY
   vrf TENANT
   ip address virtual 172.16.30.254/24
!
```
**Настройка VxLAN**

На существующем интерфейсе Vxlan1 добавляем L2VNI:
```
interface Vxlan1
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
!
```
После чего добавляем L3VNI:
```
interface Vxlan1
   vxlan vrf TENANT vni 50000
!
```
**BGP EVPN**

Добавляем в следующие параметры в bgp router на каждом Leaf:

```
router bgp 65101
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
   vrf TENANT
      rd 10.255.0.11:50000
      route-target import evpn 65000:50000
      route-target export evpn 65000:50000
      redistribute connected
!
```
**EVPN Route Type 5**

EVPN Type-5 используется для распространения IP Prefix.

Например:
```
172.16.10.0/24
172.16.20.0/24
172.16.30.0/24
```
Connected-маршруты экспортируются из VRF благодаря:
```
router bgp 65101
   vrf TENANT
      redistribute connected
```
При этом маршрут получает Route Target:
```
65000:50000
```
Удалённый VTEP проверяет RT и импортирует маршрут в:
```
VRF TENANT
```
Проверка:
```
show bgp evpn route-type ip-prefix ipv4
```
Например:
```
show bgp evpn route-type ip-prefix ipv4 172.16.10.0/24
```

Для host-to-host forwarding внутри EVPN фабрики важную роль также играют Type-2 MAC/IP routes. Type-5 предназначен именно для распространения IP-prefix маршрутов и особенно полезен при подключении внешних сетей, firewall, border leaf и других L3-сервисов.

## 6. Конфигурация устройств

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
   switchport access vlan 10
!
interface Loopback0
   ip address 10.255.0.11/32
!
interface Management1
!
interface Vlan10
   vrf TENANT
   ip address 172.16.10.254/24
!
interface Vlan20
   vrf TENANT
   ip address 172.16.20.254/24
!
interface Vlan30
   vrf TENANT
   ip address 172.16.30.254/24
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
      redistribute connected
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
!
interface Ethernet6
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
   ip address 172.16.10.254/24
!
interface Vlan20
   vrf TENANT
   ip address 172.16.20.254/24
!
interface Vlan30
   vrf TENANT
   ip address 172.16.30.254/24
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
      redistribute connected
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
Leaf-3#  sh run
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
   ip address 172.16.10.254/24
!
interface Vlan20
   vrf TENANT
   ip address 172.16.20.254/24
!
interface Vlan30
   vrf TENANT
   ip address 172.16.30.254/24
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
      redistribute connected
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

## 7. Проверка

**Underlay**
```
show ip bgp summary
show ip route
show ip route 10.255.0.11
show ip route 10.255.0.12
show ip route 10.255.0.13
```
**EVPN Control Plane**
```
show bgp evpn summary
show bgp evpn
show bgp evpn instance
show bgp evpn route-type mac-ip
show bgp evpn route-type imet
show bgp evpn route-type ip-prefix ipv4
```
**VXLAN Data Plane**
```
show interfaces Vxlan1
show vxlan vni
show vxlan address-table
show vxlan config-sanity
show vxlan config-sanity detail
```
**VRF**
```
show vrf
show ip interface brief vrf TENANT
show ip route vrf TENANT
show arp vrf TENANT
```
**VLAN**
```
show vlan
show interfaces Vlan10
show interfaces Vlan20
show interfaces Vlan30
```
**MAC**
```
show mac address-table
show mac address-table dynamic
show vxlan address-table
```
