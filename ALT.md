# №1 Сеть на подсети
## HQ-SRV
Смотрим название адаптера
```
ip a
```
`ens18`  
ip  
```
echo 192.168.0.40/25 > /etc/net/ifaces/ens18/ipv4address
```
gateway  
```
echo default via 192.168.0.1 > /etc/net/ifaces/ens18/ipv4route
```
Параметр BOOTPROTO отвечает за способ получения сетевой картой адреса, `static`  
```
nano /etc/net/ifaces/ens18/options
```
```
BOOTPROTO=static
TYPE=eth
NM_CONTROLLED=yes
DISABLED=no
CONFIG_IPV4=yes
```
```
nano /etc/resolv.conf
```
```
nameserver 8.8.8.8
```
Перезагрузка:
```
service network restart
```
или
```
systemctl restart network.service
```

# NAT ISP,HQ-R,BR-R:
```
iptables -A POSTROUTING -t nat -j MASQUERADE
```
```
iptables-save
```
```
nano /etc/net/scripts/nat
```
```
#!/bin/sh
/sbin/iptables -A POSTROUTING -t nat -j MASQUERADE
```
```
chmod +x /etc/net/scripts/nat
```
# №1.2 FRR HQ-R,BR-R,ISP
```
apt-get -y install frr
```
```
nano /etc/frr/daemons
```
```
ospfd=yes
```
```
systemctl restart frr
```
```
vtysh
```
```
sh in br
```
|Interface|Status|VRF|Adresses|
|:-:|:-:|:-:|:-:|
|ens18|up|default|192.168.0.162/30|
|ens19|up|default|192.168.0.129/27|
|lo|up|default||
```
router ospf
```
```
net 192.168.0.160/30 area 0
net 192.168.0.128/27 area 0
```
```
do sh ip ospf neighbor
```
СОХРАНИТЬ КОНФИГИ:
```
do w
```
# № 1.3 DHCP HQ-R
```
apt-get -y install dhcp-server
```
`/etc/sysconfig/dhcpd`, указываю интерфейс внутренней сети:
```
DHCPDARGS=ens19
```
```
cp /etc/dhcp/dhcpd.conf.sample /etc/dhcp/dhcpd.conf
```
`/etc/dhcp/dhcpd.conf` параметры раздачи:
```
ddns-update-style-none;

subnet 192.168.0.0 netmask 255.255.255.128 {
        option routers                  192.168.0.1;
        option subnet-mask              255.255.255.128;
        option domain-name-servers      8.8.8.8, 8.8.4.4;

        range dynamic-bootp 192.168.0.20 192.168.0.50;
        default-lease-time 21600;
        max-lease-time 43200;
}
```
```
systemctl restart dhcpd
```
```
systemctl status dhcpd.service
```
```
chkconfig dhcpd on
service dhcpd start
```
HQ-SRV
```
nano /etc/net/ifaces/ens18/ipv4address
```
```
#192.168.0.40
```
```
nano /etc/net/ifaces/ens18/options
```
```
BOOTROTO=dhcp
TYPE=eth
NM_CONTROLLED=yes
DISABLED=no
CONFIG_IPV4=yes
```
```
service network restart
```
```
ens18:
    inet 192.168.0.38/25 brd 192.168.0.127
```
# №1.4 Создание пользователей
Настроить
|Учётная запись|Пароль|Примечание|
|:-:|:-:|:-:|
|Admin|P@ssw0rd|CLI, HQ-SRV|
|Branch admin|P@ssw0rd|BR-SRV, BR-R|
|Network admin|P@ssw0rd|HQ-R, BR-R, HQ-SRV|

Пользователь `admin` на `HQ-SRV`
```
adduser admin
```
```
usermod -aG root admin
```
```
passwd admin
P@ssw0rd
P@ssw0rd
```
```
nano /etc/passwd
```
```
admin:x:0:501::/home/admin:/bin/bash
```
# №1.5 IPERF3 HQ-R ISP
```
apt-get -y install iperf3
```
ISP как сервер:
> если надо открыть порт
>```
>iptables -A INPUT -p tcp --dport 5201 -j ACCEPT
>```
```
iperf3 -s
```
HQ-R:
```
iperf3 -c 192.168.0.161 -f M
```
```
[ID] Interval      Transfer   Bitrate        Retr Cwnd
[ 5] 0.00-1.00 sec 345 MBytes 344 MBytes/sec    0 538 KBytes
[ 5] 1.00-2.00 sec 338 MBytes 338 MBytes/sec    0 676 KBytes
[ 5] 3.00-4.00 sec 341 MBytes 341 MBytes/sec    0 749 KBytes
```
# №1.6 UrBackup BR-R,HQ-R
UrBackup - система резервного копирования типа "клиент-сервер"
ISP как сервер:
```
apt-get -y install urbackup-server
```
```
mkdir -p /mnt/backups
```
Каталог принадлежит пользователю urbackup:
```
chown -R urbackup:urbackup /mnt/backups
```
```
systemctl enable --now urbackup-server
```
> urbackup прослушивает порты 55413 и 55414

Есть веб-интерфейс `<IP-сервера:55414>`
Установка на клиентах BR-R,HQ-R:
```
apt-get -y install urbackup-client
```
```
systemctl enable --now urbackup-client
```
Клиенты

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/e5df3a40-996b-4d0f-9559-90757d9a7d4b)  

В настройках меняются:
* Количество и интервал инкрементальных файловых бэкапов
* Количество и интервал полных файловых бэкапов
* Количество и интервал полных образов
* Количество и интервал инкрементальных образов
* Тома и каталоги  
Выбираем каталог `/etc/frr` и сохраняем
-------------------------------------
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/1ead826e-782a-4693-a0ce-c81a51102ea9)  

Теперь можно делать бэкапы

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/dc59516b-0280-4012-a19e-7b2e8a2c2ab5)