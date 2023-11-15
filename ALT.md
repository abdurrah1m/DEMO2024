# Настройка статическго ip
## HQ-SRV
Смотрим название адаптера
```
ip a
```
`ens18`  
ip  
```
nano /etc/net/ifaces/ens18/ipv4address
```
```
192.168.0.40/24
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
