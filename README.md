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

## 2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку.

### Решение.

Для решения задачи добавим в Vagrantfile ещё одну виртуалку ras. \
Для "наливки" и настройки будем использвать отдельный плейбук ras.yml. \
Этот плейбук выполняет установку openvpn и easy-rsa. \
Также копирует и запускает модуль systemd для openvpn из предыдущей задачи.\
Далее нам необходимо зайти на ras и выполнить инициализацию pki.
```

vagrant@ras:/etc/openvpn$ sudo -i

root@ras:/etc/openvpn# /usr/share/easy-rsa/easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/pki
```

Далее сгенерируем необходимые ключи и сертификаты для сервера.
```

root@ras:/etc/openvpn# echo 'rasvpn' | /usr/share/easy-rsa/easyrsa build-ca nopass

Using SSL: openssl OpenSSL 1.1.1f  31 Mar 2020
Generating RSA private key, 2048 bit long modulus (2 primes)
........+++++
...........+++++
e is 65537 (0x010001)
Can't load /etc/openvpn/pki/.rnd into RNG
140191930611008:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:98:Filename=/etc/openvpn/pki/.rnd
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/pki/ca.crt

root@ras:/etc/openvpn# echo 'rasvpn' | /usr/share/easy-rsa/easyrsa gen-req server nopass

Using SSL: openssl OpenSSL 1.1.1f  31 Mar 2020
Generating a RSA private key
..................+++++
.........................+++++
writing new private key to '/etc/openvpn/pki/private/server.key.e5bzBNiB7i'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [server]:
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/pki/reqs/server.req
key: /etc/openvpn/pki/private/server.key

root@ras:/etc/openvpn# echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req server server

Using SSL: openssl OpenSSL 1.1.1f  31 Mar 2020


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 1080 days:

subject=
    commonName                = rasvpn


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: Using configuration from /etc/openvpn/pki/safessl-easyrsa.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'rasvpn'
Certificate is to be certified until Feb 19 13:48:44 2027 GMT (1080 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/pki/issued/server.crt

root@ras:/etc/openvpn# /usr/share/easy-rsa/easyrsa gen-dh

Using SSL: openssl OpenSSL 1.1.1f  31 Mar 2020
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
.......................................................................................................+......................................................................+..............................................................................................................................+.............................................................................................................+............................+.......+.......................................+...................................................................................................+.......................................+.........................................................+.....................+........................................................................................................................................................................+..................................................................................+.................................................................................................................+................................................+......................+......................................+................................................................................................................................+........................................+..............+...............................................................................................................+...................................................+.............................................................................+......+................................................................+.................................................................+...........................+..............................................+...............+.....................................................................................+...............................................................................................................................................................................................................................................................+..............................................................................................+......+..................................................................................................................................+...............................................................+..........................................................................................................................................+...........................................................................................+..................................................+.........................+.............................+............................................................................................+................................................................................................................................................................................................................+..........................................................................................................................................................................................+..............................................................+..............................................................................................................+......................+....................................+.................................................+................................................................................................................................+................................................................................................................+................................................+......................................+.......................+.....+.+......................................................................................+.............................................................................................................................................................................................+........................+...............+..............................................+...................................................................................................................................................+..........................................................................+.....................................................................................................................................................................................................+............+.....................................................................................................................................................................+..............+....................................................................+.................................................................................+......+......................................................................................+......................................................................................................................+.........................................................................................................................................................................................+...+............+.........+...............................................................................................+..........................................................................................................................................................................................................+...................................................+....................+...................................................................................................................................+....................................................................................................................................................+..................................................................................................+.............................+..................+..........................................................+...+........................................................................+.+....................................................................................................................................................................................................................+.........................+...........................................................................................................................................................................................................+..................................................................+.....+....+..................+..............................................................................................................................................+................................................................................................................+.............................................................................................+......................................................................................................+.....................+...............................................+.........................+.....................................+.....................................+..................................+...+..........................+.................................................+.........+...............................................................................+..+.......+.............................................................+...........................................................................................................................................+...............................................................+....................................................................................................+....................................................+...................................................................................................................................+............................................................................................+............................................................................................................................................................................................+...........+...............................................................................................++*++*++*++*

DH parameters of size 2048 created at /etc/openvpn/pki/dh.pem

root@ras:/etc/openvpn# openvpn --genkey --secret ca.key
```

Далее генерируем сертификаты для клиента.
```

root@ras:/etc/openvpn# echo 'client' | /usr/share/easy-rsa/easyrsa gen-req client nopass

Using SSL: openssl OpenSSL 1.1.1f  31 Mar 2020
Generating a RSA private key
.....................+++++
............................................+++++
writing new private key to '/etc/openvpn/pki/private/client.key.utatAkXduL'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [client]:
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/pki/reqs/client.req
key: /etc/openvpn/pki/private/client.key

root@ras:/etc/openvpn# echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req client client

Using SSL: openssl OpenSSL 1.1.1f  31 Mar 2020


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 1080 days:

subject=
    commonName                = client


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: Using configuration from /etc/openvpn/pki/safessl-easyrsa.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'client'
Certificate is to be certified until Feb 19 13:50:53 2027 GMT (1080 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/pki/issued/client.crt
```

Создадим конфигурационный файл /etc/openvpn/server.conf
```

root@ras:/etc/openvpn# nano server.conf
root@ras:/etc/openvpn# cat server.conf 
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.10.0 255.255.255.0
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

Далее нам необходимо скопировать сертификаты и ключи для клиента на хост машину. \
Для этого был создан отдельный плейбук ras_copy_pki.yml.
```

chumaksa@debpc:~/otus/otus-task22$ sudo ansible-playbook ras_copy_pki.yml 

PLAY [Configure ras] **********************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [ras]

TASK [Copy ca.crt] ************************************************************************************************************************************************************************************************
changed: [ras]

TASK [Copy client.crt] ********************************************************************************************************************************************************************************************
changed: [ras]

TASK [Copy client.key] ********************************************************************************************************************************************************************************************
changed: [ras]

PLAY RECAP ********************************************************************************************************************************************************************************************************
ras                        : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

На хосте создаём конфигурационный файл client.conf.
```

chumaksa@debpc:~$ sudo nano /etc/openvpn/client.conf 
[sudo] password for chumaksa: 
chumaksa@debpc:~$ cat /etc/openvpn/client.conf
dev tun
proto udp
remote 192.168.56.30 1207
client
resolv-retry infinite
remote-cert-tls server
ca ./ca.crt
cert ./client.crt
key ./client.key
route 10.10.10.0 255.255.255.0
persist-key
persist-tun
comp-lzo
verb 3
```

Далее подключаемся к openvpn серверу ras и проверяем его доступность.
```

chumaksa@debpc:/etc/openvpn$ sudo openvpn --config client.conf
2024-03-06 18:11:27 WARNING: Compression for receiving enabled. Compression has been used in the past to break encryption. Sent packets are not compressed unless "allow-compression yes" is also set.
2024-03-06 18:11:27 Note: --cipher is not set. OpenVPN versions before 2.5 defaulted to BF-CBC as fallback when cipher negotiation failed in this case. If you need this fallback please add '--data-ciphers-fallback BF-CBC' to your configuration and/or add BF-CBC to --data-ciphers.
2024-03-06 18:11:27 Note: '--allow-compression' is not set to 'no', disabling data channel offload.
2024-03-06 18:11:27 WARNING: file './client.key' is group or others accessible
2024-03-06 18:11:27 OpenVPN 2.6.3 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] [DCO]
2024-03-06 18:11:27 library versions: OpenSSL 3.0.11 19 Sep 2023, LZO 2.10
2024-03-06 18:11:27 DCO version: N/A
2024-03-06 18:11:27 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.56.30:1207
2024-03-06 18:11:27 Socket Buffers: R=[212992->212992] S=[212992->212992]
2024-03-06 18:11:27 UDPv4 link local: (not bound)
2024-03-06 18:11:27 UDPv4 link remote: [AF_INET]192.168.56.30:1207
2024-03-06 18:11:27 TLS: Initial packet from [AF_INET]192.168.56.30:1207, sid=f4416592 42fdba90
2024-03-06 18:11:27 VERIFY OK: depth=1, CN=rasvpn
2024-03-06 18:11:27 VERIFY KU OK
2024-03-06 18:11:27 Validating certificate extended key usage
2024-03-06 18:11:27 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
2024-03-06 18:11:27 VERIFY EKU OK
2024-03-06 18:11:27 VERIFY OK: depth=0, CN=rasvpn
2024-03-06 18:11:27 Control Channel: TLSv1.3, cipher TLSv1.3 TLS_AES_256_GCM_SHA384, peer certificate: 2048 bit RSA, signature: RSA-SHA256
2024-03-06 18:11:27 [rasvpn] Peer Connection Initiated with [AF_INET]192.168.56.30:1207
2024-03-06 18:11:27 TLS: move_session: dest=TM_ACTIVE src=TM_INITIAL reinit_src=1
2024-03-06 18:11:27 TLS: tls_multi_process: initial untrusted session promoted to trusted
2024-03-06 18:11:28 SENT CONTROL [rasvpn]: 'PUSH_REQUEST' (status=1)
2024-03-06 18:11:28 PUSH: Received control message: 'PUSH_REPLY,topology net30,ping 10,ping-restart 120,ifconfig 10.10.10.6 10.10.10.5,peer-id 1,cipher AES-256-GCM'
2024-03-06 18:11:28 OPTIONS IMPORT: --ifconfig/up options modified
2024-03-06 18:11:28 net_route_v4_best_gw query: dst 0.0.0.0
2024-03-06 18:11:28 net_route_v4_best_gw result: via 192.168.248.166 dev wlx18d6c717ce87
2024-03-06 18:11:28 ROUTE_GATEWAY 192.168.248.166/255.255.255.0 IFACE=wlx18d6c717ce87 HWADDR=18:d6:c7:17:ce:87
2024-03-06 18:11:28 TUN/TAP device tun0 opened
2024-03-06 18:11:28 net_iface_mtu_set: mtu 1500 for tun0
2024-03-06 18:11:28 net_iface_up: set tun0 up
2024-03-06 18:11:28 net_addr_ptp_v4_add: 10.10.10.6 peer 10.10.10.5 dev tun0
2024-03-06 18:11:28 net_route_v4_add: 10.10.10.0/24 via 10.10.10.5 dev [NULL] table 0 metric -1
2024-03-06 18:11:28 Initialization Sequence Completed
2024-03-06 18:11:28 Data Channel: cipher 'AES-256-GCM', peer-id: 1, compression: 'lzo'
2024-03-06 18:11:28 Timers: ping 10, ping-restart 120

chumaksa@debpc:~$ ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.18 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.17 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.16 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=1.19 ms

--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 1.155/1.171/1.186/0.011 ms
```




