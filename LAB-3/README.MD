# Построение Underlay сети (ISIS)

## 1. Цель работы

Настроить динамическую маршрутизацию **IS-IS** в качестве Underlay-протокола для фабрики Leaf-Spine на оборудовании Arista EOS.

Топология состоит из:

- 2 Spine-коммутаторов;
- 3 Leaf-коммутаторов;
- двух физических подключений каждого Leaf к обоим Spine;
- Loopback0 на каждом устройстве.

В качестве Underlay используется IPv4.

Основные задачи:

1. Настроить IP-адресацию Loopback-интерфейсов.
2. Настроить `/31` сети на point-to-point соединениях Spine ↔ Leaf.
3. Включить IP routing.
4. Настроить IS-IS.
5. Перевести физические Spine ↔ Leaf соединения в режим IS-IS point-to-point.
6. Анонсировать Loopback0 через IS-IS.
7. Проверить формирование IS-IS соседств.
8. Проверить доступность Loopback-интерфейсов между всеми устройствами.

---

## 2. Адресное пространство

### Loopback адреса

Каждому устройству назначается уникальный `/32` адрес на интерфейсе `Loopback0`.

| Device | Interface | IP address |
|---|---|---|
| SPINE-1 | Loopback0 | `10.255.0.1/32` |
| SPINE-2 | Loopback0 | `10.255.0.2/32` |
| LEAF-1 | Loopback0 | `10.255.0.11/32` |
| LEAF-2 | Loopback0 | `10.255.0.12/32` |
| LEAF-3 | Loopback0 | `10.255.0.13/32` |

---

### Point-to-Point сети

Для соединений между Spine и Leaf используются сети `/31`.

Это позволяет использовать только два IP-адреса для каждого point-to-point соединения.

| Соединение | Network | Spine IP | Leaf IP |
|---|---|---|---|
| SPINE-1 ↔ LEAF-1 | `10.0.1.0/31` | `10.0.1.0` | `10.0.1.1` |
| SPINE-1 ↔ LEAF-2 | `10.0.1.2/31` | `10.0.1.2` | `10.0.1.3` |
| SPINE-1 ↔ LEAF-3 | `10.0.1.4/31` | `10.0.1.4` | `10.0.1.5` |
| SPINE-2 ↔ LEAF-1 | `10.0.2.0/31` | `10.0.2.0` | `10.0.2.1` |
| SPINE-2 ↔ LEAF-2 | `10.0.2.2/31` | `10.0.2.2` | `10.0.2.3` |
| SPINE-2 ↔ LEAF-3 | `10.0.2.4/31` | `10.0.2.4` | `10.0.2.5` |

---

## 3. Схема сети

![схема сети](isis-clos-scheme.png)

Физические соединения:

```text
SPINE-1 Ethernet1 <----> Ethernet1 LEAF-1
SPINE-1 Ethernet2 <----> Ethernet1 LEAF-2
SPINE-1 Ethernet3 <----> Ethernet1 LEAF-3

SPINE-2 Ethernet1 <----> Ethernet2 LEAF-1
SPINE-2 Ethernet2 <----> Ethernet2 LEAF-2
SPINE-2 Ethernet3 <----> Ethernet2 LEAF-3
```

---

## 4. IS-IS

Для Underlay используется IS-IS.

Имя IS-IS instance:

```text
UNDERLAY
```
Для всех устройств используется одна IS-IS Area:

```text
49.0001
```
---

**NET**

В отличие от OSPF, IS-IS не использует Router ID в качестве основного идентификатора IS-IS маршрутизатора.

Каждому маршрутизатору необходимо назначить уникальный **NET — Network Entity Title**.

Формат:

```text
49.0001.xxxx.xxxx.xxxx.00
```

Где:

```text
49
```

Private AFI.

```text
0001
```

IS-IS Area ID.

```text
xxxx.xxxx.xxxx
```

Уникальный System ID маршрутизатора.

```text
00
```

NSEL.

Используем следующие NET:

| Device | NET |
|---|---|
| SPINE-1 | `49.0001.0000.0000.0001.00` |
| SPINE-2 | `49.0001.0000.0000.0002.00` |
| LEAF-1 | `49.0001.0000.0000.0011.00` |
| LEAF-2 | `49.0001.0000.0000.0012.00` |
| LEAF-3 | `49.0001.0000.0000.0013.00` |

System ID соответствует номеру устройства:

```text
SPINE-1 -> 0000.0000.0001
SPINE-2 -> 0000.0000.0002

LEAF-1  -> 0000.0000.0011
LEAF-2  -> 0000.0000.0012
LEAF-3  -> 0000.0000.0013
```

---

**IS-IS Level**

На всех устройствах используется только:

```eos
is-type level-2
```
Это позволяет использовать фабрику как единый IS-IS backbone.

---

**IS-IS Point-to-Point**

Физические соединения Spine ↔ Leaf являются point-to-point.

На интерфейсах используется:

```eos
isis network point-to-point
```

Пример:

```eos
interface Ethernet1
   no switchport
   ip address 10.0.1.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
```

---

**Loopback**

Loopback0 также необходимо добавить в IS-IS:

```eos
interface Loopback0
   ip address 10.255.0.1/32
   isis enable UNDERLAY
   isis passive
```

---

## 5. Конфигурация оборудования

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
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.0.1.2/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.0.1.4/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
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
   isis enable UNDERLAY
   isis circuit-type level-2
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0000.0000.0001.00
   is-type level-2
   !
   address-family ipv4 unicast
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
Spine-2(config)#sh run
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
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.0.2.2/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.0.2.4/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
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
   isis enable UNDERLAY
   isis circuit-type level-2
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0000.0000.0002.00
   is-type level-2
   !
   address-family ipv4 unicast
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
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.0.2.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
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
   isis enable UNDERLAY
   isis circuit-type level-2
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0000.0000.0011.00
   is-type level-2
   !
   address-family ipv4 unicast
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
Leaf-2(config)#sh run
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
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.0.2.3/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
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
   isis enable UNDERLAY
   isis circuit-type level-2
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0000.0000.0012.00
   is-type level-2
   !
   address-family ipv4 unicast
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
   description to Spine-1
   no switchport
   ip address 10.0.1.5/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.0.2.5/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis network point-to-point
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
   isis enable UNDERLAY
   isis circuit-type level-2
!
interface Management1
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0000.0000.0013.00
   is-type level-2
   !
   address-family ipv4 unicast
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

## 5. Проверка IS-IS

### Проверка соседства

```eos
show isis neighbors
```

На каждом Spine должно быть три IS-IS соседа.

Например на SPINE-1:

```text
LEAF-1
LEAF-2
LEAF-3
```

На каждом Leaf должно быть два соседа:

```text
SPINE-1
SPINE-2
```

Все adjacency должны находиться в состоянии:

```text
UP
```

и иметь тип:

```text
L2
```

---

### Проверка интерфейсов IS-IS

```eos
show isis interface
```

Физические интерфейсы должны отображаться как:

```text
Type: point-to-point
```

и работать на:

```text
Level 2
```

---

### Проверка Link State Database

```eos
show isis database
```

Подробный вывод:

```eos
show isis database detail
```

В Level-2 LSDB должны присутствовать все пять устройств:

```text
SPINE-1
SPINE-2
LEAF-1
LEAF-2
LEAF-3
```

---

### Проверка таблицы маршрутизации

```eos
show ip route
```

Маршруты, полученные через IS-IS Level-2, отображаются с кодом:

```text
I L2
```

Например на LEAF-1 должны присутствовать маршруты к Loopback:

```text
10.0.0.1/32
10.0.0.2/32
10.0.0.12/32
10.0.0.13/32
```

---

# Проверка связности

Проверяем доступность Loopback-интерфейсов.

С LEAF-1:

```eos
ping 10.0.0.1 source 10.0.0.11
ping 10.0.0.2 source 10.0.0.11
ping 10.0.0.12 source 10.0.0.11
ping 10.0.0.13 source 10.0.0.11
```

С LEAF-2:

```eos
ping 10.0.0.1 source 10.0.0.12
ping 10.0.0.2 source 10.0.0.12
ping 10.0.0.11 source 10.0.0.12
ping 10.0.0.13 source 10.0.0.12
```

---

# Итог

В результате построен IPv4 Underlay на базе IS-IS.

Используемая архитектура:

```text
IGP:               IS-IS
IS-IS Instance:    UNDERLAY
IS-IS Level:       Level-2
IS-IS Area:        49.0001

Loopback pool:      10.0.0.0/24
P2P pool:           10.0.1.0/24

Loopback prefix:    /32
P2P prefix:         /31
```

