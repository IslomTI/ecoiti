# rtr-cod (EcoRouter)
Входим в привилегированный режим

    enable
    configure terminal
1 Имя устройства

    hostname rtr-cod
    domain-name cod.ssa2026.region
2 Настройка внешнего интерфейса (к ISP)
Смотрим таблицу 83: IP 178.207.179.4/29

    interface gigabitethernet 0/0
     description WAN_TO_ISP
     ip address 178.207.179.4 255.255.255.248
     no shutdown
    exit
3 Настройка внутреннего интерфейса (к Firewall fw-cod)
Адрес придумываем из приватной сети, например 10.0.1.1/30

    interface gigabitethernet 0/1
     description LAN_TO_FW
     ip address 10.0.1.1 255.255.255.252
     no shutdown
    exit
4 Настройка BGP с провайдером (AS 64500 <-> AS 31133)

    router bgp 64500
     bgp router-id 178.207.179.4  
Указываем соседа (шлюз провайдера)
 
     neighbor 178.207.179.1 remote-as 31133
     neighbor 178.207.179.1 description ISP_UPLINK
 
     address-family ipv4
      neighbor 178.207.179.1 activate  
Запрет анонса внутренних сетей (пункт 129) - не добавляем network 10.x.x.x
Но свой внешний адрес анонсировать нужно (пункт 86)
  
      network 178.207.179.0 mask 255.255.255.248  
     exit
    exit
Ждем маршрут по умолчанию от провайдера (пункт 130)
Команду ниже вводим, если авто-получение не сработало (зависит от версии ПО)
neighbor 178.207.179.1 default-originate 



5 Настройка GRE туннеля до офиса (rtr-a)

    interface Tunnel1
IP адрес туннеля (сеть 10.10.10.0/24, мин. маска /30 -> .1 и .2)
 
     ip address 10.10.10.1 255.255.255.252
От кого строим (наш внешний IP)
 
    tunnel source 178.207.179.4
К кому строим (внешний IP rtr-a из таблицы 83)
 
     tunnel destination 178.207.179.28
     tunnel mode gre ip
Включаем OSPF на туннеле и задаем пароль (пункт 137)
 
     ip ospf message-digest-key 1 md5 P@ssw0rd
    exit
6 Настройка OSPF (динамическая маршрутизация)

    router ospf 1
     router-id 1.1.1.1
Все интерфейсы пассивные по умолчанию (пункт 134)
     
     passive-interface default
Туннель активный (чтобы обмениваться маршрутами с rtr-a)
     
     no passive-interface Tunnel1
Интерфейс к firewall тоже активный (если fw участвует в OSPF)
     
     no passive-interface gigabitethernet 0/1
Объявляем сети

     network 10.10.10.0 0.0.0.3 area 0
     network 10.0.1.0 0.0.0.3 area 0
Включаем аутентификацию для зоны 0 и cохраняем настройки
     
     area 0 authentication message-digest
    exit
    write
    
# после srv1

    enable
    conf t
1 Включаем службу AAA (Authentication, Authorization, Accounting)

    aaa new-model
2 Указываем, где живет сервер RADIUS
IP сервера srv1-cod

    radius-server host 192.168.100.10 key radius_secret
3 Создаем список методов аутентификации "default"
Сначала пробуем group radius, если сервер недоступен - local (локальная база)

    aaa authentication login default group radius local
4 Настраиваем локального пользователя (на случай сбоя RADIUS)
    
    username admin privilege 15 secret P@ssw0rdLocal
В задании сказано: "Для rtr-cod пользователь netuser не должен существовать локально"
Поэтому netuser мы тут НЕ создаем.

5 Применяем защиту на линии (Console и VTY/SSH)
    
    line console 0
     login authentication default
    exit

    line vty 0 4
     login authentication default
     transport input ssh
    exit
    write

# rtr-a (EcoRouter)

    enable
    configure terminal
1 Имя
    
    hostname rtr-a
    domain-name office.ssa2026.region
2 Внешний интерфейс

    interface gigabitethernet 0/0
     description WAN_TO_ISP
     ip address 178.207.179.28 255.255.255.248
     no shutdown
    exit
3 Внутренний интерфейс (VLAN Trunk или роутер на палочке)
В схеме L-2 rtr-a подключен к sw1-a. Обычно шлюзом для VLAN является роутер.
Настроим саб-интерфейсы для VLAN (100, 200, 300)
VLAN 300 (MGMT)

    interface gigabitethernet 0/1.300
     encapsulation dot1Q 300
     ip address 10.10.30.1 255.255.255.0
     no shutdown
    exit
VLAN 200 (CLI)

    interface gigabitethernet 0/1.200
     encapsulation dot1Q 200
     ip address 192.168.200.1 255.255.255.0
     no shutdown
    exit
Подними основной интерфейс

    interface gigabitethernet 0/1
     no shutdown
    exit
4 BGP с провайдером (AS та же 64500, но другой сосед тот же)
ВНИМАНИЕ: Обычно у филиалов своя AS или они не говорят по BGP. 
Но в задании сказано "Настройте... BGP c ISP"
Если AS та же (64500), то это iBGP, но через ISP это странно. 
Скорее всего AS та же, настройки те же, только IP свои.

    router bgp 64500
     bgp router-id 178.207.179.28
     neighbor 178.207.179.25 remote-as 31133
     neighbor 178.207.179.25 description ISP_GW
     address-family ipv4
      neighbor 178.207.179.25 activate
      network 178.207.179.24 mask 255.255.255.248
     exit
    exit
5GRE Туннель (ответная часть)

    interface Tunnel1
     ip address 10.10.10.2 255.255.255.252
     tunnel source 178.207.179.28
     tunnel destination 178.207.179.4
     tunnel mode gre ip
     ip ospf message-digest-key 1 md5 P@ssw0rd
    exit
6 OSPF

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


# sw1-cod (Alt Linux)
1 Задаем имя хоста

    hostnamectl set-hostname sw1-cod.cod.ssa2026.region

2 Создаем Агрегацию (Bond0) для sw1-sw2
Создаем папку интерфейса
    
    mkdir -p /etc/net/ifaces/bond0
Описываем настройки: LACP нет в задании явно, но сказано "active-backup"

    echo "TYPE=bond" > /etc/net/ifaces/bond0/options
    echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
    echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
Указываем, какие физические порты входят в бонд

    echo "ens4" > /etc/net/ifaces/bond0/ports
    echo "ens5" >> /etc/net/ifaces/bond0/ports
Отключаем IP на самом бонде (он будет транком)
    
    echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address
ВАЖНО: Удаляем настройки с физических портов ens4/ens5, если они есть

    rm -rf /etc/net/ifaces/ens4
    rm -rf /etc/net/ifaces/ens5

3 Настройка VLAN управления (VLAN 300)
Интерфейс будет называться vlan300

    mkdir -p /etc/net/ifaces/vlan300
    echo "TYPE=vlan" > /etc/net/ifaces/vlan300/options
    echo "VID=300" >> /etc/net/ifaces/vlan300/options
VLAN "сидит" на интерфейсе bond0 (или на bridge, если мы делаем мост)
Для простоты управления вешаем на bond0

    echo "HOST=bond0" >> /etc/net/ifaces/vlan300/options
Задаем IP управления
    
    echo "10.10.30.11/24" > /etc/net/ifaces/vlan300/ipv4address
    echo "default via 10.10.30.1" > /etc/net/ifaces/vlan300/ipv4route
4 Настройка остальных VLAN (Транки)
В Linux, чтобы он работал как свитч и пропускал трафик сквозь себя,
нужно создавать Bridge (мост) для каждого VLAN или один общий Bridge с vlan-filtering.
Самый простой способ для новичка - Bridge для каждого VLAN.

Пример для VLAN 100 (Servers):
Создаем VLAN интерфейс на входящем порту (bond0)

    mkdir -p /etc/net/ifaces/bond0.100
    echo "TYPE=vlan" > /etc/net/ifaces/bond0.100/options
    echo "VID=100" >> /etc/net/ifaces/bond0.100/options
    echo "HOST=bond0" >> /etc/net/ifaces/bond0.100/options
Создаем VLAN интерфейс на порту, куда подключены серверы (например ens6)

    mkdir -p /etc/net/ifaces/ens6.100
    echo "TYPE=vlan" > /etc/net/ifaces/ens6.100/options
    echo "VID=100" >> /etc/net/ifaces/ens6.100/options
    echo "HOST=ens6" >> /etc/net/ifaces/ens6.100/options
Объединяем их в мост br100
    
    mkdir -p /etc/net/ifaces/br100
    echo "TYPE=bri" > /etc/net/ifaces/br100/options
    echo "bond0.100" > /etc/net/ifaces/br100/ports
    echo "ens6.100" >> /etc/net/ifaces/br100/ports
5 Применяем настройки Проверяем
    
    systemctl restart network
    ip a

# sw2-cod (Alt Linux)
1 Имя хоста

    hostnamectl set-hostname sw2-cod.cod.ssa2026.region
2 Агрегация (Bond0) - Active-Backup

    mkdir -p /etc/net/ifaces/bond0
    echo "TYPE=bond" > /etc/net/ifaces/bond0/options
    echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
    echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
    echo "ens4" > /etc/net/ifaces/bond0/ports
    echo "ens5" >> /etc/net/ifaces/bond0/ports
    echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

Удаляем следы старых интерфейсов
    
    rm -rf /etc/net/ifaces/ens4
    rm -rf /etc/net/ifaces/ens5

3 VLAN управления (VLAN 300)

    mkdir -p /etc/net/ifaces/vlan300
    echo "TYPE=vlan" > /etc/net/ifaces/vlan300/options
    echo "VID=300" >> /etc/net/ifaces/vlan300/options
    echo "HOST=bond0" >> /etc/net/ifaces/vlan300/options
IP адрес управления (другой, не как у sw1!)

    echo "10.10.30.12/24" > /etc/net/ifaces/vlan300/ipv4address
Шлюз по умолчанию нужен только один на устройство, если уже прописал - ок

    echo "default via 10.10.30.1" > /etc/net/ifaces/vlan300/ipv4route

4 Проброс VLAN (Транки) для клиентов
Например, VLAN 400 (CLI)

    mkdir -p /etc/net/ifaces/bond0.400
    echo "TYPE=vlan" > /etc/net/ifaces/bond0.400/options
    echo "VID=400" >> /etc/net/ifaces/bond0.400/options
    echo "HOST=bond0" >> /etc/net/ifaces/bond0.400/options
    systemctl restart network

# srv1-cod (RADIUS и DNS)
Пример настройки интерфейса ens18 (проверь имя через ip a)
    
    mkdir -p /etc/net/ifaces/ens18
    echo "TYPE=eth" > /etc/net/ifaces/ens18/options
Сервер должен быть в VLAN 100. В Linux можно настроить vlan-интерфейс
Но часто серверы подключают в порт свитча, который УЖЕ настроен в access vlan 100.
Если так, то просто ставим IP:

    echo "192.168.100.10/24" > /etc/net/ifaces/ens18/ipv4address
    echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
    systemctl restart network

1 Установка

    apt-get update
    apt-get install freeradius freeradius-utils

2 Добавляем клиентов (роутер и свитчи)
Редактируем файл /etc/raddb/clients.conf (путь может отличаться: /etc/freeradius/3.0/clients.conf)
Добавь в конец файла:

    client rtr-cod {
        ipaddr = 10.10.30.1  # IP роутера (mgmt vlan) или шлюза
        secret = radius_secret # Придумай общий пароль (секрет) для общения устройств
    }

    client sw1-cod {
        ipaddr = 10.10.30.11
        secret = radius_secret
    }
... и так далее для sw2

3 Добавляем пользователя netuser
Редактируем файл /etc/raddb/users (или mods-config/files/authorize)
Добавь в начало файла (перед дефолтными правилами):

    netuser Cleartext-Password := "P@ssw0rd"
           Service-Type = Administrative-User

4 Запуск
    
    systemctl enable --now radiusd (или freeradius)

5 Проверка локально (тест)
    
    radtest netuser P@ssw0rd localhost 0 testing123
Если получил "Access-Accept" - сервер работает!

1 Установка пакетов
    
    apt-get update
    apt-get install bind bind-utils

2 Настройка основного конфига
Открываем /etc/bind/options.conf (или named.conf в зависимости от версии)
Нужно разрешить слушать всех и пересылать запросы на 100.100.100.100
    
    nano /etc/bind/options.conf

Внутри блока options { ... } меняем/добавляем:

    listen-on { any; };           # Слушать на всех интерфейсах
    allow-query { any; };         # Разрешить запросы от всех
    recursion yes;                # Разрешить рекурсию
    # Пункт 150 задания: пересылка на 100.100.100.100
    forwarders { 100.100.100.100; };
    dnssec-validation no;         # Отключаем для простоты

3 Описание зон
Открываем /etc/bind/local.conf (или named.conf) и добавляем:

    nano /etc/bind/local.conf

    zone "cod.ssa2026.region" IN {
        type master;
        file "/var/lib/bind/cod.ssa2026.region.db";
    };

Обратная зона для сети 192.168.100.x (если серверы там)

    zone "100.168.192.in-addr.arpa" IN {
        type master;
        file "/var/lib/bind/100.168.192.db";
    };

4 Создание файла прямой зоны
Создаем файл /var/lib/bind/cod.ssa2026.region.db

    nano /var/lib/bind/cod.ssa2026.region.db

    $TTL 86400
    @   IN  SOA     srv1-cod.cod.ssa2026.region. root.cod.ssa2026.region. (
            2026012801 ; Serial
            3600       ; Refresh
            1800       ; Retry
            604800     ; Expire
            86400 )    ; Minimum TTL

    @       IN  NS      srv1-cod.cod.ssa2026.region.
    @       IN  A       192.168.100.10  ; IP этого сервера (srv1)

    ; Записи для устройств [cite: 146]
    srv1-cod IN  A       192.168.100.10
    srv2-cod IN  A       192.168.100.11
    fw-cod   IN  A       192.168.100.1
    rtr-cod  IN  A       10.10.30.1
    sw1-cod  IN  A       10.10.30.11
    sw2-cod  IN  A       10.10.30.12
    sip-cod  IN  A       192.168.100.20
    admin-cod IN A       10.10.30.100
    ; Добавь сюда monitoring (для Zabbix)
    monitoring IN CNAME  srv1-cod

5 Проверка и запуск Проверка: dig @localhost srv1-cod.cod.ssa2026.region

    named-checkconf
    systemctl enable --now bind

1 Создаем структуру папок (строго по заданию /var/ca)

    mkdir -p /var/ca/{certs,crl,newcerts,private}
    chmod 700 /var/ca/private
    touch /var/ca/index.txt
    echo 1000 > /var/ca/serial

2 Копируем конфиг OpenSSL

    cp /etc/ssl/openssl.cnf /var/ca/openssl.cnf

3 Редактируем конфиг

    nano /var/ca/openssl.cnf

Ищем секцию [ CA_default ] и меняем пути:

    dir             = /var/ca
    certificate     = $dir/ca.crt
    private_key     = $dir/private/ca.key
    new_certs_dir   = $dir/newcerts
    database        = $dir/index.txt
    serial          = $dir/serial
    default_days    = 1825    ; Пункт 157: Срок жизни 5 лет

В секции [ req_distinguished_name ] можно задать дефолты:

    countryName_default             = RU        ; Пункт 158
    0.organizationName_default      = IRPO      ; Пункт 159
    commonName_default              = ssa2026   ; Пункт 160

4 Генерируем корневой ключ и сертификат

    cd /var/ca
Генерируем ключ (пароль попросят ввести - запомни его!)

    openssl genrsa -out private/ca.key 4096

Создаем сам сертификат (Self-signed)

    openssl req -new -x509 -key private/ca.key -out ca.crt -days 1825 -config openssl.cnf
На вопросы отвечай: RU, IRPO, ssa2026 (Common Name)

5 Распространение доверия (чтобы сам сервер верил своему CA)

    cp ca.crt /usr/share/ca-certificates/ssa2026.crt
    update-ca-trust


# после srv2
1 Установка iSCSI инициатора

    apt-get install open-iscsi lvm2 nfs-utils

2 Задаем имя инициатора (Client Name)

    echo "InitiatorName=iqn.2026-01.region.ssa2026.cod:iscsi" > /etc/iscsi/initiatorname.iscsi
    systemctl restart iscsid

3 Подключение к таргету
IP адрес srv2-cod в сети DATA (VLAN 200). Допустим 192.168.200.11

    iscsiadm -m discovery -t st -p 192.168.200.11
    iscsiadm -m node --login

Проверка: lsblk (должен появиться новый диск, например sdb)

4 Настройка LVM и файловой системы [cite: 183-187]
Создаем физический том

    pvcreate /dev/sdb
Создаем группу томов VG

    vgcreate VG /dev/sdb
Создаем логический том DATA на 100% места

    lvcreate -l 100%FREE -n DATA VG
Форматируем в XFS
    
    mkfs.xfs /dev/VG/DATA

5 Монтирование

    mkdir -p /opt/data
Узнаем UUID диска

    blkid /dev/VG/DATA
Добавляем в /etc/fstab (замени UUID на свой):
UUID="xxxx-xxxx" /opt/data xfs _netdev 0 0
    
    mount -a

6 Настройка NFS сервера [cite: 190-192]
Редактируем /etc/exports
    
    nano /etc/exports

Добавляем строку:
/opt/data   10.10.30.0/24(rw,sync,no_root_squash)
Где 10.10.30.0/24 - это сеть MGMT-COD

Применяем

    exportfs -ra
    systemctl enable --now nfs-server

# srv2-cod (Alt Linux)
1 Установка

    apt-get install targetcli
    systemctl enable --now target

2 Настройка через консоль targetcli
    
    targetcli

Внутри консоли вводим команды:
Создаем диск-хранилище

    /backstores/block create name=disk1 dev=/dev/sdb

Создаем цель (Target) с именем по заданию (месяц 01)
    
    /iscsi create iqn.2026-01.region.ssa2026.cod:data.target

Создаем ACL (разрешаем доступ клиенту srv1)
Мы назовем клиента просто "iscsi", как просят в задании для srv1

    /iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/acls create iqn.2026-01.region.ssa2026.cod:iscsi

Привязываем диск к цели

    /iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/luns create /backstores/block/disk1

Сохраняем и выходим
    
    exit
