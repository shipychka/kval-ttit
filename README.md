# kval-ttit

!Начало
Если в задании не будут использоваться встроенные репозитории, а будет возможность скачивать все пакеты из интернета, необходимо отключить проверку пакетов через cdrom зайдя по пути
Nano /etc/apt/sources.list 
и закомментировать находящуюся там строку.
Для коректной работы сети используй NMTUI только на машине ISP. На остальных машинах настройку IP-адресации производите через файл конфигурации /etc/network/interfaces

!Задание 1 модуль 1
1. Произведите базовую настройку устройств  
● Настройте имена устройств согласно топологии. Используйте полное доменное имя
 Примечание: для выполнения данного задания необходимо постоянное изменение имени каждого устройства, указанного на топологии
 !!!(временно изменение, действует только до перезагрузки системы и не является верным выполнением задания)
Решение:
Для фиксированного изменения имени компьютера, необходимо использовать команду: 
hostnamectl set-hostname Имя устройства
Для изменения имени компьютера в текущем сеансе без перезагрузки можно воспользоваться командой: 
newgrp
● Локальная сеть в сторону HQ-SRV(VLAN100) должна вмещать не более 64 адресов 
● Локальная сеть в сторону HQ-CLI(VLAN200) должна вмещать не более 16 адресов 
● Локальная сеть в сторону BR-SRV должна вмещать не более 32 адресов ● Локальная сеть для управления(VLAN999) должна вмещать не более 8 адресов 
● Сведения об адресах занесите в отчёт, в качестве примера
Имя устройства	IP
HQ-CLI	192.168.200.2 255.255.255.240 — к HQ-RTR
ISP	172.16.4.1 255.255.255.240 — к HQ-R
172.16.5.1 255.255.255.240 — к BR-R
HQ-RTR	192.168.100.1 255.255.255.192 — к HQ-SRV
172.16.4.2 255.255.255.240 — к ISP
192.168.200.1 255.255.255.240 — к HQ-CLI
HQ-SRV	192.168.100.2 255.255.255.192 — к HQ-RTR

BR-RTR	192.168.0.1 255.255.255.224 — к BR-SRV
172.16.5.2 255.255.255.240— к ISP
BR-SRV	192.168.0.2 255.255.255.224— к BR-RTR
Сеть управления (VLAN 999)	192.168.999.0 255.255.255.248
Следующим шагом необходимо установить выбранные IP адреса на соответствующие машины, для этого существуют 2 способа.

Первый способ: через network-manager
!!!Второй способ: через редактирования конфига интерфейсов
Вариант ручной настройки без использования любых программ (в случае если не будет возможности установки nmtui или она будет запрещена). Перед установкой интерфейсов необходимо воспользоваться командой IP A для определения имён 7 интерфейсов, находим незаполненный интерфейс, в примере ниже незаполненным интерфейсов является ens256
!!! Определив интерфейс, необходимо воспользоваться командой для просмотра и изменения конфигураций интерфейсов 
nano /etc/network/interfaces 
или
vi /etc/network/interfaces
И затем сконфигурировать настройки интерфейсов в соответствии с таблицей адресации по примеру, представленному на скриншоте ниже

*Inis file describes the network intertaces avallable on your sys and how to activate them. For more information, see interfaces(
source /etc/network/interfaces.d/*
# The loopback network interface
auto lo
iface lo inet loopback
# The primary network interface
allow-hotplug ens192
iface ens192 inet dhcp

auto ens224
iface ens224 inet static
address 172.16.4.1
netmask 255.255.255.240

auto ens256
iface ens256 inet static
address 172.16.5.1
netmask 255.255.255.240
Пример настройки интерфейсов ISP

!!!Для настройки VLAN на роутере HQ-RTR нужно скачать утилиту VLAN:
apt install vlan. Также нужно установить модуль 8021: 
modprobe 8021q 
и добавить его в автозапуск echo 8021q >> /etc/modules

!!! Теперь можно преступить к настройке файла конфигурации /etc/network/interface. Он должен выглядеть следующим образом:

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens192
iface ens192 inet static
address 172.16.4.2
netmask 255.255.255.240
gateway 172.16.4.1

auto ens224
iface ens224 inet static
address 102.168.100.1
netmask 255.255.255.192

auto ens224:1
iface ens224:1 inet static
address 102.168.200.1
netmask 255.255.255.240

auto ens224.100
iface ens224 inet static
address 102.168.100.3
netmask 255.255.255.192
vlan-raw-device ens224

auto ens224.200
iface ens224 inet static
address 102.168.200.3
netmask 255.255.255.240
vlan-raw-device ens224:1

Рисунок 7 — Настройка интерфейсов HQ-RTR

!!!2) Настройка ISP
iptables:
apt-get install iptables iptables-persistent
Затем нужно создать правила iptables
iptables –t nat –A POSTROUTING –s 172.16.4.0/28 –o ens192 –j MASQUERADE  
iptables –t nat –A POSTROUTING –s 172.16.5.0/28 –o ens192 –j MASQUERADE  
iptables-save > /etc/iptables/rules.v4 
Перезапускаем iptables: systemctl restart iptables  
Для проверки вводим команду iptables –L –t nat
После настройки на интерфейсах ISP может слететь Ip. Также на роутерах и ISP нужно зайти в файл /etc/sysctl.conf и раскомментировать строку «net.ipv4.ip_forward=0» и привести её к виду «net.ipv4.ip_forward=1». Также для работы nat и доступа в интернет на роутерах в качестве gateway указать адрес ISP.

!!!На HQ-RTR и BR-RTR Нужно зайти в файл /etc/resolv.conf и оставить там только одну строку: nameserver 1.1.1.1

Правила для HQ-RTR 
iptables –t nat –A POSTROUTING –s 192.168.100.0/26 –o ens192 –j MASQUERADE  

!!! 3. Создание локальных учётных записей

adduser sshuser
Затем появится поле ввода пароля как показано на рисунке 9

root@hq-cli:/home/locadm# useradd sshuser -u 1010 -U
root@hq-cli: /home/locadm# passwd sshuser
New password:
Retype new password:
passwd: password updated successfully
root@hq-cli: /home/locadm#
Рисунок 9 — окно ввода пароля при создании пользователя 

!!!Для смены id используется команда usermode -u 1010 sshuser
Так же возможно понадобится выдать Root права для данных клиентов это можно выполнить посредством команды visudo
в открывшемся окне необходимо вписать изменения для каждой новой созданной учётной записи как показано на рисунке 10

# User privilege specification
root ALL= (ALL: ALL) ALL
sshsuer ALL=(ALL:ALL) ALL
Рисунок 10 — выдача Root прав пользователям

!!! 5. Настройка безопасного удалённого доступа
Первым делом необходимо перейти по пути nano /etc/ssh/sshd_config где в окне конфигурации нам необходимо на HQ-SRV найти строки и изменить значения как указанно на рисунке 10

Port 2024
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
AllowUsers sshuser
#HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
#Hostkey /etc/ssh/ssh_host_ed25519_key
Рисунок 10 — смена порта доступа по ssh и права подключения только определённого пользователя

#LoginGraceTime 2m
#PermitRootLogin prohibit-password
#StrictModes yes
MaxAuthTries 2
#MaxSessions 10
Рисунок 11— Ограничение попыток авторизации

# no default banner path
Banner /etc/ssh-banner
Рисунок 12 — Указание файла баннера

!!!Для настройки баннера нужно зайти в файл /etc/ssh-banner и написать следующее: Authorized acces only
Для применения конфигурации необходимо перезагрузить службу командой systemctl restart ssh
Для проверки доступа нужно написать команду: ssh sshuser@192.168.100.2 -p 2024
Где -p — указание порта. Без указания порта подключится не получится
Также, при неправильном вводе пароля должно вывестись сообщение баннера

!!! 6) Реализация GRE-туннеля между офисами

Нужно зайти в файл /etc/modules и добавить там строку ip_gre:

# /etc/modules: kernel modules to load at boot time.
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
# Parameters can be specified after the module name.
ip_ gre

!!! Вся последующая настройка проводится в файле /etc/network/interfaces

auto tun1
iface tuni inet tunnel
address 10.10.0.1
netmask 255.255.255.252
mode gre
local 172.16.4.2
endpoint 172.16.5.2
ttl 64
Рисунок 14 - Настройка GRE на HQ-RTR

auto tun1
iface tuni inet tunnel
address 10.10.0.2
netmask 255.255.255.252
imode gre
local 172.16.5.2
endpoint 172.16.4.2
tt1 64

Рисунок 15 - Настройка GRE на BR-RTR

Ping 10.10.0.1 и ping 10.10.0.2 для проверки работоспособности туннеля с обеих сторон:


8. Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической маршрутизации используйте link state протокол на ваше усмотрение.

Решение: Первым делом необходимо установить пакеты FRR, для этого необходимо воспользоваться командой:
apt install frr
Следующим шагом необходимо произвести изменения конфигурационных  файлов 

nano /etc/frr/daemons
и изменить параметры на YES для протокола OSPF
Рисунок 17 — настройка конфигурации FRR

После сохранения конфига, следующим шагом необходимо, перезапустить frr.service командой 
systemctl restart frr 
Далее, после перезагрузки, посредством команды vtysh перейти в режим конфигурирования (Настройки идентичны Cisco IOS).
Рисунок 18 — пример конфигурационного окна
Посредством команд:
Conf t
router ospf
перейти к конфигурированию протокола ospf
Настройка производится посредством объявления 
ospf router-id x.x.x.x
и прилегающих к маршрутизатору сетей
network x.x.x.x/x area x
как показано на рисунке 9

rtr.au-team. irpo# cont t
rtr.au-team. irpo(config) router ospf
rtr.au-team. irpo(config-router)# passive-interface default
rtr.au-team. irpo(config-router)# network 192.168.100.0/26 area 0
ntr. au-team. inpo(config-router)# network 192.168.200.0/28 area
rtr. au-team. irpo(config-router)# network 10.10.0.0/30 area 0
ntr. au-team. irpo(config-router)#

Рисунок 19 — пример настройки OSPF на HQ-RTR

#br-tr.au-team. irpo (config-router)# area 0 authentication
#br-tr.au-team. irpo (config-router)# c
jor-ptr.au-team. irpo(config-router)# exit
jbr-tr.au-team. irpo (config)# interface tun1
dor-ptr.au-team. irpo(config-if)# no ip ospf passive
ybr-tr.au-team. irpo(config-if)# ip ospf authentication
ebr-rtr.au-team. irpo(config-if)# ip ospf authentication-key password

Рисунок 20— Включение авторизации и последующая настройка интерфейса

!!!После завершения конфигурации в frr, необходимо записать конфигурацию в память устройства, командой write, иначе при перезагрузке frr или устройства, все настройки вернутся к дефолтным
Для этого необходимо
Для завершения настройки сети необходимо сконфигурировать настройку для передачи пакетов между сетями в файле nano /etc/sysctl.conf
переменную net.ipv4.ip_forward=1 необходимо раскоментить и сохранить изменения в файле, и применить изменения командой sysctl -p

Рисунок 21 — настройка пересылки пакетов в режиме маршрутизатора

Примечание: при каждой перезагрузке устройства, данная настройка будет изменятся обратно, что связано с загрузкой операционной системы на виртуальной машине для того, чтобы снова включить пересылку пакетов необходимо прописать sysctl -p
Идентичная настройка проводится на BR-RTR, только указывается другая подсеть(192.168.0.0/27). Указание сети туннеля и настройки авторизации абсолютно идентичны


8) Настройка динамической трансляции адресов
Точно также как и для ISP устанавливаем пакеты iptables: 
apt install iptables iptables-persistent
iptables –t nat –A POSTROUTING –s 192.168.100.0/26 –o ens192 –j MASQUERADE  

iptables –t nat –A POSTROUTING –s 192.168.200.0/28 –o ens192 –j MASQUERADE  

Правило для BR-RTR
iptables –t nat –A POSTROUTING –s 192.168.0.0/27 –o ens192 –j MASQUERADE  

Затем эти правила нужно сохранить: iptables-save > /etc/iptables/rules.v4
После этого перезапускаем службу

9.Настройте автоматическое распределение IP-адресов на роутере HQ-R. 
a. Учтите, что у сервера должен быть зарезервирован адрес. 
Первым шагом необходимо на машине HQ-R установить dhcp server командой 
apt install isc-dhcp-server
После установки пакета следующим шагом необходимо сконфигурировать файл для указания интерфейсов прослушивания DHCP сервера зайти можно с помощью команды

nano /etc/default/isc-dhcp-server
и настроить интерфейс, направленный в сторону клиента, если в сети подразумевается DHCP-relay, то 2 интерфейса в сторону клиента, и в сторону сети откуда исходит запрос. Строка v6 закоментирована, чтобы DHCP даже не думал пробовать его раздавать
Рисунок 22 — Указание интерфейса для передачи адреса

!!!Далее необходимо настроить 2 конфигурационных файла для IPv4 для IPv6
Которые можно найти по путям nano /etc/dhcp/dhcpd.conf и nano /etc/dhcp/dhcpd6.conf соответственно

subnet 192.168.200.0 netmask 255.255.255.240 {
range 192.168.200.4 192.168.200.14;
option domain-name-servers 192.168.100.2;
option domain-name
"au-team. inpo";
option routers 192.168.200.1;
default-lease-time 600;
max-lease-time 7200;

Рисунок 23— Пример настройки DHCP для ipv4 без Relay

ddns-update-style interim — способ автообновления базы dns
authoritative — делает сервер доверенным
subnet — указание сети
range — пул адресов
option routers — шлюз по умолчанию
Перезапускаем службу DHCP: systemctl restart isc-dhcp-server

Для проверки на HQ-CLI нужно указать получение адреса по DHCP
allow-hotplug ens192
iface ens192 inet dhcp
#address 192.168.200.2
#netmask 255.255.255.240
#gateway 192.168.200.1
Рисунок 24 — Настройка интерфейса

Прописываем systemctl restart networking для применения и проверяем выданный ip-адрес командой ip -c a

10. Настройте DNS-сервер на сервере HQ-SRV:

Вся настройка будет происходить на сервере HQ-SRV
Первым делом необходимо установить пакеты для dns командой
apt install bind9 dnsutils
где:
bind9 — пакеты для создания dns сервера
dnsutils — дополнительные пакеты, которые помогут проверить работоспособность (команда host)
Следующим шагом необходимо создать зоны для прямого и обратного просмотра dns
Для этого переходим по пути nano /etc/bind/named.conf.default-zones и создаём зоны как показано на скриншотах ниже

zone "au-team. inpo" {
         type master;
         file "/etc/bind/au-team. irpo";
} ;
zone "100.168.192. in-addr .arpa" {
         type master;
         file "/etc/bind/au-team. irpo_obr";
};
zone "200.168.192. in-addr.arpa"{
         type master;
         file "/etc/bind/au-team. inpo_haobr";
} ;

Рисунок 26 — зоны для hq.work(На скриншоте указаны 2 обратные зоны, т. к. у HQ-CLI и HQ-SRV IP-адреса заканчиваются на одинаковые октеты и из-за этого DNS может не работать)

где:
zone — создаваемая зона
type — выбор между первичным и вторичным dns. (Master и Slave)
file — расположение конфигурационного файла зоны
allow-update — разрешение динамических обновлений
где zone:
hq.work — зона прямого просмотра
in-addr.arpa — зона обратного просмотра ipv4

Следующим шагом необходимо создать конфигурационные вайлы для наших зон. Это можно сделать, скопировав стандартные шаблоны командой cp
Пример:
cp /etc/bind/db.local /etc/bind/au-team.irpo — создание файла для прямой зоны
cp /etc/bind/db.127 /etc/bind/ au-team.irpo_obr — создание обратной зоны ipv4

Первым шагом сконфигурируем зону прямого просмотра, переходим по пути
nano /etc/bind/au-team.irpo и конфигурируем файл как показано на скриншоте ниже

Рисунок 27 — зона прямого просмотра 
Где:
NS запись — обозначение сервера отвественного за разрешение запросов к dns
A запись — основная запись для зоны прямого просмотра по протоколу ipv4
CNAME — необязательный параметр, для указания альтернативного имени записи
Вторым шагом настроим зону обратного просмотра как указано на скриншоте ниже
Зона находится по пути 
nano /etc/bind/au-team.irpo_obr
Рисунок 28— настройка зоны обратного просмотра hq.work для ipv4

Рисунок 29— настройка второй зоны обратного просмотра hq.work для ipv4
Где:
PTR запись — основная запись для зоны обратного просмотра
Проверка выполняется посредством команд
host IP-адрес
host имя машины


!!!Задание 11
Настройка даты и времени согласно месту проведения экзамена

timedatectl set-timezone Asia/Tomsk

Команда для проверки: timedatectl

Для выполнения задания будет использоваться утилита MDADM. Её нужно установить: apt install mdadm .
Далее прописываем команду lsblk для отображения дисков(Рисунок 31)

Для создания рейда будут использоваться диски sdb, sdc и sdd. Для начала на них нужно создать разделы командой fdisk /dev/sdb. Сначала вводим g, чтобы создать раздел, затем вводим n и прокликиваем enter. Для выхода и сохранения вводим w. (Рисунок 32)

Рисунок 32 — создание разделов

Всё тоже самое проделываем с дисками sdc и sdd.



Теперь можно приступить к созданию рейд-массива(Рисунок 33):
mdadm --create --verbose /dev/md0 -l 5 -n 3 /dev/sdb1 /dev/sdc/1 /dev/sdd1 






