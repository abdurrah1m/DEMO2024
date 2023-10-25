nat debian https://quaded.com/nat-%D0%B2-debian/  
dhcp debian https://setiwik.ru/kak-nastroit-dhcp-server-v-debian-11/?ysclid=lo4mpr8jcj791182994  
# NANO
<kbd>ALT</kbd> + <kbd>/</kbd> Перейти в конец файла  
<kbd>CTRL</kbd> + <kbd>SHIFT</kbd> Скопировать  
<kbd>CTRL</kbd> + <kbd>U</kbd> Вставить  
<kbd>CTRL</kbd> + <kbd>K</kbd> Вырезать  

# Сеть на подсети

|Имя устройства |Интерфейс |Ip-адрес |Маска/Префикс |Шлюз |
|---------------|----------|---------|--------------|-----|
|ISP|ens18|10.10.201.100|/24 255.255.255.0|10.10.201.254|
||ens19|192.168.0.165|/30 255.255.255.252||
||ens20|192.168.0.161|/30 255.255.255.252||
|BR-R|ens18|192.168.0.129|/27 255.255.255.224||
||ens19|192.168.0.162|/30 255.255.255.252|192.168.0.161|
|HQ-R|ens18|192.168.0.1|/25 255.255.255.128||
||ens19|192.168.0.166|/30 255.255.255.252|192.168.0.165|
|BR-SRV|ens18|192.168.0.130|/27 255.255.255.224|192.168.0.129|
|HQ-SRV|ens18|192.168.0.2|/25 255.255.255.128|192.168.0.1|
# FRR HSRP

```
apt update
apt install frr
```



# DHCP SERVER

Установка DHCP `apt install isc-dhcp-server`  
Конфиг `nano /etc/default/isc-dhcp-server`  
Указываю интерфейс, который в сторону Интернета `INTERFACESV4="ens19"`  
```
nano /etc/dhcp/dhcpd.conf
subnet 192.168.0.0 netmask 255.255.255.0 {
range 192.168.0.2 192.168.0.125;
option domain-name-servers 8.8.8.8, 8.8.4.4;
option routers 192.168.0.1;
}
```
Применяю изменения `systemctl restart isc-dhcp-server.service`
