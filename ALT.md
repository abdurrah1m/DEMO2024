# Настройка статическго ip
## HQ-SRV
Смотрим название адаптера
```
ip a
```
`ens18`  
ip  
```
echo 192.168.0.40/24 > /etc/net/ifaces/ens18/ipv4address
```
gateway  
```
nano /etc/net/ifaces/ens18/ipv4route
```
```
192.168.0.1
```
Параметр BOOTPROTO отвечает за способ получения сетевой картой адреса, `static`  
```
nano /etc/net/ifaces/ens18/options
```
```
BOOTPROTO=static
```
Перезагрузка:
```
reboot
```
