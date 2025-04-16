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
 !!!(временно BR-RTR	192.168.0.1 255.255.255.224 — к BR-SRV
                      172.16.5.2 255.255.255.240— к ISP
              BR-SRV	192.168.0.2 255.255.255.224— к BR-RTR
              Сеть управления (VLAN 999)	192.168.999.0 255.255.255.248!!!
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


