https://sysahelper.gitbook.io/sysahelper/main/complex_works/main/demo2024  
https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/index.html  
# №1.1 Сеть на подсети
|Имя устройства |Интерфейс |Ip-адрес |Маска/Префикс |Шлюз |
|:-:|:-:|:-:|:-:|:-:|
|ISP|ens18|10.10.201.174|/24 255.255.255.0|10.10.201.254|
||ens19|192.168.0.161|/30 255.255.255.252|              |
||ens20|192.168.0.165|/30 255.255.255.252||
|BR-R|ens18|192.168.0.162|/30 255.255.255.252|192.168.0.161|
||ens19|192.168.100.1|/27 255.255.255.224||
|HQ-R|ens18|192.168.0.166|/30 255.255.255.252|192.168.0.165|
||ens19|192.168.0.1|/25 255.255.255.128||
|BR-SRV|ens18|192.168.100.2|/27 255.255.255.224|192.168.100.1|
|HQ-SRV|ens18|192.168.0.2|/25 255.255.255.128|192.168.0.1|

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/3412d3b9-7b67-4fc1-8cc9-9b0ac66ff7dc)

## HQ-SRV
Смотрим название адаптера:
```
ip -с a
```
> -c выводит информацию с цветом

`ens18`  
Настройка ip-адреса:  
```
echo 192.168.0.40/25 > /etc/net/ifaces/ens18/ipv4address
```
Настройка шлюза по умолчанию: 
```
echo default via 192.168.0.1 > /etc/net/ifaces/ens18/ipv4route
```
Параметры интерфейса:
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
DNS-сервер:
```
echo nameserver 8.8.8.8 > /etc/resolv.conf
```
Создание нового интерфейса (предположительно свериться в ip a):
```
cp /etc/net/ifaces/ens18 /etc/net/ifaces/ens19
```
Перезагрузка адаптера:
```
service network restart
```
или
```
systemctl restart network.service
```
# NAT с помощью firewalld ISP,HQ-R,BR-R:
Настройки интерфейсов должны быть такими:
```
NM_CONTROLLED=no
DISABLED=no
```
Установка firewalld:
```
apt-get -y install firewalld
```
Автозагрузка:
```
systemctl enable --now firewalld
```
Правила к исходящим пакетам:
```
firewall-cmd --permanent --zone=public --add-interface=ens33
```
Правила к входящим пакетам:
```
firewall-cmd --permanent --zone=trusted --add-interface=ens34
```
Включение NAT:
```
firewall-cmd --permanent --zone=public --add-masquerade
```
Сохранение правил:
```
firewall-cmd --reload
```
# NAT 2 способ ISP,HQ-R,BR-R:
Правило:
```
iptables -A POSTROUTING -t nat -j MASQUERADE
```
Применение правил, работает только до перезагрузки:
```
iptables-save
```
Сохранение правил:
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
Автозагрузка:
```
service iptables enable
```
# NAT 3 способ ISP,HQ-R,BR-R:
Вариант настройки через iptables: Выполните команды (предполагается, что внешний интерфейс носит имя enp1s0):
iptables -t nat -A POSTROUTING -s 192.0.2.0/24 -o enp1s0 -j SNAT --to-source 198.51.100.234
iptables -A INPUT -i wlp3s2 -j ACCEPT
iptables -A FORWARD -i wlp3s2 -o enp1s0 -j ACCEPT
iptables-save > /etc/sysconfig/iptables

# №1.2 FRR HQ-R,BR-R,ISP
Установка пакета:
```
apt-get -y install frr
```
Автозагрузка:
```
systemctl enable --now frr
```
Включение демона службы ospf:
```
nano /etc/frr/daemons
```
```
ospfd=yes
```
```
systemctl restart frr
```
Вход в среду роутера:
```
vtysh
```
Показать интерфейсы:
```
sh in br
```
|Interface|Status|VRF|Adresses|
|:-:|:-:|:-:|:-:|
|ens18|up|default|192.168.0.162/30|
|ens19|up|default|192.168.0.129/27|
|lo|up|default||

Активировать ospf:
```
router ospf
```
Вводим СЕТИ:
```
net 192.168.0.160/30 area 0
net 192.168.0.128/27 area 0
```
Показать соседей:
```
do sh ip ospf neighbor
```
СОХРАНИТЬ КОНФИГИ:
```
do w
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/a39631c1-a683-47d2-a63a-4bbb93d7556a)

# № 1.3 DHCP HQ-R
Установка пакета:
```
apt-get -y install dhcp-server
```
`/etc/sysconfig/dhcpd`, указываю интерфейс внутренней сети:
```
DHCPDARGS=ens19
```
Копирую образец:
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
Автозагрузка:
```
chkconfig dhcpd on
service dhcpd start
```
HQ-SRV (клиент):
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
# №1.6 crontab BR-R,HQ-R
Заход в планировщик заданий:
```
EDITOR=nano crontab -e
```
минута | час | день | месяц | день недели | "команда, например `reboot`":
```
9 15 * * * cp /etc/frr/frr.conf /etc/networkbackup
```
```
ls /etc/networkbackup
```
```
frr.conf
```
# №1.6 UrBackup BR-R,HQ-R
UrBackup - система резервного копирования типа "клиент-сервер"  
ISP выступает в роли сервера:
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
# №1.7 SSH
HQ-SRV:
```
apt-get -y install openssh-server
```
```
systemctl enable --now sshd
```
```
nano /etc/openssh/sshd_config
```
```
Port 2222
PermitRootLogin no
PasswordAuthentication yes
```
Подключение
```
ssh student@192.168.0.40 -p 2222
```
# №1.8 SSH
HQ-SRV:
```
nano /etc/openssh/sshd_config
```
Выбор пользователей
```
AllowUsers student@192.168.0.1 student@192.168.0.140 student@192.168.0.129 student@10.10.201.174
```

# Сервер точного времени HQ-R
Переставить часовой пояс на вех машинах:
```
timedatectl set-timezone Asia/Yekaterinburg
```
Установить `chrony` на всех устройствах:
```
apt-get install -y chrony
```
Автозагрузка:
```
systemctl enable --now chronyd
```
Конфигурация `HQ-R`:
```
nano /etc/chrony.conf
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/7cc4f29b-f180-40d9-bdde-fe031658417c)

Конфигурация на клиентах (в зависимости от сети):
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/d0115300-bf60-47ed-ad7d-3f3beabd438f)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/06ea5432-dcef-43d8-b20e-a14afb36c0d3)

Просмотр клиентов:
```
chronyc clients
```
# №2 DNS HQ-SRV
https://sysahelper.gitbook.io/sysahelper/main/linux_admin/main/altdnsserversetup  
Установка:
```
apt-get update && apt-get install bind -y
```
```
nano /var/lib/bind/etc/options.conf
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/96dc68eb-413b-4438-89e4-f377ae9dc9bd)

```
systemctl enable --now bind
```
```
nano /etc/resolv.conf
```
```
search hq.work
search branch.work
nameserver 127.0.0.1
```
```
nano /var/lib/bind/etc/local.conf
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/42494756-a69b-4d00-adb6-2b80b4ac5cc7)

```
cp /var/lib/bind/etc/zone/{localdomain,hq.db}
```
```
chown named. /var/lib/bind/etc/zone/champ.db
chmod 600 /var/lib/bind/etc/zone/champ.db
```
```
nano /var/lib/bind/etc/zone/hq.db
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/6f3af999-1b05-47b1-a522-9fb760c0eabe)

```
nano /var/lib/bind/etc/zone/branch.db
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/5d60448f-0bc0-4e64-b8a9-615173d88fa0)

```
nano /var/lib/bind/etc/zone/0.db
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/9e079573-5137-41cc-a6d9-65b57e1acbd2)

Проверка:
```
named-checkconf -z
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/4d99eca1-66d6-46f6-a156-d43dd1f0f1ca)


```
systemctl restart bind
```
```
nslookup hq-r.hq.work
```
![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/ab4b1341-ec1c-46cf-ae02-a261528c8f53)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/c0f0a401-2bbf-430a-8b4c-ea683fd20048)

# №2.1 SambaDC admc
Перед установкой отключить конфликтующие службы krb5kdc, slapd, bind:
```
systemctl stop krb5kdc
systemctl disable krb5kdc
systemctl status krb5kdc
```
Установка:
```
apt-get install -y task-samba-dc admc
```
Имя узла и домена
```
hostnamectl set-hostname hq-srv.hq.work; exec bash
```
```
domainname hq.work
```
Для корректного распознования dns-запросов, `/etc/resolv.conf`:
```
nameserver 127.0.0.1
```
```
resolvconf -u
```
Начальное состояние samba:
```
rm -f /etc/samba/smb.conf
```
```
rm -rf /var/lib/samba
```
```
rm -rf /var/cache/samba
```
```
mkdir -p /var/lib/samba/sysvol
```
Через веб-браузер входим в среду настройки домена:
```
192.168.0.40:8080
```
Вот такие настройки

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/1e8abedf-b249-4cea-b2a6-8549c6d5d0d1)

ОБЯЗАТЕЛЬНО ПЕРЕЗАГРУЗИТЬ МАШИНУ  
При входе получаем билет

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/e6dd4177-c23e-4d73-a79c-09e147c5b425)

Просмотр полученного билета

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/65372d90-bd96-4eff-8f97-cfc0112b6e68)

Настройка kerberos:
```
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

## Клиенты домена
### CLI
Установка active directory:
```
apt-get install task-auth-ad-sssd
```
Настройки адаптера

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/0f5cc71f-d1b3-40de-94bd-0eafc084b71b)

Аутентификация

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/23067dd0-8f19-4530-98cf-7df64a9feadf)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/e7686094-4d5f-4885-9917-754df2a1f130)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/cd3290bc-02d5-43e6-83cc-8173fd54c31d)

### BR-SRV
Установка active directory:
```
apt-get install task-auth-ad-sssd
```
```
nano /etc/resolv.conf
```
```
search hq.work
search branch.work
nameserver 192.168.0.40
```
Ввод в домен system-auth write ad <Домен> <Имя компьютера> <Рабочая группа> <Имя пользователя> <Пароль>:
```
system-auth write ad hq.work br-srv hq 'administrator' 'P@ssw0rd'
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/50b23e0e-c57a-46ff-9326-97ced427810b)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/b549eb73-463b-416d-9cf6-85586423c26c)

# №2.1 DNS HQ-SRV
Смотрим созданные доменом зоны:
```
samba-tool dns zonelist 127.0.0.1 -U administrator
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/6ba94fc9-e7e9-4600-a229-8c74fbb7cd7a)

Создадим зону branch.work и две обратные:
```
samba-tool dns zonecreate 127.0.0.1 branch.work -U administrator
```
