nat debian https://quaded.com/nat-%D0%B2-debian/  
dhcp debian https://setiwik.ru/kak-nastroit-dhcp-server-v-debian-11/?ysclid=lo4mpr8jcj791182994  
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
# №1.2 FRR HSRP

```
apt update
apt install frr
```



# №1.3 DHCP SERVER

Установка DHCP
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
