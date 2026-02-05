![](/1.png)
![](/2.png)
![](/3.png)

Название устройства  ОС
rtr-cod	             EcoRouterOS (7-jasmine)
fw-cod	              Ideco NGFW NOVUM 21
sw1-cod	             Альт Сервер 11
sw2-cod	             Альт Сервер 11
cli-cod	             Альт Рабочая станция 11
srv1-cod	            Альт Сервер 11
srv2-cod	            Альт Сервер 11
sip-cod	             SNG7-PBX16
admin-cod	           Альт Рабочая станция 11
rtr-a	               EcoRouterOS (7-jasmine)
sw1-a	               Альт Сервер 11
sw2-a	               Альт Сервер 11
dc-a	                Альт Сервер 11
cli1-a	              Альт Рабочая станция 11
cli2-a	              Альт Рабочая станция 11


Это отличная структура для шпаргалки. Я сохранил твой стиль (Заголовок -> Описание -> Блок кода), но сделал **три критически важных исправления**, чтобы система заработала:

1. **Порядок серверов:** Я перенес настройку **srv2-cod (Target)** *перед* настройкой **srv1-cod (Initiator)**. Ты не сможешь подключить диск на первом сервере, пока не создашь его на втором.
2. **Сеть на srv2:** В твоем тексте у `srv2-cod` не было настройки IP-адреса. Без IP `srv1` его не найдет. Я добавил настройку сети.
3. **Форматирование:** Я использовал блоки кода с подсветкой (`bash`/`cisco`), чтобы на GitHub это выглядело профессионально и читаемо.

Вот готовый код для твоего `README.md`:

---

# rtr-cod (EcoRouter)

Входим в привилегированный режим

```bash
enable
configure terminal

```

1 Имя устройства 

```bash
hostname rtr-cod
domain-name cod.ssa2026.region

```

2 Настройка внешнего интерфейса (к ISP) 
Смотрим таблицу 83: IP 178.207.179.4/29

```bash
interface gigabitethernet 0/0
 description WAN_TO_ISP
 ip address 178.207.179.4 255.255.255.248
 no shutdown
exit

```

3 Настройка внутреннего интерфейса (к Firewall fw-cod)
Адрес придумываем из приватной сети, например 10.0.1.1/30

```bash
interface gigabitethernet 0/1
 description LAN_TO_FW
 ip address 10.0.1.1 255.255.255.252
 no shutdown
exit

```

4 Настройка BGP с провайдером (AS 64500 <-> AS 31133) 

```bash
router bgp 64500
 bgp router-id 178.207.179.4

```

Указываем соседа (шлюз провайдера)

```bash
 neighbor 178.207.179.1 remote-as 31133
 neighbor 178.207.179.1 description ISP_UPLINK

 address-family ipv4
  neighbor 178.207.179.1 activate

```

Запрет анонса внутренних сетей (пункт 129) - не добавляем network 10.x.x.x
Но свой внешний адрес анонсировать нужно (пункт 86)

```bash
  network 178.207.179.0 mask 255.255.255.248
 exit
exit

```

Ждем маршрут по умолчанию от провайдера (пункт 130). Команду ниже вводим, если авто-получение не сработало.

```bash
neighbor 178.207.179.1 default-originate

```

5 Настройка GRE туннеля до офиса (rtr-a) 

```bash
interface Tunnel1

```

IP адрес туннеля (сеть 10.10.10.0/24, мин. маска /30 -> .1 и .2)

```bash
 ip address 10.10.10.1 255.255.255.252

```

От кого строим (наш внешний IP)

```bash
 tunnel source 178.207.179.4

```

К кому строим (внешний IP rtr-a из таблицы 83)

```bash
 tunnel destination 178.207.179.28
 tunnel mode gre ip

```

Включаем OSPF на туннеле и задаем пароль (пункт 137)

```bash
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

```

6 Настройка OSPF (динамическая маршрутизация) 

```bash
router ospf 1
 router-id 1.1.1.1

```

Все интерфейсы пассивные по умолчанию (пункт 134)

```bash
 passive-interface default

```

Туннель активный (чтобы обмениваться маршрутами с rtr-a)

```bash
 no passive-interface Tunnel1

```

Интерфейс к firewall тоже активный (если fw участвует в OSPF)

```bash
 no passive-interface gigabitethernet 0/1

```

Объявляем сети

```bash
 network 10.10.10.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0

```

Включаем аутентификацию для зоны 0 и cохраняем настройки

```bash
 area 0 authentication message-digest
exit
write

```

# rtr-cod (RADIUS Client)

*Настраиваем после того, как подняли srv1-cod*

```bash
enable
conf t

```

1 Включаем службу AAA (Authentication, Authorization, Accounting) 

```bash
aaa new-model

```

2 Указываем, где живет сервер RADIUS (IP сервера srv1-cod)

```bash
radius-server host 192.168.100.10 key radius_secret

```

3 Создаем список методов аутентификации "default"
Сначала пробуем group radius, если сервер недоступен - local (локальная база)

```bash
aaa authentication login default group radius local

```

4 Настраиваем локального пользователя (на случай сбоя RADIUS)

```bash
username admin privilege 15 secret P@ssw0rdLocal

```

*В задании сказано: "Для rtr-cod пользователь netuser не должен существовать локально". Поэтому netuser мы тут НЕ создаем.*

5 Применяем защиту на линии (Console и VTY/SSH)

```bash
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

# rtr-a (EcoRouter)

```bash
enable
configure terminal

```

1 Имя 

```bash
hostname rtr-a
domain-name office.ssa2026.region

```

2 Внешний интерфейс 

```bash
interface gigabitethernet 0/0
 description WAN_TO_ISP
 ip address 178.207.179.28 255.255.255.248
 no shutdown
exit

```

3 Внутренний интерфейс (VLAN Trunk или роутер на палочке)
В схеме L-2 rtr-a подключен к sw1-a. Настроим саб-интерфейсы для VLAN.

VLAN 300 (MGMT) 

```bash
interface gigabitethernet 0/1.300
 encapsulation dot1Q 300
 ip address 10.10.30.1 255.255.255.0
 no shutdown
exit

```

VLAN 200 (CLI) 

```bash
interface gigabitethernet 0/1.200
 encapsulation dot1Q 200
 ip address 192.168.200.1 255.255.255.0
 no shutdown
exit

```

Подними основной интерфейс

```bash
interface gigabitethernet 0/1
 no shutdown
exit

```

4 BGP с провайдером 

```bash
router bgp 64500
 bgp router-id 178.207.179.28
 neighbor 178.207.179.25 remote-as 31133
 neighbor 178.207.179.25 description ISP_GW
 address-family ipv4
  neighbor 178.207.179.25 activate
  network 178.207.179.24 mask 255.255.255.248
 exit
exit

```

5 GRE Туннель (ответная часть) 

```bash
interface Tunnel1
 ip address 10.10.10.2 255.255.255.252
 tunnel source 178.207.179.28
 tunnel destination 178.207.179.4
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

```

6 OSPF 

```bash
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

# sw1-cod (Alt Linux)

1 Задаем имя хоста

```bash
hostnamectl set-hostname sw1-cod.cod.ssa2026.region

```

2 Создаем Агрегацию (Bond0) для sw1-sw2 
Создаем папку интерфейса

```bash
mkdir -p /etc/net/ifaces/bond0

```

Описываем настройки: active-backup

```bash
echo "TYPE=bond" > /etc/net/ifaces/bond0/options
echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options

```

Указываем, какие физические порты входят в бонд

```bash
echo "ens4" > /etc/net/ifaces/bond0/ports
echo "ens5" >> /etc/net/ifaces/bond0/ports

```

Отключаем IP на самом бонде (он будет транком)

```bash
echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

```

ВАЖНО: Удаляем настройки с физических портов ens4/ens5

```bash
rm -rf /etc/net/ifaces/ens4
rm -rf /etc/net/ifaces/ens5

```

3 Настройка VLAN управления (VLAN 300) 
Интерфейс будет называться vlan300

```bash
mkdir -p /etc/net/ifaces/vlan300
echo "TYPE=vlan" > /etc/net/ifaces/vlan300/options
echo "VID=300" >> /etc/net/ifaces/vlan300/options

```

VLAN "сидит" на интерфейсе bond0. Для простоты управления вешаем на bond0

```bash
echo "HOST=bond0" >> /etc/net/ifaces/vlan300/options

```

Задаем IP управления и шлюз

```bash
echo "10.10.30.11/24" > /etc/net/ifaces/vlan300/ipv4address
echo "default via 10.10.30.1" > /etc/net/ifaces/vlan300/ipv4route

```

4 Настройка остальных VLAN (Транки) 
Создаем мост для VLAN 100 (Servers):

```bash
# VLAN на бонде
mkdir -p /etc/net/ifaces/bond0.100
echo "TYPE=vlan" > /etc/net/ifaces/bond0.100/options
echo "VID=100" >> /etc/net/ifaces/bond0.100/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.100/options

# VLAN на порту серверов (ens6)
mkdir -p /etc/net/ifaces/ens6.100
echo "TYPE=vlan" > /etc/net/ifaces/ens6.100/options
echo "VID=100" >> /etc/net/ifaces/ens6.100/options
echo "HOST=ens6" >> /etc/net/ifaces/ens6.100/options

# Мост br100
mkdir -p /etc/net/ifaces/br100
echo "TYPE=bri" > /etc/net/ifaces/br100/options
echo "bond0.100" > /etc/net/ifaces/br100/ports
echo "ens6.100" >> /etc/net/ifaces/br100/ports

```

5 Применяем настройки

```bash
systemctl restart network
ip a

```

---

# sw2-cod (Alt Linux)

1 Имя хоста

```bash
hostnamectl set-hostname sw2-cod.cod.ssa2026.region

```

2 Агрегация (Bond0) - Active-Backup

```bash
mkdir -p /etc/net/ifaces/bond0
echo "TYPE=bond" > /etc/net/ifaces/bond0/options
echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
echo "ens4" > /etc/net/ifaces/bond0/ports
echo "ens5" >> /etc/net/ifaces/bond0/ports
echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

rm -rf /etc/net/ifaces/ens4
rm -rf /etc/net/ifaces/ens5

```

3 VLAN управления (VLAN 300)

```bash
mkdir -p /etc/net/ifaces/vlan300
echo "TYPE=vlan" > /etc/net/ifaces/vlan300/options
echo "VID=300" >> /etc/net/ifaces/vlan300/options
echo "HOST=bond0" >> /etc/net/ifaces/vlan300/options

```

IP адрес управления (другой, не как у sw1!)

```bash
echo "10.10.30.12/24" > /etc/net/ifaces/vlan300/ipv4address
echo "default via 10.10.30.1" > /etc/net/ifaces/vlan300/ipv4route

```

4 Проброс VLAN (Транки) для клиентов
Например, VLAN 400 (CLI)

```bash
mkdir -p /etc/net/ifaces/bond0.400
echo "TYPE=vlan" > /etc/net/ifaces/bond0.400/options
echo "VID=400" >> /etc/net/ifaces/bond0.400/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.400/options
systemctl restart network

```

---

# srv2-cod (Alt Linux) - iSCSI Target

**ВАЖНО: Настраиваем SRV2 СНАЧАЛА, чтобы SRV1 мог к нему подключиться!**

1 Настройка Сети
*У сервера должен быть IP в сети DATA (VLAN 200 или 100), чтобы его видел srv1. Предположим VLAN 100 или транк.*

```bash
mkdir -p /etc/net/ifaces/ens18
echo "TYPE=eth" > /etc/net/ifaces/ens18/options
echo "192.168.100.11/24" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
systemctl restart network

```

2 Установка пакетов 

```bash
apt-get install targetcli
systemctl enable --now target

```

3 Настройка через консоль targetcli 

```bash
targetcli

```

Внутри консоли вводим команды:

```bash
# Создаем диск-хранилище
/backstores/block create name=disk1 dev=/dev/sdb

# Создаем цель (Target)
/iscsi create iqn.2026-01.region.ssa2026.cod:data.target

# Разрешаем доступ клиенту (ACL)
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/acls create iqn.2026-01.region.ssa2026.cod:iscsi

# Привязываем диск
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/luns create /backstores/block/disk1

exit

```

---

# srv1-cod (RADIUS, DNS, iSCSI Init)

1 Настройка сети
Пример настройки интерфейса ens18 (проверь имя через ip a)

```bash
mkdir -p /etc/net/ifaces/ens18
echo "TYPE=eth" > /etc/net/ifaces/ens18/options
echo "192.168.100.10/24" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
systemctl restart network

```

2 Установка RADIUS 

```bash
apt-get update
apt-get install freeradius freeradius-utils

```

3 Добавляем клиентов (роутер и свитчи)
Редактируем файл `/etc/raddb/clients.conf`

```bash
client rtr-cod {
    ipaddr = 10.10.30.1
    secret = radius_secret
}
client sw1-cod {
    ipaddr = 10.10.30.11
    secret = radius_secret
}

```

4 Добавляем пользователя netuser 
Редактируем файл `/etc/raddb/users`

```bash
netuser Cleartext-Password := "P@ssw0rd"
       Service-Type = Administrative-User

```

5 Запуск и проверка

```bash
systemctl enable --now radiusd
radtest netuser P@ssw0rd localhost 0 testing123

```

6 Установка DNS (Bind) 

```bash
apt-get install bind bind-utils

```

7 Настройка основного конфига (`/etc/bind/options.conf`)

```bash
options {
    listen-on { any; };
    allow-query { any; };
    recursion yes;
    forwarders { 100.100.100.100; };
    dnssec-validation no;
};

```

8 Описание зон (`/etc/bind/local.conf`)

```bash
zone "cod.ssa2026.region" IN {
    type master;
    file "/var/lib/bind/cod.ssa2026.region.db";
};
zone "100.168.192.in-addr.arpa" IN {
    type master;
    file "/var/lib/bind/100.168.192.db";
};

```

9 Файл прямой зоны (`/var/lib/bind/cod.ssa2026.region.db`)

```bash
$TTL 86400
@   IN  SOA     srv1-cod.cod.ssa2026.region. root.cod.ssa2026.region. (
        2026012801 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      srv1-cod.cod.ssa2026.region.
@       IN  A       192.168.100.10

; Записи для устройств
srv1-cod IN  A       192.168.100.10
srv2-cod IN  A       192.168.100.11
fw-cod   IN  A       192.168.100.1
rtr-cod  IN  A       10.10.30.1
sw1-cod  IN  A       10.10.30.11
sw2-cod  IN  A       10.10.30.12
sip-cod  IN  A       192.168.100.20
admin-cod IN A       10.10.30.100
monitoring IN CNAME  srv1-cod

```

Проверка: `systemctl enable --now bind`

10 Центр Сертификации (CA) 

```bash
mkdir -p /var/ca/{certs,crl,newcerts,private}
chmod 700 /var/ca/private
touch /var/ca/index.txt
echo 1000 > /var/ca/serial
cp /etc/ssl/openssl.cnf /var/ca/openssl.cnf

```

Редактируем `/var/ca/openssl.cnf`:

* `dir = /var/ca`
* `default_days = 1825`
* `organizationName_default = IRPO`
* `commonName_default = ssa2026`

Генерируем:

```bash
cd /var/ca
openssl genrsa -out private/ca.key 4096
openssl req -new -x509 -key private/ca.key -out ca.crt -days 1825 -config openssl.cnf
cp ca.crt /usr/share/ca-certificates/ssa2026.crt
update-ca-trust

```

11 iSCSI Инициатор (Клиент) 
*Подключаем диск, который мы расшарили на SRV2*

```bash
apt-get install open-iscsi lvm2 nfs-utils
echo "InitiatorName=iqn.2026-01.region.ssa2026.cod:iscsi" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid

```

Подключение к таргету (IP адрес srv2-cod!)

```bash
iscsiadm -m discovery -t st -p 192.168.100.11
iscsiadm -m node --login

```

12 LVM и NFS 

```bash
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

mkdir -p /opt/data
# Узнаем UUID: blkid /dev/VG/DATA
# Добавляем в fstab: UUID="xxxx" /opt/data xfs _netdev 0 0
mount -a

# NFS
echo "/opt/data 10.10.30.0/24(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server

```

# admin-cod (Admin Workstation)

1 Подключение сетевого диска (NFS Client) 

```bash
apt-get update
apt-get install nfs-clients
mkdir -p /mnt/nfs

```

Настройка авто-монтирования (`/etc/fstab`)

```bash
# Добавляем строку: IP_SRV1:/path /local_path nfs _netdev 0 0
echo "10.10.30.11:/opt/data /mnt/nfs nfs _netdev 0 0" >> /etc/fstab
mount -a

```

2 Подключение к БД (DBeaver) 
*Выполняется в графическом интерфейсе*

* **Host**: 192.168.100.11 (srv2-cod)
* **Database**: postgres
* **User**: superadmin
* **Password**: P@ssw0rdSQL

---

# dc-a (Samba AD DC)

1 Подготовка
*Установи статический IP (VLAN 100), например 192.168.100.10*

```bash
hostnamectl set-hostname dc-a.office.ssa2026.region
# Проверь /etc/hosts: 192.168.100.10 dc-a.office.ssa2026.region dc-a

```

2 Установка и поднятие домена 

```bash
apt-get install samba-dc bind bind-utils
rm -f /etc/samba/smb.conf

samba-tool domain provision \
  --realm=OFFICE.SSA2026.REGION \
  --domain=OFFICE \
  --adminpass=P@ssw0rd \
  --server-role=dc \
  --dns-backend=BIND9_DLZ

```

3 Интеграция с Bind
В файл `/etc/bind/named.conf` (или options) добавь:

```bash
tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";
# В конец файла:
include "/var/lib/samba/bind-dns/named.conf";

```

Права доступа:

```bash
chgrp named /var/lib/samba/private/dns.keytab
chmod g+r /var/lib/samba/private/dns.keytab

```

4 Запуск

```bash
systemctl stop smb nmb
systemctl disable smb nmb
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc
systemctl restart bind

```

5 Создание пользователей и групп 

```bash
# Подразделения
samba-tool ou create "OU=ofadmins"
samba-tool ou create "OU=ofusers"

# Группы
samba-tool group add ofadmins --groupou="OU=ofadmins"
samba-tool group add ofusers --groupou="OU=ofusers"

# Пользователи
samba-tool user create ofadmin1 P@ssw0rd --userou="OU=ofadmins"
samba-tool user create ofuser1 P@ssw0rd --userou="OU=ofusers"
samba-tool user create user1 P@ssw0rd

# Добавление в группы
samba-tool group addmembers ofadmins ofadmin1
samba-tool group addmembers ofusers ofuser1

```

---

# cli1-a (Client)

1 Ввод в домен 

```bash
# Указываем DNS на контроллер домена
echo "nameserver 192.168.100.10" > /etc/resolv.conf

# Ввод (используй system-auth или net ads join)
system-auth write ad domain office.ssa2026.region computer cli1-a login administrator password P@ssw0rd

# Включаем Winbind
systemctl enable --now winbind

```

2 Групповые политики (ADMC) 
*Выполняется в графике под ofadmin1*

* Запустить `admc`.
* Подключиться к домену.
* GPO -> User Configuration -> Desktop -> Wallpaper (запретить смену, установить картинку).
* GPO -> User Configuration -> Network -> Prohibit changes.

---

# Zabbix (Monitoring)

**Настройка на srv1-cod** 

1 Установка

```bash
apt-get install zabbix-server-pgsql zabbix-web-apache-pgsql zabbix-agent

```

2 Импорт БД (на удаленный srv2)

```bash
# Вводим пароль P@ssw0rdZabbix
zcat /usr/share/doc/zabbix-server-pgsql-*/create.sql.gz | psql -h 192.168.100.11 -U zabbix_user -d zabbix

```

3 Конфиг `/etc/zabbix/zabbix_server.conf`

```bash
DBHost=192.168.100.11
DBName=zabbix
DBUser=zabbix_user
DBPassword=P@ssw0rdZabbix

```

4 Настройка HTTPS (Apache)
В конфиге `/etc/httpd2/conf/sites-available/ssl.conf`:

```apache
SSLEngine on
SSLCertificateFile /var/ca/certs/srv1-cod.crt
SSLCertificateKeyFile /var/ca/private/srv1-cod.key

```

Запуск:

```bash
systemctl enable --now httpd2 zabbix-server zabbix-agent

```

**Настройка Агентов (Linux Hosts)** 
В файле `/etc/zabbix/zabbix_agentd.conf`:

```bash
Server=192.168.100.10
ServerActive=192.168.100.10
Hostname=<ИМЯ_ЭТОЙ_МАШИНЫ>

```

**Настройка SNMP (EcoRouter)** 

```bash
snmp-server community public ro

```

---

# sip-cod (IP Telephony)

*Настройка через Web-интерфейс (http://IP-SIP-COD)*

1. 
**Создать Extensions (Chan_SIP)** :


* 1001 (admin-cod)
* 1002 (cli-cod)
* 2001 (cli1-a)
* 2002 (cli2-a)
* Secret: `P@ssw0rd`


2. 
**Настройка портов**:


* Settings -> Asterisk SIP Settings.
* **Chan SIP**: Bind Port `5060`.
* **PJSIP**: Bind Port `5160` (или отключить).


3. 
**Софтфоны (на клиентах)**:


* Установить `linphone`.
* Вход: `1001@IP_SIP_COD`, пароль `P@ssw0rd`.
