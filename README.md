# otus-task22

# VPN

1. Между двумя виртуалками поднять vpn в режимах:
	* tun
	* tap
Описать в чём разница, замерить скорость между виртуальными \
машинами в туннелях, сделать вывод об отличающихся показателях \
скорости.
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, \
подключиться с локальной машины на виртуалку.

## 1. Между двумя виртуалками поднять vpn

### Решение

Для выполнения задания потребуется две виртуальные машины: server и client. \
Развернём виртуалки с помощью Vagrant. Для этого создадим Vagrantfile следующего содержания:
```

# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/focal64"
  
  config.vm.define "server" do |server|
    server.vm.hostname = "server.loc"
    server.vm.network "private_network", ip: "192.168.56.10"
  end
  
  config.vm.define "client" do |client|
    client.vm.hostname = "client.loc"
    client.vm.network "private_network", ip: "192.168.56.20"
  end
  
end
```

После поднятия машин из Vagrantfile необходимо зайти на каждую и установить пакет openvpn.
```

vagrant@server:~$ sudo apt install openvpn
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  libpkcs11-helper1
Suggested packages:
  resolvconf openvpn-systemd-resolved easy-rsa
The following NEW packages will be installed:
  libpkcs11-helper1 openvpn
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 527 kB of archives.
After this operation, 1352 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu focal/main amd64 libpkcs11-helper1 amd64 1.26-1 [44.3 kB]
Get:2 http://archive.ubuntu.com/ubuntu focal-updates/main amd64 openvpn amd64 2.4.12-0ubuntu0.20.04.1 [483 kB]
Fetched 527 kB in 3s (165 kB/s)  
Preconfiguring packages ...
Selecting previously unselected package libpkcs11-helper1:amd64.
(Reading database ... 94801 files and directories currently installed.)
Preparing to unpack .../libpkcs11-helper1_1.26-1_amd64.deb ...
Unpacking libpkcs11-helper1:amd64 (1.26-1) ...
Selecting previously unselected package openvpn.
Preparing to unpack .../openvpn_2.4.12-0ubuntu0.20.04.1_amd64.deb ...
Unpacking openvpn (2.4.12-0ubuntu0.20.04.1) ...
Setting up libpkcs11-helper1:amd64 (1.26-1) ...
Setting up openvpn (2.4.12-0ubuntu0.20.04.1) ...
 * Restarting virtual private network daemon.                                                                                                                                                               [ OK ] 
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn.service → /lib/systemd/system/openvpn.service.
Processing triggers for systemd (245.4-4ubuntu3.22) ...
Processing triggers for man-db (2.9.1-1) ...
Processing triggers for libc-bin (2.31-0ubuntu9.14) ...


vagrant@client:~$ sudo apt install openvpn
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  libpkcs11-helper1
Suggested packages:
  resolvconf openvpn-systemd-resolved easy-rsa
The following NEW packages will be installed:
  libpkcs11-helper1 openvpn
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 527 kB of archives.
After this operation, 1352 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu focal/main amd64 libpkcs11-helper1 amd64 1.26-1 [44.3 kB]
Get:2 http://archive.ubuntu.com/ubuntu focal-updates/main amd64 openvpn amd64 2.4.12-0ubuntu0.20.04.1 [483 kB]
Fetched 527 kB in 2s (294 kB/s)  
Preconfiguring packages ...
Selecting previously unselected package libpkcs11-helper1:amd64.
(Reading database ... 94801 files and directories currently installed.)
Preparing to unpack .../libpkcs11-helper1_1.26-1_amd64.deb ...
Unpacking libpkcs11-helper1:amd64 (1.26-1) ...
Selecting previously unselected package openvpn.
Preparing to unpack .../openvpn_2.4.12-0ubuntu0.20.04.1_amd64.deb ...
Unpacking openvpn (2.4.12-0ubuntu0.20.04.1) ...
Setting up libpkcs11-helper1:amd64 (1.26-1) ...
Setting up openvpn (2.4.12-0ubuntu0.20.04.1) ...
 * Restarting virtual private network daemon.                                                                                                                                                               [ OK ] 
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn.service → /lib/systemd/system/openvpn.service.
Processing triggers for systemd (245.4-4ubuntu3.22) ...
Processing triggers for man-db (2.9.1-1) ...
Processing triggers for libc-bin (2.31-0ubuntu9.14) ...
```

Далее переходим к настройке сервера. \
Создаём файл-ключ:
```

vagrant@server:~$ sudo openvpn --genkey --secret /etc/openvpn/static.key
vagrant@server:~$ sudo cat /etc/openvpn/static.key 
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
545ffe58d777d99a99f4e85f2afc0ab1
7a2abda7e5ab0a0a957eabe92423123c
a9a422c1dc797c9fb108511e2deb931f
d2bb2627551cd9a01acb54a499bad605
b35df6f86f143161e0e2f9e2acd2443c
72cf5e7e3078d399097145e11ae0990e
a5a515a6cea75abbdf26cf13012fc9ac
d3a066b44119775b0920ac40f944a05b
2535f18295c3b2b25864bcdd06f8e7d2
be7427e9702070ffac116a8e8a61d164
e0d1895846be6ffa0ceb370b75627a54
a33483add7e0d83438242bf13726f339
db9ebf518601d98e96ecf9a2af65019c
a98cdd7dd5776c7e928d6803d5848425
faf8deb00bd0dea11cd3e0c3155372bb
7f5e5726ee5339f85ef508347f5afc4c
-----END OpenVPN Static key V1-----
```

Далее создаём конфигурационный файл vpn-сервера.
```

vagrant@server:~$ sudo touch /etc/openvpn/server.conf
vagrant@server:~$ sudo nano /etc/openvpn/server.conf
vagrant@server:~$ cat /etc/openvpn/server.conf
dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

Создадим файл модуль для управления нашим openvpn через systemd.
```

vagrant@server:~$ sudo touch /etc/systemd/system/openvpn@.service
vagrant@server:~$ sudo nano /etc/systemd/system/openvpn@.service
vagrant@server:~$ cat /etc/systemd/system/openvpn@.service
[Unit]
Description=OpenVPN Tunneling Application On %I
After=network.target

[Service]
Type=notify
PrivateTmp=true
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf

[Install]
WantedBy=multi-user.target
```

После добавления нового модуля выполним выполним systemctl daemon-reload и запустим его.
```

vagrant@server:~$ sudo systemctl daemon-reload 
vagrant@server:~$ sudo systemctl start openvpn@server
vagrant@server:~$ sudo systemctl enable openvpn@server
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn@server.service → /etc/systemd/system/openvpn@.service.
vagrant@server:~$ systemctl status openvpn@server.service 
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-01-22 09:18:47 UTC; 4min 10s ago
   Main PID: 19109 (openvpn)
     Status: "Pre-connection initialization successful"
      Tasks: 1 (limit: 1117)
     Memory: 940.0K
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─19109 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf
```

Проверим появился ли новый сетевой интерфейс.
```

vagrant@server:~$ ip -c a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:7f:55:95:07:8e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 81240sec preferred_lft 81240sec
    inet6 fe80::7f:55ff:fe95:78e/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:df:39:62 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.10/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fedf:3962/64 scope link 
       valid_lft forever preferred_lft forever
5: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 100
    link/ether 92:ac:41:89:0e:a1 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global tap0
       valid_lft forever preferred_lft forever
    inet6 fe80::90ac:41ff:fe89:ea1/64 scope link 
       valid_lft forever preferred_lft forever
```

Видим, что появился наш новый интерфейс tap0. Настройка сервера закончена. \
Далее настраиваем openvpn на машине client. \
Создаём конфигурационный файл клиента.
```

vagrant@client:~$ sudo touch /etc/openvpn/client.conf
vagrant@client:~$ sudo nano /etc/openvpn/client.conf 
vagrant@client:~$ cat /etc/openvpn/client.conf
dev tap
remote 192.168.56.10
ifconfig 10.10.10.2 255.255.255.0
topology subnet
route 192.168.56.0 255.255.255.0
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
На сервер клиента в директорию /etc/openvpn необходимо скопировать \
файл-ключ static.key, который был создан на сервере. \



