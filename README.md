![](/1.png)
![](/2.png)
![](/3.png)

```
Название устройства    ОС
rtr-cod	               EcoRouterOS (7-jasmine)
fw-cod                 Ideco NGFW NOVUM 21
sw1-cod	               Альт Сервер 11
sw2-cod	               Альт Сервер 11
cli-cod	               Альт Рабочая станция 11
srv1-cod               Альт Сервер 11
srv2-cod               Альт Сервер 11
sip-cod	               SNG7-PBX16
admin-cod              Альт Рабочая станция 11
rtr-a                  EcoRouterOS (7-jasmine)
sw1-a                  Альт Сервер 11
sw2-a                  Альт Сервер 11
dc-a                   Альт Сервер 11
cli1-a                 Альт Рабочая станция 11
cli2-a                 Альт Рабочая станция 11
```

```bash
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable --now network

```

---

## 1. 🟢 EcoRouter: rtr-cod (HQ Router)

```bash
enable
configure terminal

hostname rtr-cod
domain-name cod.ssa2026.region

# Интерфейсы
interface gigabitethernet 0/0
 description WAN_TO_ISP
 ip address 178.207.179.4 255.255.255.248
 no shutdown
exit
interface gigabitethernet 0/1
 description LAN_TO_FW
 ip address 10.0.1.1 255.255.255.252
 no shutdown
exit

# BGP (AS 64500)
router bgp 64500
 bgp router-id 178.207.179.4
 neighbor 178.207.179.1 remote-as 31133
 neighbor 178.207.179.1 description ISP_UPLINK
 address-family ipv4
  neighbor 178.207.179.1 activate
  # Анонсируем сеть ЦОД (Network ID для .4/29 -> .0)
  network 178.207.179.0 mask 255.255.255.248
 exit
exit
# [cite_start]Ждем маршрут по умолчанию от провайдера [cite: 130]
neighbor 178.207.179.1 default-originate

# GRE Tunnel to Office
interface Tunnel1
 ip address 10.10.10.1 255.255.255.252
 tunnel source 178.207.179.4
 tunnel destination 178.207.179.28
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

# OSPF
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet 0/1
 area 0 authentication message-digest
 # Wildcard маски!
 network 10.10.10.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
exit

# [cite_start]RADIUS Client [cite: 100]
aaa new-model
radius-server host 192.168.100.10 key radius_secret
aaa authentication login default group radius local
username admin privilege 15 secret P@ssw0rdLocal

line console 0
 login authentication default
exit
line vty 0 4
 login authentication default
 transport input ssh
exit
write

```

---

## 2. 🔥 Ideco NGFW: fw-cod (HQ Firewall)

*Настройка через Web-интерфейс (заходи с admin-cod: [https://10.10.30.1](https://www.google.com/search?q=https://10.10.30.1))*

1. **Внешний интерфейс (WAN):** IP `10.0.1.2/30`, Шлюз `10.0.1.1`.
2. **Локальные интерфейсы (VLANs):** Создать на порту LAN.
* **VLAN 300 (MGMT):** IP `10.10.30.1/24` (Шлюз для свитчей/админа).
* **VLAN 100 (SRV):** IP `192.168.100.1/24` (Шлюз для серверов).
* **VLAN 200 (DATA):** IP `192.168.100.1/24` (Алиас на том же интерфейсе или отдельный).
* **VLAN 500 (VOIP):** IP `192.168.100.1/24`.


3. **OSPF:** Включить. Router-ID `3.3.3.3`. Networks: `10.0.1.0/30`, `10.10.30.0/24`, `192.168.100.0/24`. Auth MD5 `P@ssw0rd`.
4. **Firewall:** Правило **Allow Any** (Разрешить все). Включить NAT.
5. **DNS:** Указать `192.168.100.10`.

---

## 3. 🐧 Alt Server: sw1-cod (HQ Core Switch)

*Порты: ens3->FW, ens4+ens5->sw2 (Bond), ens6->srv2, ens7->admin*

```bash
hostnamectl set-hostname sw1-cod.cod.ssa2026.region

# Очистка
rm -rf /etc/net/ifaces/*
mkdir -p /etc/net/ifaces/lo
echo "TYPE=loopback" > /etc/net/ifaces/lo/options
echo "127.0.0.1/8" > /etc/net/ifaces/lo/ipv4address

# Поднимаем физику
mkdir -p /etc/net/ifaces/ens3
echo "TYPE=eth" > /etc/net/ifaces/ens3/options
mkdir -p /etc/net/ifaces/ens6
echo "TYPE=eth" > /etc/net/ifaces/ens6/options
mkdir -p /etc/net/ifaces/ens7
echo "TYPE=eth" > /etc/net/ifaces/ens7/options

# Bond0 (LACP к sw2)
mkdir -p /etc/net/ifaces/bond0
echo "TYPE=bond" > /etc/net/ifaces/bond0/options
echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
echo "ens4" > /etc/net/ifaces/bond0/ports
echo "ens5" >> /etc/net/ifaces/bond0/ports
echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

# --- VLAN 300 (Admin PC + IP Switch) ---
# Теги от FW и Bond
mkdir -p /etc/net/ifaces/ens3.300
echo "TYPE=vlan" > /etc/net/ifaces/ens3.300/options
echo "VID=300" >> /etc/net/ifaces/ens3.300/options
echo "HOST=ens3" >> /etc/net/ifaces/ens3.300/options

mkdir -p /etc/net/ifaces/bond0.300
echo "TYPE=vlan" > /etc/net/ifaces/bond0.300/options
echo "VID=300" >> /etc/net/ifaces/bond0.300/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.300/options

# МОСТ (Соединяем Админа ens7 и теги)
mkdir -p /etc/net/ifaces/br300
echo "TYPE=bri" > /etc/net/ifaces/br300/options
echo "ens3.300 bond0.300 ens7" > /etc/net/ifaces/br300/ports
# IP Свитча
echo "10.10.30.11/24" > /etc/net/ifaces/br300/ipv4address
echo "default via 10.10.30.1" > /etc/net/ifaces/br300/ipv4route

# --- VLAN 100 (Servers) ---
mkdir -p /etc/net/ifaces/ens3.100
echo "TYPE=vlan" > /etc/net/ifaces/ens3.100/options
echo "VID=100" >> /etc/net/ifaces/ens3.100/options
echo "HOST=ens3" >> /etc/net/ifaces/ens3.100/options

mkdir -p /etc/net/ifaces/bond0.100
echo "TYPE=vlan" > /etc/net/ifaces/bond0.100/options
echo "VID=100" >> /etc/net/ifaces/bond0.100/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.100/options

# К серверу srv2 (Trunk)
mkdir -p /etc/net/ifaces/ens6.100
echo "TYPE=vlan" > /etc/net/ifaces/ens6.100/options
echo "VID=100" >> /etc/net/ifaces/ens6.100/options
echo "HOST=ens6" >> /etc/net/ifaces/ens6.100/options

# Мост
mkdir -p /etc/net/ifaces/br100
echo "TYPE=bri" > /etc/net/ifaces/br100/options
echo "ens3.100 bond0.100 ens6.100" > /etc/net/ifaces/br100/ports

# --- VLAN 200 (Data) ---
# Повторяем логику VLAN 100 для ens3, bond0, ens6
mkdir -p /etc/net/ifaces/ens3.200
echo "TYPE=vlan" > /etc/net/ifaces/ens3.200/options
echo "VID=200" >> /etc/net/ifaces/ens3.200/options
echo "HOST=ens3" >> /etc/net/ifaces/ens3.200/options

mkdir -p /etc/net/ifaces/bond0.200
echo "TYPE=vlan" > /etc/net/ifaces/bond0.200/options
echo "VID=200" >> /etc/net/ifaces/bond0.200/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.200/options

mkdir -p /etc/net/ifaces/ens6.200
echo "TYPE=vlan" > /etc/net/ifaces/ens6.200/options
echo "VID=200" >> /etc/net/ifaces/ens6.200/options
echo "HOST=ens6" >> /etc/net/ifaces/ens6.200/options

mkdir -p /etc/net/ifaces/br200
echo "TYPE=bri" > /etc/net/ifaces/br200/options
echo "ens3.200 bond0.200 ens6.200" > /etc/net/ifaces/br200/ports

systemctl restart network

```

---

## 4. 🐧 Alt Server: sw2-cod (HQ Access Switch)

*Порты: ens4+ens5->sw1 (Bond), ens6->srv1, ens7->sip, ens8->cli*

```bash
hostnamectl set-hostname sw2-cod.cod.ssa2026.region
rm -rf /etc/net/ifaces/*
mkdir -p /etc/net/ifaces/lo
echo "TYPE=loopback" > /etc/net/ifaces/lo/options
echo "127.0.0.1/8" > /etc/net/ifaces/lo/ipv4address

# Поднимаем физику
mkdir -p /etc/net/ifaces/ens6
echo "TYPE=eth" > /etc/net/ifaces/ens6/options
mkdir -p /etc/net/ifaces/ens7
echo "TYPE=eth" > /etc/net/ifaces/ens7/options
mkdir -p /etc/net/ifaces/ens8
echo "TYPE=eth" > /etc/net/ifaces/ens8/options

# Bond0
mkdir -p /etc/net/ifaces/bond0
echo "TYPE=bond" > /etc/net/ifaces/bond0/options
echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
echo "ens4" > /etc/net/ifaces/bond0/ports
echo "ens5" >> /etc/net/ifaces/bond0/ports
echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

# --- VLAN 300 (IP Switch) ---
# Мост не нужен, админа тут нет
mkdir -p /etc/net/ifaces/bond0.300
echo "TYPE=vlan" > /etc/net/ifaces/bond0.300/options
echo "VID=300" >> /etc/net/ifaces/bond0.300/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.300/options
# IP .12 !!!
echo "10.10.30.12/24" > /etc/net/ifaces/bond0.300/ipv4address
echo "default via 10.10.30.1" > /etc/net/ifaces/bond0.300/ipv4route

# --- VLAN 500 (SIP - Access) ---
mkdir -p /etc/net/ifaces/bond0.500
echo "TYPE=vlan" > /etc/net/ifaces/bond0.500/options
echo "VID=500" >> /etc/net/ifaces/bond0.500/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.500/options

mkdir -p /etc/net/ifaces/br500
echo "TYPE=bri" > /etc/net/ifaces/br500/options
# ens7 без точки!
echo "bond0.500 ens7" > /etc/net/ifaces/br500/ports

# --- VLAN 400 (Client - Access) ---
mkdir -p /etc/net/ifaces/bond0.400
echo "TYPE=vlan" > /etc/net/ifaces/bond0.400/options
echo "VID=400" >> /etc/net/ifaces/bond0.400/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.400/options

mkdir -p /etc/net/ifaces/br400
echo "TYPE=bri" > /etc/net/ifaces/br400/options
# ens8 без точки!
echo "bond0.400 ens8" > /etc/net/ifaces/br400/ports

# --- VLAN 100/200 (Trunk to srv1) ---
# VLAN 100
mkdir -p /etc/net/ifaces/bond0.100
echo "TYPE=vlan" > /etc/net/ifaces/bond0.100/options
echo "VID=100" >> /etc/net/ifaces/bond0.100/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.100/options

mkdir -p /etc/net/ifaces/ens6.100
echo "TYPE=vlan" > /etc/net/ifaces/ens6.100/options
echo "VID=100" >> /etc/net/ifaces/ens6.100/options
echo "HOST=ens6" >> /etc/net/ifaces/ens6.100/options

mkdir -p /etc/net/ifaces/br100
echo "TYPE=bri" > /etc/net/ifaces/br100/options
echo "bond0.100 ens6.100" > /etc/net/ifaces/br100/ports

# VLAN 200
mkdir -p /etc/net/ifaces/bond0.200
echo "TYPE=vlan" > /etc/net/ifaces/bond0.200/options
echo "VID=200" >> /etc/net/ifaces/bond0.200/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.200/options

mkdir -p /etc/net/ifaces/ens6.200
echo "TYPE=vlan" > /etc/net/ifaces/ens6.200/options
echo "VID=200" >> /etc/net/ifaces/ens6.200/options
echo "HOST=ens6" >> /etc/net/ifaces/ens6.200/options

mkdir -p /etc/net/ifaces/br200
echo "TYPE=bri" > /etc/net/ifaces/br200/options
echo "bond0.200 ens6.200" > /etc/net/ifaces/br200/ports

systemctl restart network

```

---

## 5. 🟢 EcoRouter: rtr-a (Office Router)

```bash
enable
conf t
hostname rtr-a
domain-name office.ssa2026.region

# WAN
interface gigabitethernet 0/0
 description WAN
 ip address 178.207.179.28 255.255.255.248
 no shutdown
exit

# LAN (Router-on-a-Stick)
# Sub-interfaces для VLAN
interface gigabitethernet 0/1.300
 encapsulation dot1Q 300
 ip address 10.10.30.1 255.255.255.0
 no shutdown
exit
interface gigabitethernet 0/1.200
 encapsulation dot1Q 200
 ip address 192.168.200.1 255.255.255.0
 no shutdown
exit
interface gigabitethernet 0/1.100
 encapsulation dot1Q 100
 ip address 192.168.100.1 255.255.255.0
 no shutdown
exit
interface gigabitethernet 0/1
 no shutdown
exit

# BGP
router bgp 64500
 bgp router-id 178.207.179.28
 neighbor 178.207.179.25 remote-as 31133
 address-family ipv4
  neighbor 178.207.179.25 activate
  # Анонсируем сеть Офиса (.24)
  network 178.207.179.24 mask 255.255.255.248
 exit
exit

# GRE Tunnel
interface Tunnel1
 ip address 10.10.10.2 255.255.255.252
 tunnel source 178.207.179.28
 tunnel destination 178.207.179.4
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

# OSPF
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet 0/1.300
 area 0 authentication message-digest
 network 10.10.10.0 0.0.0.3 area 0
 network 10.10.30.0 0.0.0.255 area 0
 # Анонсируем серверную и клиентскую сети офиса
 network 192.168.100.0 0.0.0.255 area 0
 network 192.168.200.0 0.0.0.255 area 0
exit
write

```

---

## 6. 🐧 Alt Server: sw1-a (Office Core Switch)

*Порты: ens3->rtr, ens4+ens5->sw2 (Redundancy STP), ens6->DC*

```bash
hostnamectl set-hostname sw1-a.office.ssa2026.region
rm -rf /etc/net/ifaces/*
mkdir -p /etc/net/ifaces/lo
echo "TYPE=loopback" > /etc/net/ifaces/lo/options
echo "127.0.0.1/8" > /etc/net/ifaces/lo/ipv4address

mkdir -p /etc/net/ifaces/ens3
echo "TYPE=eth" > /etc/net/ifaces/ens3/options
mkdir -p /etc/net/ifaces/ens4
echo "TYPE=eth" > /etc/net/ifaces/ens4/options
mkdir -p /etc/net/ifaces/ens5
echo "TYPE=eth" > /etc/net/ifaces/ens5/options
mkdir -p /etc/net/ifaces/ens6
echo "TYPE=eth" > /etc/net/ifaces/ens6/options

# --- VLAN 300 (MGMT + STP Root) ---
# От роутера и двух линков
mkdir -p /etc/net/ifaces/ens3.300
echo "TYPE=vlan" > /etc/net/ifaces/ens3.300/options
echo "VID=300" >> /etc/net/ifaces/ens3.300/options
echo "HOST=ens3" >> /etc/net/ifaces/ens3.300/options

mkdir -p /etc/net/ifaces/ens4.300
echo "TYPE=vlan" > /etc/net/ifaces/ens4.300/options
echo "VID=300" >> /etc/net/ifaces/ens4.300/options
echo "HOST=ens4" >> /etc/net/ifaces/ens4.300/options

mkdir -p /etc/net/ifaces/ens5.300
echo "TYPE=vlan" > /etc/net/ifaces/ens5.300/options
echo "VID=300" >> /etc/net/ifaces/ens5.300/options
echo "HOST=ens5" >> /etc/net/ifaces/ens5.300/options

# МОСТ
mkdir -p /etc/net/ifaces/br300
echo "TYPE=bri" > /etc/net/ifaces/br300/options
echo "STP=on" >> /etc/net/ifaces/br300/options
echo "ens3.300 ens4.300 ens5.300" > /etc/net/ifaces/br300/ports

# IP .11
echo "10.10.30.11/24" > /etc/net/ifaces/br300/ipv4address
echo "default via 10.10.30.1" > /etc/net/ifaces/br300/ipv4route

# [cite_start]Root Priority [cite: 120]
echo '#!/bin/sh' > /etc/net/ifaces/br300/ifup-post
echo 'brctl setbridgeprio br300 4096' >> /etc/net/ifaces/br300/ifup-post
chmod +x /etc/net/ifaces/br300/ifup-post

# --- VLAN 200 (Clients Transit) ---
# Тоже самое для ens3, ens4, ens5. (Создай vlan интерфейсы...)
mkdir -p /etc/net/ifaces/ens3.200
echo "TYPE=vlan" > /etc/net/ifaces/ens3.200/options
echo "VID=200" >> /etc/net/ifaces/ens3.200/options
echo "HOST=ens3" >> /etc/net/ifaces/ens3.200/options

mkdir -p /etc/net/ifaces/ens4.200
echo "TYPE=vlan" > /etc/net/ifaces/ens4.200/options
echo "VID=200" >> /etc/net/ifaces/ens4.200/options
echo "HOST=ens4" >> /etc/net/ifaces/ens4.200/options

mkdir -p /etc/net/ifaces/ens5.200
echo "TYPE=vlan" > /etc/net/ifaces/ens5.200/options
echo "VID=200" >> /etc/net/ifaces/ens5.200/options
echo "HOST=ens5" >> /etc/net/ifaces/ens5.200/options

mkdir -p /etc/net/ifaces/br200
echo "TYPE=bri" > /etc/net/ifaces/br200/options
echo "STP=on" >> /etc/net/ifaces/br200/options
echo "ens3.200 ens4.200 ens5.200" > /etc/net/ifaces/br200/ports

# Root Priority
echo '#!/bin/sh' > /etc/net/ifaces/br200/ifup-post
echo 'brctl setbridgeprio br200 4096' >> /etc/net/ifaces/br200/ifup-post
chmod +x /etc/net/ifaces/br200/ifup-post

# --- VLAN 100 (DC Access) ---
mkdir -p /etc/net/ifaces/ens3.100
echo "TYPE=vlan" > /etc/net/ifaces/ens3.100/options
echo "VID=100" >> /etc/net/ifaces/ens3.100/options
echo "HOST=ens3" >> /etc/net/ifaces/ens3.100/options

mkdir -p /etc/net/ifaces/br100
echo "TYPE=bri" > /etc/net/ifaces/br100/options
# ens6 - порт DC (Access)
echo "ens3.100 ens6" > /etc/net/ifaces/br100/ports

systemctl restart network

```

---

## 7. 🐧 Alt Server: sw2-a (Office Access Switch)

*Порты: ens4+ens5->sw1 (STP), ens6,ens7->Clients*

```bash
hostnamectl set-hostname sw2-a.office.ssa2026.region
rm -rf /etc/net/ifaces/*
# ... (lo loopback как выше)

# --- VLAN 300 (IP Switch) ---
# Настройка ens4.300, ens5.300
# Бридж br300, STP=on
# Ports="ens4.300 ens5.300"
# IP 10.10.30.12, GW 10.10.30.1

# --- VLAN 200 (Clients) ---
# Настройка ens4.200, ens5.200
# Бридж br200, STP=on
# Ports="ens4.200 ens5.200 ens6 ens7"
# (ens6, ens7 - access порты клиентов)

systemctl restart network

```

---

## 8. 💾 Storage: srv2-cod (iSCSI Target)

*Диск: проверь `lsblk` (sdb/vdb?)*

```bash
# Сеть
# ens18.100, ens18.200 (IP .11) -> смотри sw1-cod настройки
# У srv2-cod IP 192.168.100.11

apt-get install targetcli
systemctl enable --now target

targetcli
'''
/backstores/block create name=disk1 dev=/dev/sdb
/iscsi create iqn.2026-01.region.ssa2026.cod:data.target
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/acls create iqn.2026-01.region.ssa2026.cod:iscsi
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/luns create /backstores/block/disk1
exit
'''

```

---

## 9. 🛠️ Services: srv1-cod (Radius, DNS, Initiator)

```bash
# Сеть: IP 192.168.100.10

# --- RADIUS ---
apt-get install freeradius freeradius-utils
nano /etc/raddb/clients.conf
'''
client rtr-cod { ipaddr = 10.10.30.1 secret = radius_secret }
client sw1-cod { ipaddr = 10.10.30.11 secret = radius_secret }
client sw2-cod { ipaddr = 10.10.30.12 secret = radius_secret }
'''
nano /etc/raddb/users
'''
netuser Cleartext-Password := "P@ssw0rd"
       Service-Type = Administrative-User
'''
systemctl enable --now radiusd

# --- DNS (Bind) ---
apt-get install bind bind-utils
nano /etc/bind/options.conf
'''
options {
    listen-on { any; };
    allow-query { any; };
    recursion yes;
    forwarders { 100.100.100.100; };
    dnssec-validation no;
};
'''
nano /etc/bind/local.conf
'''
zone "cod.ssa2026.region" IN { type master; file "/var/lib/bind/cod.ssa2026.region.db"; };
zone "100.168.192.in-addr.arpa" IN { type master; file "/var/lib/bind/100.168.192.db"; };
'''

# Прямая зона
nano /var/lib/bind/cod.ssa2026.region.db
'''
$TTL 86400
@ IN SOA srv1-cod.cod.ssa2026.region. root.cod.ssa2026.region. ( 1 1h 15m 1w 1d )
@ IN NS srv1-cod.cod.ssa2026.region.
@ IN A 192.168.100.10
srv1-cod IN A 192.168.100.10
srv2-cod IN A 192.168.100.11
sw1-cod  IN A 10.10.30.11
sw2-cod  IN A 10.10.30.12
rtr-cod  IN A 10.10.30.1
fw-cod   IN A 192.168.100.1
sip-cod  IN A 192.168.100.20
admin-cod IN A 10.10.30.100
monitoring IN CNAME srv1-cod
'''

# Обратная зона (PTR)
nano /var/lib/bind/100.168.192.db
'''
$TTL 86400
@ IN SOA srv1-cod.cod.ssa2026.region. root.cod.ssa2026.region. ( 1 1h 15m 1w 1d )
@ IN NS srv1-cod.cod.ssa2026.region.
10 IN PTR srv1-cod.cod.ssa2026.region.
11 IN PTR srv2-cod.cod.ssa2026.region.
20 IN PTR sip-cod.cod.ssa2026.region.
'''
systemctl enable --now bind

# --- iSCSI Initiator ---
apt-get install open-iscsi lvm2 nfs-utils
echo "InitiatorName=iqn.2026-01.region.ssa2026.cod:iscsi" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid
iscsiadm -m discovery -t st -p 192.168.100.11
iscsiadm -m node --login

# LVM & NFS
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

mkdir -p /opt/data
# blkid /dev/VG/DATA -> UUID
# fstab -> UUID="<<UUID>>" /opt/data xfs _netdev 0 0
mount -a

echo "/opt/data 10.10.30.0/24(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server

```

---

## 10. 🏢 Active Directory: dc-a

```bash
hostnamectl set-hostname dc-a.office.ssa2026.region
apt-get install samba-dc bind bind-utils
rm -f /etc/samba/smb.conf

# Provision
samba-tool domain provision \
  --realm=OFFICE.SSA2026.REGION \
  --domain=OFFICE \
  --adminpass=P@ssw0rd \
  --server-role=dc \
  --dns-backend=BIND9_DLZ

# Bind Integration
nano /etc/bind/named.conf
'''
tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";
include "/var/lib/samba/bind-dns/named.conf";
'''
chgrp named /var/lib/samba/private/dns.keytab
chmod g+r /var/lib/samba/private/dns.keytab

systemctl disable smb nmb
systemctl enable --now samba-ad-dc
systemctl restart bind

# [cite_start]Users [cite: 201]
samba-tool ou create "OU=ofadmins"
samba-tool ou create "OU=ofusers"
samba-tool group add ofadmins --groupou="OU=ofadmins"
samba-tool group add ofusers --groupou="OU=ofusers"

samba-tool user create ofadmin1 P@ssw0rd --userou="OU=ofadmins"
samba-tool group addmembers ofadmins ofadmin1

samba-tool user create ofuser1 P@ssw0rd --userou="OU=ofusers"
samba-tool group addmembers ofusers ofuser1

samba-tool user create user1 P@ssw0rd

```

---

## 11. 📊 Services: Zabbix & SIP

```bash
# --- Zabbix (srv1) ---
# DB (srv2): create user zabbix_user...
# Import: zcat /usr/share/doc/zabbix-server-pgsql-*/create.sql.gz | psql -h 192.168.100.11 ...
# Config: /etc/zabbix/zabbix_server.conf -> DBHost=192.168.100.11

# --- SIP (sip-cod) ---
# 1. Settings -> Asterisk SIP Settings -> PJSIP -> Port 5160
# [cite_start]2. Settings -> Asterisk SIP Settings -> Chan_SIP -> Bind Port 5060 [cite: 244]
# 3. Extensions: 1001, 1002, 2001, 2002

```
