# Cheat Sheet: Network & System Administration (Module B)

## [cite_start]1. rtr-cod (EcoRouter) [cite: 79]
*IP: 178.207.179.4/29, AS 64500*

```bash
enable
configure terminal

! 1. [cite_start]Имя устройства [cite: 97]
hostname rtr-cod
domain-name cod.ssa2026.region

! 2. [cite_start]Внешний интерфейс (ISP) [cite: 83]
interface gigabitethernet 0/0
 description WAN_TO_ISP
 ip address 178.207.179.4 255.255.255.248
 no shutdown
exit

! 3. Внутренний интерфейс (к FW)
interface gigabitethernet 0/1
 description LAN_TO_FW
 ! Адрес транзитной сети до FW
 ip address 10.0.1.1 255.255.255.252
 no shutdown
exit

! 4. [cite_start]BGP [cite: 127-131]
router bgp 64500
 bgp router-id 178.207.179.4
 neighbor 178.207.179.1 remote-as 31133
 neighbor 178.207.179.1 description ISP_UPLINK
 
 address-family ipv4
  neighbor 178.207.179.1 activate
  ! [cite_start]Анонсируем только свой внешний блок [cite: 86]
  network 178.207.179.0 mask 255.255.255.248
 exit
exit
! Принудительное ожидание дефолта (опционально)
neighbor 178.207.179.1 default-originate

! 5. [cite_start]GRE Tunnel [cite: 123]
interface Tunnel1
 ip address 10.10.10.1 255.255.255.252
 tunnel source 178.207.179.4
 tunnel destination 178.207.179.28
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

! 6. [cite_start]OSPF [cite: 133]
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet 0/1
 network 10.10.10.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
 area 0 authentication message-digest
exit
write
