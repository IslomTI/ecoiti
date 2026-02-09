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

---

## 1. 🟢 EcoRouter: rtr-cod (HQ)

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
neighbor 178.207.179.1 default-originate

# GRE Tunnel
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

# RADIUS Client
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

## 2. 🟢 EcoRouter: rtr-a (Office)

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
# VLAN 300 (MGMT)
interface gigabitethernet 0/1.300
 encapsulation dot1Q 300
 ip address 10.10.30.1 255.255.255.0
 no shutdown
exit
# VLAN 200 (CLI)
interface gigabitethernet 0/1.200
 encapsulation dot1Q 200
 ip address 192.168.200.1 255.255.255.0
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
  # Анонсируем сеть Офиса (Network ID для .28/29 -> .24)
  network 178.207.179.24 mask 255.255.255.248
 exit
exit

# GRE Tunnel (Ответ)
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
exit
write

```

---

## 3. 🐧 Alt Linux: Switches (Network)

```bash
# --- sw1-cod ---
hostnamectl set-hostname sw1-cod.cod.ssa2026.region

# Bond0
nano /etc/net/ifaces/bond0/options
'''
TYPE=bond
bond-mode 1
bond-miimon 100
'''
nano /etc/net/ifaces/bond0/ports
'''
ens4
ens5
'''
/etc/net/ifaces/bond0/ipv4address -> BOOTPROTO=static
rm -rf /etc/net/ifaces/ens4 /etc/net/ifaces/ens5

# VLAN 300 (MGMT IP .11)
nano /etc/net/ifaces/vlan300/options
'''
TYPE=vlan
VID=300
HOST=bond0
'''
/etc/net/ifaces/vlan300/ipv4address -> 10.10.30.11/24
/etc/net/ifaces/vlan300/ipv4route -> default via 10.10.30.1

# Bridge 100 (Servers)
nano /etc/net/ifaces/bond0.100/options
'''
TYPE=vlan
VID=100
HOST=bond0
'''
nano /etc/net/ifaces/ens6.100/options
'''
TYPE=vlan
VID=100
HOST=ens6
'''
nano /etc/net/ifaces/br100/options
'''
TYPE=bri
PORTS="bond0.100 ens6.100"
'''
systemctl restart network

# --- sw2-cod (IP .12) ---
hostnamectl set-hostname sw2-cod.cod.ssa2026.region
# (Bond0 настройки идентичны sw1)
# ...

# VLAN 300 (MGMT IP .12 !!!)
nano /etc/net/ifaces/vlan300/options
'''
TYPE=vlan
VID=300
HOST=bond0
'''
/etc/net/ifaces/vlan300/ipv4address -> 10.10.30.12/24
/etc/net/ifaces/vlan300/ipv4route -> default via 10.10.30.1
systemctl restart network

```

---

## 4. 💾 Storage: srv2-cod (Target)

```bash
# Сеть
/etc/net/ifaces/ens18/options -> TYPE=eth
/etc/net/ifaces/ens18/ipv4address -> 192.168.100.11/24
/etc/net/ifaces/ens18/ipv4route -> default via 192.168.100.1
systemctl restart network

# iSCSI
apt-get install targetcli
systemctl enable --now target

# Проверка диска
lsblk
# (Допустим диск sdb)

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

## 5. 🛠️ Services: srv1-cod (Radius, DNS, Initiator)

```bash
# Сеть (IP .10)
/etc/net/ifaces/ens18/ipv4address -> 192.168.100.10/24
systemctl restart network

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

# --- iSCSI Initiator & NFS ---
apt-get install open-iscsi lvm2 nfs-utils
echo "InitiatorName=iqn.2026-01.region.ssa2026.cod:iscsi" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid
iscsiadm -m discovery -t st -p 192.168.100.11
iscsiadm -m node --login

# LVM
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

# Mount
mkdir -p /opt/data
# blkid /dev/VG/DATA -> берем UUID
# fstab -> UUID="<<UUID>>" /opt/data xfs _netdev 0 0
mount -a

# NFS Export
echo "/opt/data 10.10.30.0/24(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server

```

---

## 6. 🏢 Active Directory: dc-a

```bash
# Сеть (IP .10)
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

# Users
samba-tool ou create "OU=ofadmins"
samba-tool ou create "OU=ofusers"
samba-tool group add ofadmins --groupou="OU=ofadmins"
samba-tool group add ofusers --groupou="OU=ofusers"

samba-tool user create ofadmin1 P@ssw0rd --userou="OU=ofadmins"
samba-tool group addmembers ofadmins ofadmin1

samba-tool user create ofuser1 P@ssw0rd --userou="OU=ofusers"
samba-tool group addmembers ofusers ofuser1

# USER1 (без групп!)
samba-tool user create user1 P@ssw0rd

```

---

## 7. 📊 Zabbix & SIP

```bash
# --- Zabbix (srv1) ---
# DB (srv2): create user zabbix_user...
# Import: zcat /usr/share/doc/zabbix-server-pgsql-*/create.sql.gz | psql -h 192.168.100.11 ...
# Config: /etc/zabbix/zabbix_server.conf -> DBHost=192.168.100.11
# Apache SSL: SSLCertificateFile /var/ca/certs/srv1-cod.crt

# --- SIP (sip-cod) ---
# 1. Зайти в Settings -> Asterisk SIP Settings -> PJSIP
# 2. Сменить порт PJSIP на 5160 (иначе конфликт!)
# 3. Chan_SIP -> Bind Port 5060
# 4. Создать Extensions 1001, 1002, 2001, 2002

```

```

### 🔎 Что я исправил и почему (это можно не писать в гитхаб):

1.  **IP-адреса свитчей:** В твоем файле для `sw2-cod` был IP `.11` (как у первого). Я исправил на `.12` в разделе *3. Alt Linux*, иначе будет конфликт IP.
2.  **BGP Networks:**
    * У `rtr-cod` сеть провайдера `.4/29`, значит Network ID `.0`. Исправил.
    * У `rtr-a` сеть провайдера `.28/29`, значит Network ID `.24`. Исправил.
3.  [cite_start]**Обратная зона DNS:** В задании [cite: 146] требуют PTR записи. Я добавил создание файла `/var/lib/bind/100.168.192.db` в разделе *5. Services*.
4.  [cite_start]**RADIUS Clients:** В задании [cite: 100] сказано, что `sw2-cod` тоже должен авторизоваться через RADIUS. Добавил его в `clients.conf`.
5.  [cite_start]**SIP Порты:** Добавил важное напоминание про смену порта PJSIP на 5160, так как задание требует 5060 для Chan_SIP[cite: 244]. По умолчанию они конфликтуют.
6.  [cite_start]**Пользователь user1:** Убрал добавление `user1` в группы, так как в задании [cite: 207] это запрещено.

```
