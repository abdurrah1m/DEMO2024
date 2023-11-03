<!--
<b> </b> - полужирный
<i> </i> - курсив
<blockquote>fhaufwhuhf</blockquote> - цитаты
https://html5book.ru/html-tags/ HTML-ТЕГИ
<code> </code> - МОНОШИРИННЫЙ ТЕКСТ

-->nat debian https://quaded.com/nat-%D0%B2-debian/  
SSH https://routerus.com/how-to-setup-ssh-tunneling/  
Включение ssh https://routerus.com/how-to-setup-ssh-tunneling/  
iptables https://www.dmosk.ru/instruktions.php?object=iptables-settings  
dns https://habr.com/ru/articles/713156/  

# NANO
<kbd>ALT</kbd> + <kbd>/</kbd> Перейти в конец файла  
<kbd>CTRL</kbd> + <kbd>SHIFT</kbd> Скопировать  
<kbd>CTRL</kbd> + <kbd>U</kbd> Вставить  
<kbd>CTRL</kbd> + <kbd>K</kbd> Вырезать  

# Сеть на подсети

|Имя устройства |Интерфейс |Ip-адрес |Маска/Префикс |Шлюз |
|:-:|:-:|:-:|:-:|:-:|
|ISP|ens18|10.10.201.100|/24 255.255.255.0|10.10.201.254|
||ens19|192.168.0.165|/30 255.255.255.252|              |
||ens20|192.168.0.161|/30 255.255.255.252||
|BR-R|ens18|192.168.0.129|/27 255.255.255.224||
||ens19|192.168.0.162|/30 255.255.255.252|192.168.0.161|
|HQ-R|ens18|192.168.0.1|/25 255.255.255.128||
||ens19|192.168.0.166|/30 255.255.255.252|192.168.0.165|
|BR-SRV|ens18|192.168.0.130|/27 255.255.255.224|192.168.0.129|
|HQ-SRV|ens18|192.168.0.2|/25 255.255.255.128|192.168.0.1|

Настройка зеркал во всех машинах `nano /etc/apt/sources.list`
```
deb http://mirror.yandex.ru/debian/ stable main contrib non-free
```
IP-адрес машин
```
nano /etc/network/interfaces
```
```
auto ens19
iface ens19 inet static
address 10.10.201.100
gateway 10.10.201.254
netmask 255.255.255.0
```
```
systemctl restart networking.service
```
# NAT на ISP, HQ-R,BR-R
```
apt install iptables
```
```
nano /etc/sysctl.conf
```
```
net.ipv4.ip_forward=1
```
```
sysctl -p
```
```
iptables -A POSTROUTING -t nat -j MASQUERADE
```
```
nano /etc/network/if-pre-up.d/nat
```
```
#!/bin/sh
/sbin/iptables -A POSTROUTING -t nat -j MASQUERADE
```
```
chmod +x /etc/network/if-pre-up.d/nat
```
dhcp debian https://setiwik.ru/kak-nastroit-dhcp-server-v-debian-11/?ysclid=lo4mpr8jcj791182994  

# №1.2 FRR OSPF(ISP,HQ-R,BR-R)
Установка frr
```
apt update
apt install frr
```
```
nano /etc/frr/daemons
```
Вместо `ospfd=no на:`
```
ospfd=yes
```
```
systemctl restart frr
```
`vtysh` вход в среду роутера
```
sh int br
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/ca535ac1-459f-4acc-9fcb-83950433a26c)
```
conf t
```
```
router ospf
```
```
net 192.168.0.0/25 area 0
net 192.168.0.164/30 area 0
```
```
sh ip ospf neighbor
```
![изображение](https://github.com/abdurrah1m/DEMO2024/assets/148451230/fad7b59e-76ac-4052-abd0-56df80e51617)

С HQ-SRV ДО BR-SRV `ping 192.168.0.130`  


# №1.3 DHCP SERVER на HQ-R

Установка DHCP
```
apt update
```
```
apt install isc-dhcp-server
```
Конфиг 
```
nano /etc/default/isc-dhcp-server  
```
Указываю интерфейс, который в сторону Интернета 
```
INTERFACESV4="ens19"
```
Настройки раздачи адресов  
```
nano /etc/dhcp/dhcpd.conf
```

```
subnet 192.168.0.0 netmask 255.255.255.0 {
range 192.168.0.2 192.168.0.125;
option domain-name-servers 8.8.8.8, 8.8.4.4;
option routers 192.168.0.1;
}
```
Применяю изменения
```
systemctl restart isc-dhcp-server.service
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
P@ssw0rd
```
```
usermod -aG sudo admin
```
Проверка
```
getent group sudo
```
```
sudo:x:27:admin
```

# №1.5 iperf3
HQ-R И ISP `apt install iperf3`  
ISP `iperf3 -s -p 6869`  
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/21d9e3c2-8d73-4b69-8615-5d2fb2a2b3b1)

# №1.6 Rsync бэкап конфигов
Установка `rsync` на HQ-R и BR-R
```
apt install rsync -y
```
Создание директории в машинах
```
mkdir /etc/networkbackup
```
Скрипт `crontab -e`
```
0 0 * * * rsync -avzh /etc/frr/frr.conf /etc/networkbackup
```
Проверка скрипта `40 15 * * * rsync -avzh /etc/frr/frr.conf /etc/networkbackup`
```
ls /etc/networkbackup/frr.conf
```

```
frr version 8.4.4
frr deafults tradirional
hostname HQ-R
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
```
# №2 DNS-сервер на HQ-SRV
На `HQ-SRV`:
```
apt install bind9
mkdir /opt/dns
cp /etc/bind/db.local /opt/dns/demo.db
chown -R bind:bind /opt/dns
```
Добавляем строчку в `nano /etc/apparmor.d/usr.sbin.named`:
```
/opt/dns/** rw,
```
`nano /etc/bind/named.conf.options`:
```
allow-query { any; };
```
`nano /etc/bind/named.conf.default-zones`:
```
zone "hq.work" {
    type master;
    allow-transfer { any; };
    file "/opt/dns/demo.db";
};

zone "branch.work" {
    type master;
    allow-transfer { any; };
    file "/opt/dns/demo1.db";
};

zone "0.168.192.in-addr.arpa" {
    type master;
    allow-transfer { any; };
    file "/opt/dns/back.0.168.192.db";
};

zone "128.168.192.work" {
    type master;
    allow-transfer { any; };
    file "/opt/dns/back.128.168.192.db";
};
```
