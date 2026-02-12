Да, srv1-cod и srv2-cod тоже требуют настройки сети вручную, так как DHCP в серверном сегменте обычно не используется, а порты коммутаторов для них настроены особым образом (Trunk/Tagged).
Поскольку в конфигурации коммутаторов (sw1-cod и sw2-cod) порты, идущие к этим серверам, передают тегированный трафик (VLAN 100 и VLAN 200), на самих серверах нужно создавать VLAN-интерфейсы.
Вот настройки сети для обоих серверов согласно схеме и DNS-записям из вашего файла README.md.
1. 🐧 srv1-cod (DNS, Zabbix, RADIUS)
Этот сервер подключен к sw2-cod.
 * IP (VLAN 100 - SRV): 192.168.100.10 (Основной, шлюз на FW)
 * IP (VLAN 200 - DATA): 192.168.200.10 (Для iSCSI, без шлюза)
Команды для настройки:
# 1. Имя хоста
hostnamectl set-hostname srv1-cod.cod.ssa2026.region

# 2. Отключаем NetworkManager
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable --now network

# 3. Чистим интерфейсы
rm -rf /etc/net/ifaces/*
mkdir -p /etc/net/ifaces/lo
echo "TYPE=loopback" > /etc/net/ifaces/lo/options
echo "127.0.0.1/8" > /etc/net/ifaces/lo/ipv4address

# 4. Физический интерфейс (проверьте имя через ip a, обычно ens18 или ens3)
mkdir -p /etc/net/ifaces/ens18
echo "TYPE=eth" > /etc/net/ifaces/ens18/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens18/options

# 5. VLAN 100 (Сервисная сеть)
mkdir -p /etc/net/ifaces/ens18.100
echo "TYPE=vlan" > /etc/net/ifaces/ens18.100/options
echo "VID=100" >> /etc/net/ifaces/ens18.100/options
echo "HOST=ens18" >> /etc/net/ifaces/ens18.100/options
# Настройки IP
echo "192.168.100.10/24" > /etc/net/ifaces/ens18.100/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18.100/ipv4route
# DNS указывает сам на себя (127.0.0.1), т.к. это DNS-сервер
echo "nameserver 127.0.0.1" > /etc/resolv.conf

# 6. VLAN 200 (Сеть хранения данных - iSCSI)
mkdir -p /etc/net/ifaces/ens18.200
echo "TYPE=vlan" > /etc/net/ifaces/ens18.200/options
echo "VID=200" >> /etc/net/ifaces/ens18.200/options
echo "HOST=ens18" >> /etc/net/ifaces/ens18.200/options
# IP для iSCSI инициатора
echo "192.168.200.10/24" > /etc/net/ifaces/ens18.200/ipv4address

# Применяем
systemctl restart network

2. 🐧 srv2-cod (PostgreSQL, iSCSI Target)
Этот сервер подключен к sw1-cod.
 * IP (VLAN 100 - SRV): 192.168.100.11
 * IP (VLAN 200 - DATA): 192.168.200.11
Команды для настройки:
# 1. Имя хоста
hostnamectl set-hostname srv2-cod.cod.ssa2026.region

# 2. Отключаем NetworkManager
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable --now network

# 3. Базовая настройка
rm -rf /etc/net/ifaces/*
mkdir -p /etc/net/ifaces/lo
echo "TYPE=loopback" > /etc/net/ifaces/lo/options
echo "127.0.0.1/8" > /etc/net/ifaces/lo/ipv4address

# 4. Физический интерфейс (напр. ens18)
mkdir -p /etc/net/ifaces/ens18
echo "TYPE=eth" > /etc/net/ifaces/ens18/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ens18/options

# 5. VLAN 100 (Сервисная сеть)
mkdir -p /etc/net/ifaces/ens18.100
echo "TYPE=vlan" > /etc/net/ifaces/ens18.100/options
echo "VID=100" >> /etc/net/ifaces/ens18.100/options
echo "HOST=ens18" >> /etc/net/ifaces/ens18.100/options
# Настройки IP
echo "192.168.100.11/24" > /etc/net/ifaces/ens18.100/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18.100/ipv4route
# DNS указывает на srv1
echo "nameserver 192.168.100.10" > /etc/resolv.conf

# 6. VLAN 200 (Сеть хранения данных - iSCSI Target)
mkdir -p /etc/net/ifaces/ens18.200
echo "TYPE=vlan" > /etc/net/ifaces/ens18.200/options
echo "VID=200" >> /etc/net/ifaces/ens18.200/options
echo "HOST=ens18" >> /etc/net/ifaces/ens18.200/options
# IP для iSCSI таргета
echo "192.168.200.11/24" > /etc/net/ifaces/ens18.200/ipv4address

# Применяем
systemctl restart network

Важные нюансы:
 * Проверьте имя интерфейса: Введите команду ip a. Если ваш интерфейс называется ens3 или eth0, замените ens18 в командах выше на ваше название.
 * VLAN 200: В задании сказано, что трафик VLAN DATA (200) не должен маршрутизироваться, поэтому мы не указываем шлюз (default via) для интерфейсов .200. Серверы будут общаться по iSCSI напрямую внутри этого VLAN.
 * Доступность: После настройки srv1 должен пинговать шлюз 192.168.100.1 (если FW настроен) и соседа 192.168.100.11 (srv2).
