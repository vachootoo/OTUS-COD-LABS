# Построение геораспределенной сетевой фабрики на базе технологии VXLAN BGP EVPN Multi-Site

## 1. Цели

1. Построить два независимых EVPN-домена, POD-A и POD-X, с eBGP Underlay и BGP EVPN Overlay.
2. Организовать VXLAN для L2 service VLAN10 / VNI10010 и VLAN30 / VNI10030, L3 Symmetric IRB 
3. Растянуть между площадками vlan 10 и vlan 30 и не растягивать локальные VLAN20 и VLAN40.
4. Использовать для связи между сайтами четыре маршрутизируемых DCI канала **full mesh** и BGW с **Anycast Gateway** с общим VTEP на площадку.
5. Протестировать отказ BGW, отдельных DCI-линков и потерю обоих DCI-линков одного BGW.
