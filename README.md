https://sysahelper.gitbook.io/sysahelper/main/complex_works/main/demo2024  
https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/index.html  

# Все действия выполняются на ALTSERVER И ALTSTATION 10.1

***

# Модуль 1 задание 1 
1. Выполните базовую настройку всех устройств:  
&ensp; a. Присвоить имена в соответствии с топологией  
&ensp; b. Рассчитайте IP-адресацию IPv4 и IPv6. Необходимо заполнить таблицу №1, чтобы эксперты могли проверить ваше рабочее место.  
&ensp; c. Пул адресов для сети офиса BRANCH - не более 16  
&ensp; d. Пул адресов для сети офиса HQ - не более 64  

<details>
  <summary>ТЫКНИ</summary>

| Имя устройства | Интерфейс | Ip-адрес | Маска/Префикс | Шлюз | IPv6 | NIC |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|ISP|ens18|10.10.201.191|/24 255.255.255.0|10.10.201.254| - | INTERNET |
||ens19|11.11.11.1|/30 255.255.255.252||2001:11::1/64|ISP-HQ|
||ens20|22.22.22.1|/30 255.255.255.252||2001:22::1/64|ISP-BR|
||ens22|33.33.33.1|/30 255.255.255.252||2001:33::1/64|ISP-CLI|
|BR-R|ens18|192.168.100.1|/27 255.255.255.224||2000:200::f/124|BR|
||ens20|22.22.22.2|/30 255.255.255.252|22.22.22.1|2001:22::22/64|BR-ISP|
|HQ-R|ens19|11.11.11.2|/30 255.255.255.252|11.11.11.1|2001:11::11/64|HQ-ISP|
||ens18|192.168.0.1|/25 255.255.255.128||2000:100::3f/122|HQ|
|BR-SRV|ens18|192.168.100.2|/27 255.255.255.224|192.168.100.1|2000:200::1/124|BR|
|HQ-SRV|ens18|192.168.0.2|/25 255.255.255.128|192.168.0.1|2000:100::1/122|HQ|

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/3412d3b9-7b67-4fc1-8cc9-9b0ac66ff7dc)

Присвоение имён:
```
hostnamectl set-hostname <name>;exec bash
```

## HQ-SRV
Смотрим название адаптера:
```
ip -с a
```
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
CONFIG_IPV6=yes
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

***

</details>

# Модуль 1 задание 2

Настройте внутреннюю динамическую маршрутизацию по средствам FRR. Выберите и обоснуйте выбор протокола динамической маршрутизации из расчёта, что в дальнейшем сеть будет масштабироваться.  
&ensp; a. Составьте топологию сети L3.  

<details>
  <summary>ТЫКНИ</summary>

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
sed -i 's/ospfd=no/ospfd=yes/g' /etc/frr/daemons
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

</details>

<details>
  <summary>NAT_1</summary>

# NAT и открывание портов с помощью firewalld ISP,HQ-R,BR-R:

https://www.dmosk.ru/miniinstruktions.php?mini=firewalld-centos&ysclid=lq6h7dyu12576099184
https://www.dmosk.ru/miniinstruktions.php?mini=firewalld-centos&ysclid=lq6lwe2gxx781068118

опция CONFIG_FW (в файле /etc/net/ifaces/default/options) =yes

Отключить NetworkManager:
```
systemctl disable NetworkManager
```
Настройки интерфейсов должны быть такими:
```
...
NM_CONTROLLED=no
DISABLED=no
...
```
Установка firewalld:
```
apt-get update && apt-get -y install firewalld && systemctl enable --now firewalld
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
Включаем пересылку всех пакетов на ISP между BR-R и HQ-R
```
firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i ens35 -o ens34 -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i ens34 -o ens35 -j ACCEPT
```
Открываем порты OSPF:
```
firewall-cmd --permanent --zone=trusted --add-port=89/tcp
firewall-cmd --permanent --zone=trusted --add-port=89/udp
```
HQ-R и BR-R
Включаем пересылку между интерфейсом смотрящим в ISP и туннелем:
```
firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i ens160 -o iptunnel -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i iptunnel -o ens160 -j ACCEPT
```
Открываем порты OSPF:
```
firewall-cmd --permanent --zone=trusted --add-port=89/tcp
firewall-cmd --permanent --zone=trusted --add-port=89/udp
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/4823d6ae-ed09-4919-a691-db6a1ed11570)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/079d9e20-d946-4183-9157-30456762f88e)

***

</details>

<details>
  <summary>NAT_2</summary>

# NAT 2 способ ISP,HQ-R,BR-R:
Включаем пересылку пакетов:
```
nano /etc/net/sysctl.conf
net.ipv4.ip_forward = 1
```
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
systemctl enable --now iptables
```

</details>

<details>
  <summary>NAT_3</summary>

# NAT 3 способ ISP,HQ-R,BR-R:
iptables (предполагается, что внешний интерфейс носит имя enp1s0):
```
iptables -t nat -A POSTROUTING -s 192.0.2.0/24 -o enp1s0 -j SNAT --to-source 198.51.100.234
```
```
iptables -A INPUT -i wlp3s2 -j ACCEPT
```
```
iptables -A FORWARD -i wlp3s2 -o enp1s0 -j ACCEPT
```
```
iptables-save > /etc/sysconfig/iptables
```

</details>

<details>
  <summary>NAT_4</summary>

# NAT 4 способ HQ-R,BR-R

nftables
```
apt-get update && apt-get -y install nftables
```
```
nft flush ruleset   # для удаления всех правил и таблиц в системе, которые были настроены с использованием nftables
nft add table nat   # используется для создания новой таблицы с именем "nat" 
```
Prerouting
```
nft -- add chain nat prerouting { type nat hook prerouting priority -100 \; }
```
Postrouting
```
nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
```
Применяем правило к интерфейсу, который смотрит в сторону Интернета
```
nft add rule nat postrouting oifname "ens33" masquerade
```
Сохранение правил
```
echo "#!/usr/sbin/nft -f" > /etc/nftables/nftables.nft
echo "flush ruleset" >> /etc/nftables/nftables.nft
nft list ruleset >> /etc/nftables/nftables.nft
systemctl restart nftables
```

</details>

# Модуль 1 задание 3

Настройте автоматическое распределение IP-адресов на роутере HQ-R.  
&ensp; a. Учтите, что у сервера должен быть зарезервирован адрес.

<details>
  <summary>КЛИКНИ</summary>

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

***

</details>

# Модуль 1 задание 4
Настройте локальные учётные записи на всех устройствах в соответствии с таблицей.

|Учётная запись|Пароль|Примечание|
|:-:|:-:|:-:|
|Admin|P@ssw0rd|CLI, HQ-SRV, HQ-R|
|Branch admin|P@ssw0rd|BR-SRV, BR-R|
|Network admin|P@ssw0rd|HQ-R, BR-R, HQ-SRV|

<details>
  <summary>КЛИКНИ</summary>

Пользователь `admin` на `HQ-SRV`
```
adduser admin
```
```
usermod -aG wheel admin
```
```
passwd admin
P@ssw0rd
P@ssw0rd
```

</details>

# Модуль 1 задание 5

Измерьте пропускную способность сети между двумя узлами HQ-R-ISP по средствам утилиты iperf 3. Предоставьте описание пропускной способности канала со скриншотами.

<details>
  <summary>КЛИКНИ</summary>

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

</details>

# Модуль 1 задание 6

Составьте backup скрипты для сохранения конфигурации сетевых устройств, а именно HQ-R BR-R. Продемонстрируйте их работу.

<details>
  <summary>КЛИКНИ</summary>

Создаём папку для бэкапа:
```
mkdir /etc/networkbackup
```
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

</details>

<details>
  <summary>UrBackup</summary>

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

</details>

# Модуль 1 задание 7

Настройте подключение по SSH для удалённого конфигурирования устройства HQ-SRV по порту 2222. Учтите, что вам необходимо перенаправить трафик на этот порт по средствам контролирования трафика.

<details>
  <summary>КЛИКНИ</summary>

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

</details>

# Модуль 1 задание 8

Настройте контроль доступа до HQ-SRV по SSH со всех устройств, кроме CLI.

<details>
  <summary>КЛИКНИ</summary>

HQ-SRV:
```
nano /etc/openssh/sshd_config
```
Выбор пользователей
```
AllowUsers student@192.168.0.1 student@192.168.0.140 student@192.168.0.129 student@10.10.201.174
```

</details>

# Модуль 2 задание 2

Настройте синхронизацию времени между сетевыми устройствами по протоколу NTP.  
&ensp; a. В качестве сервера должен выступать роутер HQ-R со стратумом 5  
&ensp; b. Используйте Loopback интерфейс на HQ-R, как источник сервера времени  
&ensp; c. Все остальные устройства и сервера должны синхронизировать свое время с роутером HQ-R  
&ensp; d. Все устройства и сервера настроены на московский часовой пояс (UTC +3)  

Переставить часовой пояс на всех машинах:
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

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/86123bd5-6a33-4e11-b62f-c7f5ecbdc091)

***

# Модуль 2 задание 1

Настройте DNS-сервер на сервере HQ-SRV: 
a. На DNS сервере необходимо настроить 2 зоны  
Зона hq.work, также не забудьте настроить обратную зону.

|Имя|Тип записи|Адрес|
|:-:|:-:|:-:|
|hq-r.hq.work|A, PTR|IP-адрес|
|hq-srv.hq.work|A, PTR|IP-адрес|
|br-r.branch.work|A, PTR|IP-адрес|
|br-srv.branch.work|A|IP-адрес|

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

***

# Модуль 2 задание 3

https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/install-package.html
https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/ch02s03.html  
https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/testing-samba-dc.html  
https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/ch02s02s03.html  
https://docs.altlinux.org/ru-RU/domain/10.2/html/samba/index.html  

Настройте сервер домена на базе HQ-SRV через web интерфейс, выбор его типа и технологий обоснуйте.  
a. Введите машины BR-SRV и CLI в данный домен  
b. Организуйте отслеживание подключения к домену  

Перед установкой отключить конфликтующие службы krb5kdc, slapd, bind, smb, nmb:
```
systemctl stop krb5kdc
systemctl disable krb5kdc
systemctl status krb5kdc
```
Поменять порт к web-интерфейсу:
```
nano /etc/ahttpd/ahttpd.conf
```
```
server-port  8082
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
```
samba-tool dns zonecreate 127.0.0.1 0.168.192.in-addr.arpa -U administrator
```
```
samba-tool dns zonecreate 127.0.0.1 100.168.192.in-addr.arpa -U administrator
```
Создание записи типа А:
```
samba-tool dns add 127.0.0.1 hq.work hq-r A 192.168.0.1 -U administrator
```
```
samba-tool dns add 127.0.0.1 hq.work hq-srv A 192.168.0.40 -U administrator
```
```
samba-tool dns add 127.0.0.1 branch.work br-r A 192.168.100.1 -U administrator
```
```
samba-tool dns add 127.0.0.1 branch.work br-srv A 192.168.100.2 -U administrator
```
Создание записи типа PTR:
```
samba-tool dns add 127.0.0.1 0.168.192.in-addr.arpa 1 PTR hq-r.hq.work -U administrator
```
```
samba-tool dns add 127.0.0.1 0.168.192.in-addr.arpa 40 PTR hq-srv.hq.work -U administrator
```
```
samba-tool dns add 127.0.0.1 100.168.192.in-addr.arpa 1 PTR br-r.branch.work -U administrator
```
Проверка

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/0adee76f-0961-4318-87bd-6340e7d01afb)

***

# Модуль 2 задание 4

https://sysahelper.gitbook.io/sysahelper/main/linux_admin/main/fileserver_nfs  
https://www.altlinux.org/NFS_%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80_%D1%81_Kerberos_%D0%B0%D0%B2%D1%82%D0%BE%D1%80%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B5%D0%B9  

Реализуйте файловый SMB или NFS (выбор обоснуйте) сервер на базе сервера HQ-SRV.  
a. Должны быть опубликованы общие папки по названиям:  
i. Branch_Files - только для пользователя Branch admin;  
ii. Network - только для пользователя Network admin;  
iii. Admin_Files - только для пользователя Admin;  
b. Каждая папка должна монтироваться на всех серверах в папку /mnt/<name_folder> (например, /mnt/All_files) автоматически при входе доменного пользователя в систему и отключаться при его выходе из сессии. Монтироваться должны только доступные пользователю каталоги.  
Выберу NFS, потому что легче Samba:

## Создание RAID

```
apt-get install -y mdadm
```
Смотрим утилитой `lsblk`, созданные диски

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/79af8002-bb01-4e62-a6d2-6998c5d1828c)

Стираем данные суперблоков:
```
/sbin/mdadm --zero-superblock --force /dev/sd{b,c}
```
Если мы получили ответ:

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/724d960c-766e-4699-8ec8-b8843ef374dd)

то значит, что диски не использовались ранее для RAID.

Нужно удалить старые метаданные и подпись на дисках:
```
/sbin/wipefs --all --force /dev/sd{b,c}
```
Создание RAID:
```
/sbin/mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sd{b,c}
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/5631c323-5f03-4a0d-b6c8-7a5899bc77da)

где:
* /dev/md0 — устройство RAID, которое появится после сборки;
* -l 0 — уровень RAID;
* -n 2 — количество дисков, из которых собирается массив;
* /dev/sd{b,c} — сборка выполняется из дисков sdb и sdc.

Проверяем:
```
lsblk
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/60cdcff5-5c86-4901-a655-5ff8425b4901)

Создание файла mdadm.conf:
> В файле mdadm.conf находится информация о RAID-массивах и компонентах, которые в них входят.

```
mkdir /etc/mdadm
```
```
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
```
```
/sbin/mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```

Содержимое `mdadm.conf`

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/b630d9e9-f79b-4395-a58b-704ee5aee9f5)

Создание файловой системы для массива:
```
/sbin/mkfs.ext4 /dev/md0
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/606b0728-28e9-43c5-94f7-696034642e98)

Автозагрузка раздела с помощью `fstab`. Смотрим идентификатор раздела:
```
/sbin/blkid | grep /dev/md0
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/78b61509-38bd-4e74-8bf7-263b342158d7)

Открываем `fstab` и добавляем строку:
```
nano /etc/fstab
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/d24809e4-d58c-419e-8004-cb8729211e7c)

> в данном случае массив примонтирован в каталог `/mnt`

Выполняем монтирование и проверяем

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/397a9392-8234-4e21-ba14-4b168acea47b)

## Настройка NFS-сервера

Установка пакетов для NFS сервера:
```
apt-get install -y nfs-server
apt-get install -y rpcbind
apt-get install -y nfs-clients
apt-get install -y nfs-utils
```
Автозагрузка:
```
systemctl enable --now nfs
```
Создание директории общего доступа:
```
mkdir /mnt/nfs_share
```
```
chmod 777 /mnt/nfs_share
```
Редактируем `exports`:
```
nano /etc/exports
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/50694635-7a92-4adc-9e68-ceebd01fc253)


где:
* /mnt/nfs_share - общий ресурс
* 192.168.0.0/25 - клиентская сеть, которой разрешено монтирования общего ресурса
* rw — разрешены чтение и запись
* no_root_squash — отключение ограничения прав root

Экспорт файловой системы:
```
/usr/sbin/exportfs -arv
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/b7957a6b-c029-45ac-95b6-78e6ce177e26)


> exportfs с флагом -a, означающим экспортировать или отменить экспорт всех каталогов, -r означает повторный экспорт всех каталогов, синхронизируя /var/lib/nfs/etab с /etc/exports и файлами в /etc/exports.d, а флаг -v включает подробный вывод:

Запускаем и добавляем в автозагрузку NFS-сервер:
```
systemctl enable --now nfs-server
```

## Настройка NFS-клиента

Установка пакетов для NFS-клиента:
```
apt-get update && apt-get install -y nfs-{utils,clients}
```
Создадим директорию для монтирования общего ресурса:
```
mkdir /opt/share
```
```
chmod 777 /opt/share
```
Настраиваем автомонтирование общего ресурса через `fstab`:
```
nano /etc/fstab
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/4a58bd44-b514-4e4b-b103-66ea864ec87c)

где: 192.168.0.40 - адрес файлового сервера

Монтируем:
```
mount -a
```
Проверяем:
```
df -h
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/7bb0c86b-ac11-411a-b2fa-1b95a52b55a4)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/97fbd4e1-5475-4675-af7b-00fc80d2cbf8)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/20f0eb72-6355-4b15-9eff-65b672e6c50c)

## Сервер HQ-SRV

Создание общих папок на сервере:
```
mkdir /mnt/network -p
```
```
mkdir /mnt/admin_files -p
```
```
mkdir /mnt/branch_files -p
```
Заносим в `exports`:
```
nano /etc/exports
```
```
/mnt/network 192.168.0.0/25 192.168.100.0/27(rw,sync,no_root_squash)
/mnt/branch_files 192.168.100.0/27(rw,sync,no_root_squash)
/mnt/admin_files 192.168.0.0/25 4.4.4.0/30(rw,sync,no_root_squash)
```
Экспортируем:
```
/usr/sbin/exportfs -arv
```

## Клиент HQ-SRV

Создаём папку:
```
mkdir /opt/admin
```
Задаём права:
```
chmod 777 /opt/admin/
```
Автозагрузка в `fstab`:
```
192.168.0.40:/mnt/admin_files /opt/admin nfs defaults 0 0
```
Монтаж:
```
mount -a
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/c9a66793-840b-42fc-b3cc-175aceadedca)

***

# Модуль 2 задание 5

Сконфигурируйте веб-сервер LMS Apache на сервере BR-SRV:  
a. На главной странице должен отражаться номер места  
b. Используйте базу данных mySQL  
c. Создайте пользователей в соответствии с таблицей, пароли у всех пользователей «P@ssw0rd»  

|Пользователь|Группа|
|:-:|:-:|
|Admin|Admin|
|Manager1|Manager|
|Manager2|Manager|
|Manager3|Manager|
|User1|WS|
|User2|WS|
|User3|WS|
|User4|WS|
|User5|TEAM|
|User6|TEAM|
|User7|TEAM|

Установка Moodle:
```
apt-get update && apt-get install -y moodle
```
```
apt-get install -y moodle-apache2
```
```
apt-get install -y moodle-local-mysql
```
Автозагрузка базы данных:
```
systemctl enable --now mariadb
```
Автозагрузка Apache2 (В AltLinux назывется httpd2):
```
systemctl enable --now httpd2
```
Входим в СУБД:
```
mysql -u root
```
Создаём базу данных:
```
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
```
Ставим максимальные права пользователя `moodleuser` в БД `moodle`, пароль `moodlepasswd`:
```
GRANT ALL ON moodle.* TO moodleuser@localhost IDENTIFIED BY 'moodlepasswd';
```
Обновляем политики:
```
FLUSH PRIVILEGES;
```
Выходим:
```
EXIT
```
Перезапускаем сервисы:
```
systemctl restart httpd2
systemctl restart mariadb
```
Заходим по адресу `http://localhost/moodle`  
Выбираем русский язык

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/f020417c-1936-48e2-9cc9-bae8dceec0cb)

Потверждаем пути

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/961e879c-2d88-46ba-b690-56a7320973dc)

Драйвер базы данных MariaDB

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/dcc4c7f4-ab45-461e-b99a-572a68d36c7b)

Сервер баз данных `localhost`  
Название БД `moodle`  
Пользователь БД `moodleuser`  
Пароль `moodlepasswd`  
Префикс имен таблиц `mdl_`  
Порт БД `3106`  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/26d2f4fe-053b-4f30-8e4e-eed6d91b9b57)

Продолжить

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/2d341da2-0aea-42d4-842a-d991292cc518)

Меняем количество максимальных переменных:
```
nano /etc/php/8.0/apache2-mod_php/php.ini
```
Убираем `;` перед `max_input_vars = 1000`
```
max_input_vars = 6000
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/41f0e391-c2a3-469c-b182-c628612981d6)

Должно появиться такое

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/a9869cac-81e9-478a-a4fc-ef8f14033c53)

В следующей массивной странице должны вывестись все записи `Успешно`, продолжаем

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/4d413865-88c9-4b45-b188-79e8a7b6080c)

Логин `admin`  
Пароль `P@ssw0rd`  
e-mail `любой`  
Часовой пояс `Азия/Екатеринбург`  

Дальше задаём имя ресурса и т. д.  
Попадаем на главную страницу  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/cb8407d4-a83b-4654-88b8-c9078b68f9a5)

Создание пользователя `manager`(Администрирование -> Пользователи -> Добавить пользователя)  
Дальше по таблице создаём других  
Создание групп (Администрирование -> Пользователи -> Глобальные группы -> Добавить глобальную группу)  
Остальное всё легко  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/82196cf0-f5bd-4486-87ba-b6d4aa4eb6d6)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/5e33f9e8-6f45-4097-99c7-7283fa2b6f5a)

***

# Модуль 2 задание 6

Запустите сервис MediaWiki используя docker на сервере HQ-SRV.  
a. Установите Docker и Docker Compose.  
b. Создайте в домашней директории пользователя файл wiki.yml для приложения MediaWiki:  
        i. Средствами docker compose должен создаваться стек контейнеров с приложением MediaWiki и базой данных  
        ii. Используйте два сервиса;  
        iii. Основной контейнер MediaWiki должен называться wiki и использовать образ mediawiki;  
        iv. Файл LocalSettings.php с корректными настройками должен находиться в домашней папке пользователя и автоматически монтироваться в образ;  
        v. Контейнер с базой данных должен называться db и использовать образ mysql;  
        vi. Он должен создавать базу с названием mediawiki, доступную по стандартному порту, для пользователя wiki с паролем DEP@ssw0rd;  
        vii. База должна храниться в отдельном volume с названием dbvolume.  
MediaWiki должна быть доступна извне через порт 8080.  

Установка Docker и Docker-compose:
```
apt-get update && apt-get install -y docker-engine
apt-get install -y docker-compose
```
Автозагрузка `Docker`:
```
systemctl enable --now docker
```
Привязка пользователя к `Docker`:
```
usermod student -aG docker
```
Переходим к домашней директории пользователя:
```
cd /home/student
```
Создаём файл wiki.yml:
```
touch wiki.yml
```
Привести к следующему виду:
```yml
version: '3'
services:
  mediawiki:
    image: mediawiki
    restart: always
    ports:
      - 8080:80
    links:
      - database
    container_name: wiki
    volumes:
      - images:/var/www/html/images
# Сначала устанавливаем вручную до конца, потом убираем комментарий
#      - ./LocalSettings.php:/var/www/html/LocalSettings.php
  database:
    image: mariadb
    container_name: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: mediawiki
      MYSQL_USER: wiki
      MYSQL_PASSWORD: DEP@ssw0rd
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
      TZ: Asia/Yekaterinburg
    volumes:
      - db:/var/lib/mysql
volumes:
  images:
  db:
```
Запускаем контейнеры:
```
docker-compose -f wiki.yml up -d
```
Если всё правильно переходим по `localhost:8080` и должно появиться - 'Please set up the wiki first'  
Настройка базы данных  
Для того, чтобы узнать хост базы данных:
```
docker exec -it mariadb bash
```
```
hostname -i
```
Вывод
```
172.18.0.2
```
Заполняем таблицу основываясь на wiki.yml файле и полученном хосте  
Потом создаём Админа  
После этого должен скачаться LocalSettings.php  
Копируем:
```
cp /home/user/Downloads/LocalSettings.php /home/student/
```
Снимаем комментарий ./LocalSettings.php:/var/www/html/LocalSettings.php в wiki.yml  
Перезапускаем контейнеры:
```
docker-compose -f wiki.yml up -d
```
Mediawiki успешно установлена

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/972a864d-cb67-4395-9ca8-f83d4bcead94)

***

# Модуль 3 задание 1

1. Реализуйте мониторинг по средствам rsyslog на всех Linux хостах.  
a. Составьте отчёт о том, как работает мониторинг  

#### Настройка сервера, который принимает логи

Установка `rsyslog`:
```
apt-get install rsyslog-classic
```
Автозагрузка:
```
systemctl enable --now rsyslog
```
Конфиг `/etc/rsyslog.d/00_common.conf`:
```
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")
```
`/etc/rsyslog.d/myrules.conf`:
```
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
> Название шаблона `RemoteLogs`, который принимает логи всех категорий; логи будут сохраняться в каталоге `/var/log/rsyslog/<имя компьютера, откуда пришел лог>/<приложение, чей лог пришел>.log`; конструкция `& ~` прекращает дальнейшую обработку логов

Перезапускаем службу логов:
```
systemctl restart rsyslog
```

#### Настройка клиентов

Установка пакета:
```
apt-get install -y rsyslog-classic
```
Автозагрузка:
```
systemctl enable --now rsyslog
```
Добавляем в `/etc/rsyslog.d/all.conf`:
```
*.* @@192.168.0.2:514
```
> Отправлять все логи всех важностей по tcp на сервак

В файле `/etc/rsyslog.d/20_extrasockets.conf` закомментировать всё
***

# Модуль 3 задание 2

Выполните настройку центра сертификации на базе HQ-SRV:  
a. Выдайте сертификаты для SSH;  
b. Выдайте сертификаты для веб серверов.  

Создадим директорию которую будем использовать как корневую:
```
mkdir /ca
```
Найдём путь, где расположен конфиг:
```
openssl ca
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/73f5da07-8ebb-4bba-8dbe-9fbfa902a8fb)

Сделаем бэкап конфига:
```
cp /var/lib/ssl/openssl.{cnf,cnf.backup}
```
Редактируем конфиг `/var/lib/ssl/openssl.conf`

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/8f413992-3aad-443c-811f-9179e7555bc8)

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/efbbe338-48e7-4bfc-a123-8818a569c3c3)

```
cd /ca
mkdir certs newcerts crl private
touch index.txt
echo -n '00' > serial
```
```
policy = policy_anything
commonName = supplied
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/27805547-5609-43be-802d-002ddf17853c)

```
countryName_default = RU
0.organizationName_default = hq.work
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/16b807c3-f27a-4bf4-9293-b75f76a5e905)

```
[ v3_ca]
...
basicConstraints = CA:true
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/850c3aca-bf8d-4329-afbe-3e5b03591ca7)

Генерируем ключ:
```
openssl req -nodes -new -out cacert.csr -keyout private/cakey.pem -extensions v3_ca
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/b4f31d16-2fef-4b08-95b7-0fab4fea3f96)

> `.` значит пустой

***

# Модуль 3 задание 3

Настройте SSH на всех Linux хостах:  
a. Banner ( Authorized access only! );  
b. Установите запрет на доступ root;  
c. Отключите аутентификацию по паролю;  
d. Переведите на нестандартный порт;  
e. Ограничьте ввод попыток до 4;  
f. Отключите пустые пароли;  
g. Установите предел времени аутентификации до 5 минут;  
h. Установите авторизацию по сертификату выданным HQ-SRV.  

HQ-SRV  
Генерация пары ключей:
```
ssh-keygen -t rsa -b 2048 -f alt_key
```
```
mv alt_key* .ssh/
```
Создаём `config` для подключений:
```
nano .ssh/config
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/f8bd713e-e800-49de-95dd-79e3ef2c7cef)

```
chmod 600 .ssh/config
```
Копируем публичные ключи на сервера HQ-R,BR-R,BR-SRV:
```
ssh-copy-id -i .ssh/alt_key.pub student@192.168.0.1
```
```
ssh-copy-id -i .ssh/alt_key.pub student@192.168.100.1
```
```
ssh-copy-id -i .ssh/alt_key.pub student@192.168.100.2
```
Все действия по порядку, начиная с создания пары ключей выполнить на HQ-R,BR-R,BR-SRV 

На HQ-R,HQ-SRV,BR-R,BR-SRV в файле `/etc/openssh/sshd_config`:

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/3e8f2e11-3533-464b-8e0f-f195ec2ff740)

Перезапускаем `sshd`:
```
systemctl restart sshd
```
Подключаемся:

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/97e4ec18-dcb3-4027-b805-15afa27b2ecd)

Чтобы при подключении не прописывать порт, пропишите в `.ssh/config`:
```
Port 2222
```
***

# Модуль 3 задание 4 (переделать)

Реализуйте антивирусную защиту по средствам ClamAV на устройствах HQ-SRV и BR-SRV:  
a. Настройте сканирование системы раз в сутки с сохранением отчёта  
i. Учтите, что сканирование должно проводится при условии, что от пользователей нет нагрузки  

Установка clamav:
```
apt-get update && apt-get install clamav
```
Автозагрузка:
```
systemctl enable --now clamav-daemon.service
```
```
nano /usr/local/sbin/clam_all.sh
```
```
#!/bin/bash
SCAN_DIR="/"
LOG_FILE="/var/log/clamav/manual_clamscan.log"
/usr/bin/clamscan -i -r $SCAN_DIR >> $LOG_FILE
```
Разрешение на запуск скрипта:
```
chmod +x /usr/local/sbin/clam_www_scan.sh
```
Создаем файл лога:
```
touch /var/log/clamav/manual_clamscan.log
```
Запуск по расписанию:
```
EDITOR=nano crontab -e
```
Каждый день в час ночи запускать сканирование:
```
1 1 * * * /usr/local/sbin/clam_all.sh > /dev/null
```
***

# Модуль 3 задание 5

Настройте систему управления трафиком на роутере BR-R для контроля входящего трафика в соответствии со следующими правилами:  
a. Разрешите подключения к портам DNS (порт 53), HTTP (порт 80) и HTTPS (порт 443) для всех клиентов. Эти порты необходимы для работы настраиваемых служб.  
b. Разрешите работу выбранного протокола организации защищенной связи. Разрешение портов должно быть выполнено по принципу "необходимо и достаточно".  
c. Разрешите работу протоколов ICMP (протокол управления сообщениями Internet).  
d. Разрешите работу протокола SSH (Secure Shell) (SSH используется для безопасного удаленного доступа и управления устройствами).  
e. Запретите все прочие подключения.  
f. Все другие подключения должны быть запрещены для обеспечения безопасности сети.  

Разрешаем порты:
```
firewall-cmd --permanent --zone=public --add-port=53/{tcp,udp}
firewall-cmd --permanent --zone=trusted --add-port=53/{tcp,udp}
```
```
firewall-cmd --permanent --zone=public --add-port=80/{tcp,udp}
firewall-cmd --permanent --zone=trusted --add-port=80/{tcp,udp}
```
```
firewall-cmd --permanent --zone=public --add-port=443/{tcp,udp}
firewall-cmd --permanent --zone=trusted --add-port=443/{tcp,udp}
```
```
firewall-cmd --permanent --zone=public --add-port=22/tcp
firewall-cmd --permanent --zone=trusted --add-port=22/tcp
```
Разрешаем протоколы
```
firewall-cmd --permanent --zone=public --add-protocol={icmp,gre,ospf}
firewall-cmd --permanent --zone=trusted --add-protocol={icmp,gre,ospf}
```
Разрешаем сервисы:
```
firewall-cmd --permanent --zone=public --add-service={ssh,ipsec,ntp}
firewall-cmd --permanent --zone=trusted --add-service={ssh,ipsec,ntp}
```
Блокируем остальные подключения и всё ненужное:
```
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i ens33 -o ens34 -j DROP
```
***

# Модуль 3 задание 6

Настройте виртуальный принтер с помощью CUPS для возможности печати документов из Linux-системы на сервере BR-SRV.  

Установка `cups`:
```
apt-get update && apt-get install -y cups
```
Установка наиболее популярных драйверов на принтеры:
```
apt-get install -y gutenprint-cups
```
Добавить в автозагрузку:
```
systemctl enable --now cups-browsed.service
systemctl enable --now cups-lpd.socket
```
Установка виртуального принтера:
```
apt-get install -y cups-pdf
```
Заходим по пути `Система -> Администрирование -> Параметры печати`

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/4bd0423a-d72e-4176-94d0-cf3c1cbb33fd)

Проверяем работу виртуального принтера:
* Создаем текстовый файл

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/dc200d97-f066-4839-b6ca-1578fd1cd815)

* Файл -> Напечатать... -> Cups-PDF -> Печать

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/80d81182-650d-4b52-87e4-0a9d0df686ef)


***

# Модуль 3 задание 7

Между офисами HQ и BRANCH установите защищенный туннель, позволяющий осуществлять связь между регионами с применением внутренних адресов.  

HQ-R
```
mkdir /etc/net/ifaces/iptunnel
```
```
nano /etc/net/ifaces/iptunnel/ipv4address
```
```
10.20.30.1/30
```
```
nano /etc/net/ifaces/iptunnel/options
```
```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=11.11.11.2
TUNREMOTE=22.22.22.2
TUNOPTIONS='ttl 64'
HOST=ens18
```
```
nano /etc/net/ifaces/iptunnel/ipv4route
```
```
10.20.30.0/30 via 10.20.30.2
```
```
systemctl restart network
```

BR-R
```
mkdir /etc/net/ifaces/iptunnel
```
```
nano /etc/net/ifaces/iptunnel/ipv4address
```
```
10.20.30.2/30
```
```
nano /etc/net/ifaces/iptunnel/options
```
```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=22.22.22.2
TUNREMOTE=11.11.11.2
TUNOPTIONS='ttl 64'
HOST=ens18
```
```
nano /etc/net/ifaces/iptunnel/ipv4route
```
```
10.20.30.0/30 via 10.20.30.1
```
```
systemctl restart network
```
Установим strongswan:
```
apt-get update && apt-get install -y strongswan
```
Включим и добавим в автозагрузку службы для работы IPSec:
```
systemctl enable --now strongswan-starter.service
systemctl enable --now ipsec
```
HQ-R  

Конфигурационный файл "/etc/strongswan/ipsec.conf":
```
nano /etc/strongswan/ipsec.conf
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/05f62fd9-820f-4095-ac87-e404f8752da0)

Задаём общий секрет:
```
nano /etc/strongswan/ipsec.secrets
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/e01993b0-8ace-40f5-ae80-17151b6d8982)

Перезапускаем:
```
systemctl restart ipsec
```
BR-R  
Те же самые настройки, меняем местами ip-адреса  
Проверка  

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/90b247f8-e2ed-4fc2-9270-d1cf69d66dfc)

***

# Модуль 3 задание 8

По средствам уже настроенного мониторинга установите следующие параметры:  
a. Warning  
i. Нагрузка процессора больше или равна 70%  
ii. Заполненность оперативной памяти больше или равна 80%  
iii. Заполненность диска больше или равна 85%  
b. Напишите план действия при получении Warning сообщений  

***

# Модуль 3 задание 9

Настройте программный RAID 5 из дисков по 1 Гб, которые подключены к машине BR-SRV.  

```
apt-get install -y mdadm
```
Смотрим утилитой `lsblk`, созданные диски

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/1b5040fb-2b3f-4341-a3f6-3cfde10138f5)


Стираем данные суперблоков:
```
/sbin/mdadm --zero-superblock --force /dev/sd{b,c}
```
Если мы получили ответ:

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/47d72cbd-02bb-43e6-860f-4a2d204b7e0a)


то значит, что диски не использовались ранее для RAID.

Нужно удалить старые метаданные и подпись на дисках:
```
/sbin/wipefs --all --force /dev/sd{b,c}
```
Создание RAID:
```
/sbin/mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sd{b,c}
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/748390a5-8bae-4510-8160-efe2ffe2da96)

где:
* /dev/md0 — устройство RAID, которое появится после сборки;
* -l 5 — уровень RAID;
* -n 3 — количество дисков, из которых собирается массив;
* /dev/sd{b,c,d} — сборка выполняется из дисков sdb и sdc.

Проверяем:
```
lsblk
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/330b64fd-e02b-4dfb-b875-546bd8346f8a)


Создание файла mdadm.conf:
> В файле mdadm.conf находится информация о RAID-массивах и компонентах, которые в них входят.

```
mkdir /etc/mdadm
```
```
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
```
```
/sbin/mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```

Содержимое `mdadm.conf`

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/b5968fd4-a3d4-4421-9d64-a6e091137bcf)


Создание файловой системы для массива:
```
/sbin/mkfs.ext4 /dev/md0
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/a8a5df4f-0c51-42c1-8685-f07e8ead8d6d)


Автозагрузка раздела с помощью `fstab`. Смотрим идентификатор раздела:
```
/sbin/blkid | grep /dev/md0
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/d2af9018-f835-413c-a8ed-63b46bc158f9)

Открываем `fstab` и добавляем строку:
```
nano /etc/fstab
```

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/3008389a-adb6-4d97-926d-3ffcef2789e2)

> в данном случае массив примонтирован в каталог `/mnt`

Выполняем монтирование и проверяем

![image](https://github.com/abdurrah1m/DEMO2024/assets/148451230/24a81b1a-000e-4e13-8959-c09d0f4835d6)

***

# Модуль 3 задание 10

Настройте Bacula на сервере HQ-SRV для резервного копирования etc на сервере BR-SRV.  

***
