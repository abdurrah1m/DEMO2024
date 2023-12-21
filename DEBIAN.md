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
dns bind https://timeweb.cloud/tutorials/ubuntu/nastrojka-dns-servera-bind  

# Модуль 1 задание 1

Выполните базовую настройку всех устройств:  
a. Присвоить имена в соответствии с топологией  
b. Рассчитайте IP-адресацию IPv4 и IPv6. Необходимо заполнить таблицу №1, чтобы эксперты могли проверить ваше рабочее место.  
c. Пул адресов для сети офиса BRANCH - не более 16  
d. Пул адресов для сети офиса HQ - не более 64  

|Имя устройства |Интерфейс |Ip-адрес |Маска/Префикс |Шлюз |
|:-:|:-:|:-:|:-:|:-:|
|ISP|ens18|10.10.201.174|/24 255.255.255.0|10.10.201.254|
||ens19|11.11.11.1|/30 255.255.255.252|              |
||ens20|22.22.22.1|/30 255.255.255.252||
|BR-R|ens18|22.22.22.2|/30 255.255.255.252|22.22.22.1|
||ens19|192.168.100.1|/27 255.255.255.224||
|HQ-R|ens18|11.11.11.2|/30 255.255.255.252|11.11.11.1|
||ens19|192.168.0.1|/25 255.255.255.128||
|BR-SRV|ens18|192.168.100.2|/27 255.255.255.224|192.168.100.1|
|HQ-SRV|ens18|192.168.0.2|/25 255.255.255.128|192.168.0.1|

Настройка зеркал во всех машинах `nano /etc/apt/sources.list`
```
deb http://mirror.yandex.ru/debian/ stable main contrib non-free
```
### IP-адрес машин
```
nano /etc/network/interfaces
```
HQ-SRV:
```
auto ens18
iface ens18 inet static
address 192.168.0.2/25
gateway 192.168.0.1
```
BR-SRV:
```
auto ens18
iface ens18 inet static
address 192.168.100.2/27
gateway 192.168.100.1
```
HQ-R:
```
auto ens18
iface ens18 inet static
address 11.11.11.2/30
gateway 11.11.11.1
auto ens19
iface ens19 inet static
address 192.168.0.1/25
```
BR-R:
```
auto ens18
iface ens18 inet static
address 22.22.22.2/30
gateway 22.22.22.1
auto ens19
iface ens19 inet static
address 192.168.100.1/27
```
ISP:
```
auto ens18
iface ens18 inet static
address 10.10.201.174/24
gateway 10.10.201.254
auto ens19
iface ens19 inet static
address 11.11.11.1/30
auto ens20
iface ens20 inet static
address 22.22.22.1/30
```
```
systemctl restart networking.service
```


# NAT на ISP, HQ-R,BR-R
```
nano /etc/sysctl.conf
```
```
net.ipv4.ip_forward=1
```
```
sysctl -p
```
Отключить NetworkManager:
```
systemctl disable NetworkManager
```
Установка firewalld:
```
apt-get update && apt-get -y install firewalld && systemctl enable --now firewalld
```
Правила к исходящим пакетам:
```
firewall-cmd --permanent --zone=public --add-interface=ens18
```
Правила к входящим пакетам:
```
firewall-cmd --permanent --zone=trusted --add-interface=ens19
```
Включение NAT:
```
firewall-cmd --permanent --zone=public --add-masquerade
```
Сохранение правил:
```
firewall-cmd --reload
```

***

# Модуль 1 задание 2

Настройте внутреннюю динамическую маршрутизацию по средствам FRR. Выберите и обоснуйте выбор протокола динамической маршрутизации из расчёта, что в дальнейшем сеть будет масштабироваться.  
a. Составьте топологию сети L3.  


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

# Модуль 1 задание 3

Настройте автоматическое распределение IP-адресов на роутере HQ-R.
a. Учтите, что у сервера должен быть зарезервирован адрес. 

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

***

# Модуль 1 задание 4

Настройте локальные учётные записи на всех устройствах в соответствии с таблицей.

|Учётная запись|Пароль|Примечание|
|:-:|:-:|:-:|
|Admin|P@ssw0rd|CLI, HQ-SRV, HQ-R|
|Branch admin|P@ssw0rd|BR-SRV, BR-R|
|Network admin|P@ssw0rd|HQ-R, BR-R, BR-SRV|

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

# Модуль 1 задание 5

Измерьте пропускную способность сети между двумя узлами HQ-R-ISP по средствам утилиты iperf 3. Предоставьте описание пропускной способности канала со скриншотами.  

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
# №2.1 DNS-сервер на HQ-SRV
|Имя|Тип записи|Адрес|
|:-:|:-:|:-:|
|hq-r.hq.work|A, PTR|192.168.0.1|
|hq-srv.hq.work|A, PTR|192.168.0.20|

Зона branch.work
|Имя|Тип записи|Адрес|
|:-:|:-:|:-:|
|br-r.branch.work|A, PTR|192.168.0.129|
|br-srv.branch.work|A|192.168.0.140|

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
#принимать запросы от всех
listen-on-v6 port 53 { none; };
#не слушать ipv6
forwarders { 8.8.8.8; };
#обработка dns-запросов следующим сервером после HQ-SRV
listen-on {    192.168.0.166/32;
192.168.0.20;
127.0.0.0/8; };
#прослушивать с ""
};
```
Проверка настроек:
```
named-checkconf
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
    file "/opt/dns/demob.db";
};

zone "0.168.192.in-addr.arpa.zone" {
    type master;
    allow-transfer { any; };
    file "/opt/dns/0.168.192.in-addr.arpa.zone";
};
```
Первая зона `nano /opt/dns/demo.db`:  
  
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/7c92719c-631d-41aa-b334-d4eb0a12bde4)  
Корректность зоны:
```
named-checkzone hq.work /opt/dns/demo.db
```

Вторая зона `nano /opt/dns/demob.db`:  
  
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/84e554ee-be10-435f-af96-00b63562fee5)  
Корректность зоны:
```
named-checkzone branch.work /opt/dns/demob.db
```

Обратная зона `/opt/dns/0.168.192.in-addr.arpa.zone`:  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/5a36f78c-ecad-4ad6-b908-a1f11f9cd9b6)  
Корректность зоны:
```
named-checkzone 0.168.192.in-addr.arpa.zone /opt/dns/0.168.192.in-addr.arpa.zone
```
Перезагружаем:
```
service named restart
service apparmor restart
```

Проверка с `HQ-SRV`:  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/93d3d991-0ed6-481b-9baa-05808c0af368)

Проверка с `BR-SRV`:  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/7232378f-6bf5-478c-b3c3-97851950bafd)
