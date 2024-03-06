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

После поднятия машин из Vagrantfile необходимо выполнить их первоначальную подгтовку и настройку. \
"Наливку машин будем выполнять с помощью Ansible".\
Устанавливаем openvpn и iperf3 на обе машины с помощью плейбука vpn.yml.
```

chumaksa@debpc:~/otus/otus-task22$ ansible-playbook vpn.yml 

PLAY [Configure all] **********************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [server]
ok: [client]

TASK [Install the openvpn] ****************************************************************************************************************************************************************************************
changed: [client]
changed: [server]

TASK [Install the iperf3] *****************************************************************************************************************************************************************************************
changed: [server]
changed: [client]

PLAY RECAP ********************************************************************************************************************************************************************************************************
client                     : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
server                     : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Далее настраиваем server при помощи плейбука server.yml.
```

chumaksa@debpc:~/otus/otus-task22$ ansible-playbook server.yml 

PLAY [Configure servers] ******************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [server]

TASK [Copy openvpn config file to server] *************************************************************************************************************************************************************************
changed: [server]

TASK [Create key] *************************************************************************************************************************************************************************************************
changed: [server]

TASK [Specifying a destination path] ******************************************************************************************************************************************************************************
changed: [server]

TASK [Copy openvpn@.service systemd module] ***********************************************************************************************************************************************************************
changed: [server]

TASK [Make sure a service unit is running] ************************************************************************************************************************************************************************
changed: [server]

PLAY RECAP ********************************************************************************************************************************************************************************************************
server                     : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Далее для настройки client используем плейбук client.yml.
```

chumaksa@debpc:~/otus/otus-task22$ ansible-playbook client.yml 

PLAY [Configure clients] ******************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [client]

TASK [Copy file with owner and permissions] ***********************************************************************************************************************************************************************
changed: [client]

TASK [Copy openvpn config file to client] *************************************************************************************************************************************************************************
changed: [client]

TASK [Copy openvpn@.service systemd module] ***********************************************************************************************************************************************************************
changed: [client]

TASK [Make sure a service unit is running] ************************************************************************************************************************************************************************
changed: [client]

PLAY RECAP ********************************************************************************************************************************************************************************************************
client                     : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Теперь всё готово для тестирования VPN подключения в режиме TAP.\
Проверяем сетевые интерфейсы и статус службы.\
Далее запускаем iperf.
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
       valid_lft 85984sec preferred_lft 85984sec
    inet6 fe80::7f:55ff:fe95:78e/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9e:dd:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.10/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9e:ddf5/64 scope link 
       valid_lft forever preferred_lft forever
4: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 100
    link/ether 9e:cc:ff:82:e4:5d brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global tap0
       valid_lft forever preferred_lft forever
    inet6 fe80::9ccc:ffff:fe82:e45d/64 scope link 
       valid_lft forever preferred_lft forever
       
vagrant@server:~$ systemctl status openvpn@server.service 
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-03-06 11:46:31 UTC; 1min 11s ago
   Main PID: 4233 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 1117)
     Memory: 1.0M
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─4233 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf
             
vagrant@server:~$ iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

На client проверяем сетевые интерфейсы и статус службы.\
Далее выполняем тестирование канала.
```

vagrant@client:~$ ip -c a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:7f:55:95:07:8e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 85964sec preferred_lft 85964sec
    inet6 fe80::7f:55ff:fe95:78e/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:57:b1:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.20/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe57:b109/64 scope link 
       valid_lft forever preferred_lft forever
4: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 100
    link/ether 9a:74:ee:02:6e:ef brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.2/24 brd 10.10.10.255 scope global tap0
       valid_lft forever preferred_lft forever
    inet6 fe80::9874:eeff:fe02:6eef/64 scope link 
       valid_lft forever preferred_lft forever
       
vagrant@client:~$ systemctl status openvpn@server.service 
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-03-06 11:46:42 UTC; 1min 52s ago
   Main PID: 4177 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 1117)
     Memory: 952.0K
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─4177 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf
             
vagrant@client:~$ iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 46266 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec  91.3 MBytes   153 Mbits/sec   64    115 KBytes       
[  5]   5.00-10.00  sec  87.5 MBytes   147 Mbits/sec   53   85.1 KBytes       
[  5]  10.00-15.00  sec  84.2 MBytes   141 Mbits/sec   36    106 KBytes       
[  5]  15.00-20.00  sec  85.8 MBytes   144 Mbits/sec   53    102 KBytes       
[  5]  20.00-25.00  sec  86.4 MBytes   145 Mbits/sec   61   89.0 KBytes       
[  5]  25.00-30.00  sec  85.7 MBytes   144 Mbits/sec   36   98.0 KBytes       
[  5]  30.00-35.00  sec  83.9 MBytes   141 Mbits/sec   42    104 KBytes       
[  5]  35.00-40.00  sec  90.6 MBytes   152 Mbits/sec   66    107 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   695 MBytes   146 Mbits/sec  411             sender
[  5]   0.00-40.01  sec   695 MBytes   146 Mbits/sec                  receiver

iperf Done.
```

Далее выполним тестирование в режиме TUN. \
Для этого нам нужно изменить "tuntap" с значения "tap" на "tun" режим в файле "defaults/main.yml". \
Далее последовательно запускаем плейбуки server.yml и client.yml.
```

chumaksa@debpc:~/otus/otus-task22$ ansible-playbook server.yml 

PLAY [Configure servers] ******************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [server]

TASK [Copy openvpn config file to server] *************************************************************************************************************************************************************************
changed: [server]

TASK [Create key] *************************************************************************************************************************************************************************************************
changed: [server]

TASK [Specifying a destination path] ******************************************************************************************************************************************************************************
changed: [server]

TASK [Copy openvpn@.service systemd module] ***********************************************************************************************************************************************************************
ok: [server]

TASK [Make sure a service unit is running] ************************************************************************************************************************************************************************
changed: [server]

PLAY RECAP ********************************************************************************************************************************************************************************************************
server                     : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


chumaksa@debpc:~/otus/otus-task22$ ansible-playbook client.yml 

PLAY [Configure clients] ******************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [client]

TASK [Copy file with owner and permissions] ***********************************************************************************************************************************************************************
changed: [client]

TASK [Copy openvpn config file to client] *************************************************************************************************************************************************************************
changed: [client]

TASK [Copy openvpn@.service systemd module] ***********************************************************************************************************************************************************************
ok: [client]

TASK [Make sure a service unit is running] ************************************************************************************************************************************************************************
changed: [client]

PLAY RECAP ********************************************************************************************************************************************************************************************************
client                     : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

На server проверяем сетевые интерфейсы, статус службы и запускаем iperf.
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
       valid_lft 85761sec preferred_lft 85761sec
    inet6 fe80::7f:55ff:fe95:78e/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9e:dd:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.10/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9e:ddf5/64 scope link 
       valid_lft forever preferred_lft forever
5: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 100
    link/none 
    inet 10.10.10.1/24 brd 10.10.10.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::3718:e3b0:5015:2697/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
vagrant@server:~$ systemctl status openvpn@server.service 
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-03-06 11:50:33 UTC; 25s ago
   Main PID: 4901 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 1117)
     Memory: 944.0K
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─4901 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf
vagrant@server:~$ iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

На client проверяем сетевые интерфейсы, статус службы и запускаем тестирование канала.
```

vagrant@client:~$ ip -c a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:7f:55:95:07:8e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 85796sec preferred_lft 85796sec
    inet6 fe80::7f:55ff:fe95:78e/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:57:b1:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.20/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe57:b109/64 scope link 
       valid_lft forever preferred_lft forever
5: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 100
    link/none 
    inet 10.10.10.2/24 brd 10.10.10.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::ef3d:738f:c645:2e52/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
       
vagrant@client:~$ systemctl status openvpn@server.service 
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-03-06 11:50:42 UTC; 32s ago
   Main PID: 4833 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 1117)
     Memory: 924.0K
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─4833 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf
             
vagrant@client:~$ iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 43258 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   107 MBytes   179 Mbits/sec   87    114 KBytes       
[  5]   5.00-10.00  sec   101 MBytes   170 Mbits/sec   53    115 KBytes       
[  5]  10.00-15.00  sec  86.0 MBytes   144 Mbits/sec   84    110 KBytes       
[  5]  15.00-20.00  sec  87.1 MBytes   146 Mbits/sec  113    114 KBytes       
[  5]  20.00-25.00  sec  86.4 MBytes   145 Mbits/sec   90    180 KBytes       
[  5]  25.00-30.00  sec  88.3 MBytes   148 Mbits/sec   97   95.1 KBytes       
[  5]  30.00-35.00  sec  94.0 MBytes   158 Mbits/sec   41    118 KBytes       
[  5]  35.00-40.00  sec  93.4 MBytes   157 Mbits/sec   78    120 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   743 MBytes   156 Mbits/sec  643             sender
[  5]   0.00-40.00  sec   743 MBytes   156 Mbits/sec                  receiver

iperf Done.
```

Качестве выводов можно отметить, что TAP и TUN представляют собой виртуальные сетевые интерфейсы. \
Режим TAP работает на 2-ом уровне сетевой модели, подходит для проброса vlan и позволяет использовать link-state протоколы динамической маршрутизации.\
Режим TUN работает на 3-ем уровне и не позволяет использовать link-state протоколы динамической маршрутизации.


