## Проект: отказоустойчивый кластер Bitrix core (percona-cluster,proxysql,keepalived)

Создание рабочего проекта

- [X] веб проект с развертыванием нескольких виртуальных машин должен отвечать следующим требованиям:
- [X] включен https;
- [X] основная инфраструктура в DMZ зоне;
- [X] файрволл на входе;
- [X] сбор метрик и настроенный алертинг;
- [X] везде включен selinux;
- [X] организован централизованный сбор логов;
- [X] организован backup.
- [X] инструкции и описания

<!---
### Чек лист

- [ ] инфраструктура проекта
  - [ ] сервер zabbix
    - [ ] хранение данных на СХД
  - [X] сервер хранения логов (journald)
    - [X] настройка ротации логов
- [ ] настройка Bitrix24 энтерпрайз
  - [X] HA кластер БД
    - [X] балансировка запросов
  - [X] HA кластер web-серверов
  - [ ] HA кластер memcached
  - [X] ротация логов
  - [ ] настройка мониторинга узлов
    - [ ] nagios и munin
    - [X] netdata
    - [ ] zabbix
- [X] настройка распределенной ФС
  - [X] 3 узла в кластере gluster
  - [X] добавить хранилища на узлы
    - [ ] infra 
    - [X] bx-app01, bx-app02, bx-app03 (gv0 volume)
- [X] сетевая балансировка для backend bitrix
  - [X] 2 узла в кластере NLB
  - [X] балансировка нагрузки
  - [ ] RoundRobin DNS
- [ ] блок безопасности
  - [ ] настроить на всех узлах firewall
  - [ ] настроить везде SELinux
  - [ ] на всех web-узлах в инфраструктуре включен https
  - [X] хранение логов на удаленном сервере
  - [X] прямой доступ по ssh к ВМ только по сертификатам
- [ ] отработка нештатных ситуаций, траблшутинг
  - [X] балансировщик
    - [X] потеря одного узла
    - [X] потеря мастера
    - [X] добавление узла в кластер
    - [X] удаление узла из кластера
  - [ ] база данных percona
    - [ ] потеря одного узла
    - [ ] потеря мастера
    - [ ] добавление узла в кластер
    - [ ] удаление узла из кластера
    - [ ] восстановление кластера из резервной копии
  - [X] web-кластер bitrix
    - [X] потеря одного узла
    - [X] потеря мастера
    - [X] добавление узла в кластер
    - [X] удаление узла из кластера
--->

### Описание проекта

Инфраструктура отказоустойчивого кластера Битрикс24.

Инфраструктура состояит из 4 групп ВМ, расположенные в ДЦ под управлением proxmox. Маршрутизаторы, балансировщики, бэкэнд и сервера баз данных:

Хост                | Адрес                 | Роли
--------------------|-----------------------|-----------------------------------------
prj-ros1            | VRRP: 10.100.100.1/24 | Основной маршрутизатор (master)
prj-ros2            | VRRP: 10.100.100.1/24 | Основной маршрутизатор (backup)
bx-nlb01.org.test   | 10.100.100.10/24, VRRP: 10.100.100.100 | nginx, keepalived
bx-nlb02.org.test   | 10.100.100.11/24, VRRP: 10.100.100.100 | nginx, keepalived
infra.org.test      | 10.100.100.250/24     | Сервер DNS, сервер мониторинга, cockpit
bx-app01.org.test   | 10.100.100.20/24      | Master узел Bitrix, web-сервер
bx-app02.org.test   | 10.100.100.21/24      | Master узел Bitrix, web-сервер
bx-app03.org.test   | 10.100.100.22/24      | Master узел Bitrix, web-сервер
bx-db01.org.test    | 10.100.100.30/24      | Сервер БД, master
bx-db02.org.test    | 10.100.100.31/24      | Сервер БД, master
bx-db03.org.test    | 10.100.100.32/24      | Сервер БД, arbitr

Inspiration: https://habr.com/ru/company/nixys/blog/683970/

![image](https://raw.githubusercontent.com/mmmex/project-otus/master/Screens/bitrix2.png)

![image](https://raw.githubusercontent.com/mmmex/project-otus/master/Screens/bitrix1.png)

В данном документе имеются скрытые комментарии, доступные в raw.

### Дэплой проекта

1. Описать конфигурацию инфраструктуры в файле [vms.yml](/infra/roles/proxmox/vars/vms.yml)
2. Выполнить плэйбук, который создаст все необходимые ВМ из образа cloud-init ([роль proxmox](/infra/roles/proxmox/README.md))
3. Выполнить настройку маршрутизации, настроить шлюзы доступа
4. Выполнить настройку DNS сервера, отредактировать [файл настройки DNS](/infra/vars/bind.yml), выполнить: `ansible-playbook bind-setup`
5. Проверить внимательно [инвентарь](/infra/inventory/hosts), привести его в соответствие [сам скрипт](/infra/bitrixenv-setup.yml).
6. Выполнить запуск [скрипта bitrixenv-setup](/infra/bitrixenv-setup.yml): `ansible-playbook bitrixenv-setup.yml`
7. В процессе подготовки нод будет выполнена перезагрузка, ожидание выставлено в 30сек и продолжение. В данном процессе не поддерживается идемпотентность, если например выполнить запуск второй раз, результат будет зависеть от того на каком этапе скрипт прервал работу в прошлый раз и продолжит с того места. 

<!---
Install on all servers:
yum install lvm2 nc mc vim atop iftop htop mytop iotop pciutils dos2unix tar lbzip2 zip unzip bash-completion python36 tmux screen telnet wget fail2ban cockpit pwgen -y

yum install -y policycoreutils policycoreutils-python-utils policycoreutils-python-utils

TODO: role docker
- docker: https://docs.docker.com/engine/install/centos/
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
- docker-compose: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-centos-7
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
- add to end of string PATH: ":/usr/local/bin/" in ~/.bash_profile file
--->

### Маршрутизаторы

В качестве шлюза сети выбран RouterOS v7.7 в составе двух узлов с настроенным VRRP:
- prj-ros01
- prj-ros02

<!---
https://download.mikrotik.com/routeros/7.8/chr-7.8.img.zip
--->

### Балансировка

Варианты реализации:
1. [Nginx+keepalived](https://habr.com/ru/company/netangels/blog/326400/)
2. RoundRobin DNS

В случае выхода из строя одного из сервера балансировки можно использовать роль `load-balancer`, пример плэйбука:
```yaml
---
- hosts: localhost
  roles:
    - role: proxmox
  vars:
    single_host: true
    vmname: bx-nlb02.org.test
    ipaddress: 10.100.100.11/24
    gateway: 10.100.100.1
    vmid: 1101
    cores: 1
    sockets: 1
    memory: 512

- hosts: bx-nlb02.org.test
  become: true
  roles:
    - role: load-balancer 
```
Текущий файл [конфигурации](/infra/roles/load-balancer/defaults/main.yml) роли:
```yaml
---
# defaults file for load-balancer
upstreams_servers:
  - "10.100.100.20:80"
  - "10.100.100.21:80"
  - "10.100.100.22:80"

vip_ip: "10.100.100.100:80"

vrrp_instance: 
  ip: "10.100.100.100"
  auth_pass: "vrrpsecret"

real_servers:
  - ip: "10.100.100.10 80"
    weight: 1
    connect_timeout: 2
  - ip: "10.100.100.11 80"
    weight: 1
    connect_timeout: 2
```

Что выполнит плэйбук:
1. Создаст ВМ в ДЦ с нужными параметрами
2. Выполнит настройку необходимых компонентов и запустит сервер

<!---
https://keepalived.readthedocs.io/en/latest/introduction.html
--->

### Настройка DNS

Плэйбуки:

- [bind_setup.yml](/infra/bind_setup.yml) - настройка нового экземпляра bind
- [bind_modify.yml](/infra/bind_modify.yml) - повторно запустит setup скрипт но выполнится только на bind сервере

[Настройки роли](/infra/vars/bind.yml) и подробное [описание](/infra/roles/bind/README.md).

Использование:
1. Войти в корневую папку прикладной настройки проекта: `cd infra`
2. Выполнить изменения в файле, установить необходимые настройки в [файле](/infra/vars/bind.yml)
3. Выполнить запуск плэйбука для применения изменений: `ansible-playbook bind_apply.yml`
4. После внесения изменений скрипт выполнит перезапуск сервиса автоматически.

### Настройка кластера xtradb

Плэйбуки:

- [xtradb_setup.yml](/infra/xtradb_setup.yml) - настройка кластера percona xtradb

[Полное описание роли](/infra/roles/xtradb_cluster/README.md), [файл конфигурации](/infra/roles/xtradb_cluster/defaults/main.yml). Настройки для каждого хоста располагаются в каталоге [/infra/host_vars](/infra/host_vars/)

Добавление ноды в кластер:
1. Добавить новый узел ВМ, выполнить сетевую настройку.
2. Добавить данный узел в [инвентарь](/infra/inventory/hosts) в разделы `[percona_cluster]` и `[infra]`
3. Добавить настройки кластера, создать файл в `/infra/host_vars/`
4. Войти в корневую папку прикладной настройки проекта: `cd infra`
5. Выполнить запуск плэйбука для применения изменений: `ansible-playbook xtradb_setup.yml`
4. После внесения изменений скрипт может выполнить перезапуск сервиса автоматически (подробнее в [документации к роли](/infra/roles/xtradb_cluster/README.md#restarting-mysql))

Добавление арбитра:

[Arbiter](/infra/roles/xtradb_cluster/README.md#arbitrator)

### Распределенное хранилище данных Gluster

Действия при выходе из строя одной реплики, в данном примере будет расмотрен узел bx-app03:
1. С любого узла в кластере выполняем удаление реплики в томе: `gluster volume remove-brick gv0 replica 2 bx-app03:/data/brick1/gv0 force`
```bash
[root@bx-app02 glusterfs]# gluster volume info
 
Volume Name: gv0
Type: Replicate
Volume ID: 0ce5aeb8-59f0-46a7-8523-7cd2b1cc1d6b
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: bx-app01:/data/brick1/gv0
Brick2: bx-app02:/data/brick1/gv0
Brick3: bx-app03:/data/brick1/gv0
Options Reconfigured:
cluster.granular-entry-heal: on
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
[root@bx-app02 glusterfs]# gluster volume remove-brick gv0 replica 2 bx-app03:/data/brick1/gv0 force
Remove-brick force will not migrate files from the removed bricks, so they will no longer be available on the volume.
Do you want to continue? (y/n) y
volume remove-brick commit force: success
[root@bx-app02 glusterfs]# gluster volume info gv0
 
Volume Name: gv0
Type: Replicate
Volume ID: 0ce5aeb8-59f0-46a7-8523-7cd2b1cc1d6b
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: bx-app01:/data/brick1/gv0
Brick2: bx-app02:/data/brick1/gv0
Options Reconfigured:
cluster.granular-entry-heal: on
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```
2. Удаляем узел bx-app03 из кластера: `gluster peer detach bx-app03`
```bash
[root@bx-app02 glusterfs]# gluster peer status
Number of Peers: 2

Hostname: bx-app01
Uuid: bfa41c6c-0357-4846-9b6c-f8704fe61d0a
State: Peer in Cluster (Connected)

Hostname: bx-app03
Uuid: 5a9d71f7-d4ab-4945-a1b8-d39c189c3fb2
State: Peer in Cluster (Connected)
[root@bx-app02 glusterfs]# gluster peer detach bx-app03
All clients mounted through the peer which is getting detached need to be remounted using one of the other active peers in the trusted storage pool to ensure client gets notification on any changes done on the gluster configuration and if the same has been done do you want to proceed? (y/n) y
peer detach: success
[root@bx-app02 glusterd]# gluster peer status
Number of Peers: 1

Hostname: bx-app01
Uuid: bfa41c6c-0357-4846-9b6c-f8704fe61d0a
State: Peer in Cluster (Connected)
```
3. Выполняем запуск плэйбука setup_bx-app: `ansible-playbook setup_bx-app.yml`

### Мониторинг, логгирование, репозиторий, DNS-сервер

<!---
#### Zabbix

```bash
docker run -p 8081:80 -p 8443:443 --name zabbix-server -e DB_SERVER_HOST="127.0.0.1" -e DB_SERVER_PORT="6033" -e MYSQL_USER="zabbix" -e MYSQL_PASSWORD='Otus2023!' -e MYSQL_DATABASE="zabbixdb" -e ZBX_SERVER_HOST="zbx.org.test" -e PHP_TZ="Europe/Moscow" -d zabbix/zabbix-web-nginx-mysql:tag
```
--->

#### Мониторинг

Мониторинг реализован при помощи [Netdata](https://www.netdata.cloud/)

[Роль netdata](/infra/roles/netdata/README.md)

![image](https://raw.githubusercontent.com/mmmex/project-otus/master/Screens/netdata1.png)

![image](https://raw.githubusercontent.com/mmmex/project-otus/master/Screens/netdata2.png)

![image](https://raw.githubusercontent.com/mmmex/project-otus/master/Screens/netdata3.png)

<!---
Правила firewall для graylog:
```bash
firewall-cmd --remove-service=dhcpv6-client
firewall-cmd --permanent --zone=public --add-port=9000/tcp 
firewall-cmd --permanent --zone=public --add-port=10514/tcp 
firewall-cmd --permanent --zone=public --add-port=10514/udp 
firewall-cmd --permanent --zone=public --add-port=5044/tcp
firewall-cmd --permanent --zone=public --add-service=https 
firewall-cmd --permanent --zone=public --add-service=http 
firewall-cmd --permanent --zone=public --add-service=cockpit
```
Elasticsearch SELinux configs:
```bash
semanage port -m -t http_port_t -p tcp 9200
```
--->

### Bitrix БУС/КП

Порядок восстановления одной ноды после сбоя:
1. Выводим неисправную ВМ из кластера gluster, как описано [выше](#распределенное-хранилище-данных-gluster)
2. Запуск [плэйбука](/infra/repair_bx-app.yml), который выполнит настройку новой ВМ, и установит необходимое ПО.

```yaml
---
- name: proxmox | repair bx-app node | create VM, setup for bx-app0x.org.test after emergency
  hosts: localhost
  roles:
    - role: proxmox
  vars:
    single_host: true
    vmname: bx-app03.org.test
    ipaddress: 10.100.100.22/24
    gateway: 10.100.100.1
    vmid: 1122
    cores: 2
    sockets: 1
    memory: 2048
    scsi1_disk_size: 500

- name: repair bx-app node | starting setup
  hosts: bx-app03.org.test
  become: true
  vars:
    master_ghost: bx-app02.org.test
    is_replacing: true
  roles:
    - role: etc_hosts
    - role: gluster-simple
    - role: bitrix
    - role: netdata
    - role: audit-client
    - role: journald-client
```

<!---
Centos7 installing
---
https://github.com/nextcloud/server/issues/16311

yum install epel-release -y
yum update -y
# Set SELinux to permissive mode
setenforce 0
# set permissive mode in /etc/selinux/config

yum -y install wget unzip
# install remi repo
wget https://rpms.remirepo.net/enterprise/remi-release-7.rpm
rpm -Uvh remi-release-7.rpm epel-release-latest-7.noarch.rpm
yum-config-manager --disable remi-php74
yum-config-manager --enable remi-php80
curl --silent --location https://rpm.nodesource.com/setup_10.x | bash - >/dev/null 2>&1
yum -y install httpd
yum -y install php php-cli php-common \
    php-devel php-gd php-json php-mbstring \
    php-mysqlnd php-opcache php-pdo php-pear \
    php-pecl-apcu php-pecl-zip php-process php-xml php-ldap
yum -y install nginx
yum.repos.d]# yum -y install nodejs
yum.repos.d]# yum -y install redis

# Nginx configure
# downloading file: wget https://dev.1c-bitrix.ru/docs/chm_files/redhat8.zip
# and copy config redhat8/nginx to /etc/nginx
nginx -t
systemctl --now enable nginx
# add firewall rules
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload

#MySQL
# ERROR 2059 (HY000): Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory


#PHP 8.1
# copy two files from redhat8/php.d/: 10-opcache.ini and zx-bitrix.ini to /etc/php.d
# in file zx-bitrix.ini change timezone to Europe/Moscow

yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release enable-only pxc-80
yum remove mariadb -y
yum install Percona-XtraDB-Cluster-client -y
yum install https://github.com/sysown/proxysql/releases/download/v2.4.8/proxysql-2.4.8-1-centos7.x86_64.rpm
yum install proxysql2 -y
systemctl enable --now proxysql
mysql -u admin -padmin -h 127.0.0.1 -P 6032
# https://docs.percona.com/percona-xtradb-cluster/8.0/howtos/proxysql.html#manual-configuration


#HTTPD
# copy all files from redhat8/httpd to /etc/httpd
# in /etc/php.d/10-opcache.ini file change:
opcache.memory_consumption=229
opcache.interned_strings_buffer=57
# than enable and start
systemctl --now enable httpd

#Redis
# copy all from redhat8/redis to /etc/redis/
cd /etc/
usermod -g apache /etc/redis
chown root:apache /etc/redis/ /var/log/redis/
mkdir /etc/systemd/system/redis.service.d
echo -e '[Service]\nGroup=apache' > /etc/systemd/system/redis.service.d/custom.conf
systemctl daemon-reload
systemctl --now enable redis

#Push-server
# install dependencies:
yum install python3 make -y
# install push-server:
cd /opt
wget https://repo.bitrix.info/vm/push-server-0.2.2.tgz
npm install --production ./push-server-0.2.2.tgz
# or version 0.3.0:
wget https://repo.bitrix.info/vm/push-server-0.3.0.tgz
npm install --production ./push-server-0.3.0.tgz
# add symlinks:
ln -sf /opt/node_modules/push-server/logs /var/log/push-server
ln -sf /opt/node_modules/push-server/etc/push-server /etc/push-server
# copy files:
cd /opt/node_modules/push-server
cp etc/init.d/push-server-multi /usr/local/bin/push-server-multi
cp etc/sysconfig/push-server-multi  /etc/sysconfig/push-server-multi
cp etc/push-server/push-server.service  /etc/systemd/system/
ln -sf /opt/node_modules/push-server /opt/push-server
# create tmp directory
echo 'd /tmp/push-server 0770 apache apache -' > /etc/tmpfiles.d/push-server.conf
systemd-tmpfiles --remove --create
# edit /etc/sysconfig/push-server-multi, generate key and add this to SECURITY_KEY field
# `A-Za-z0-9` are accepted only, length maybe long
cat /dev/urandom |tr -dc A-Za-z0-9 | head -c 128
# RUN_DIR - locate PID file of unit
# USER/GROUP - user for unit
GROUP=apache
SECURITY_KEY="SECURITYKEY123456"
RUN_DIR=/tmp/push-server
# start generate config:
/usr/local/bin/push-server-multi configs pub
/usr/local/bin/push-server-multi configs sub
# generated files avaliable in /etc/push-server/ 
# change unit: /etc/systemd/system/push-server.service
[Service]
User=apache
Group=apache
ExecStart=/usr/local/bin/push-server-multi systemd_start
...
# change owner:
chown wwwrun:www /opt/node_modules/push-server/logs /tmp/push-server -RH
# reconfigure
systemctl daemon-reload
systemctl --now enable push-server
# Go to the Push module configuration (site settings) and enable the use of a local push server (latest version). Specify the secret key that you configured earlier in the file /etc/sysconfig/push-server-multi

# Enable site
mkdir /var/www/html/bx-site
cd /var/www/html/bx-site
wget https://www.1c-bitrix.ru/download/scripts/bitrixsetup.php
chown www-data:www-data /var/www/html/bx-site -R
# Create database and user
create database portal;
CREATE USER 'bitrix'@'localhost' IDENTIFIED BY 'PASSWORD';
GRANT ALL PRIVILEGES ON portal.* to 'bitrix'@'localhost';
# Push server
# 

/var/www/html/bx-site/bitrix/backup
--->

### Bitrixenv

**Внимание: в данном разделе и ниже речь идет про решение [1С-Битрикс24: Энтерпрайз 1000](https://www.bitrix24.ru/prices/self-hosted.php)**

Плэйбуки:
- [standalone-bitrixenv.yml] - установка bitrixenv, в результате будет готовая нода к добавлению в кластер.

Установка первой ноды в кластере:
1. Выполнить вход на ВМ: `ssh bx01.org.test`
2. На входе система попросит задать пароль для пользователя bitrix, задаем.
3. Выполнить создание пула
4. Выполнить вход на портал и провести установку и настройку из мастера: `http://bx.org.test`
5. Установить редакцию энтерпрайз.
6. Обновить php и mysql 

**TODO: поднятие первой ноды через wrapper**

**Папка ansible находится на мастере (bx01.org.test) в папке `/etc/ansible/`. Все опреации с кластером желательно выполнять через консольный инструмент `/root/menu.sh`.**

#### [Добавление нового узла в инвентарь консоли управления BitrixVM](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=37&LESSON_ID=8819&LESSON_PATH=3908.8809.8817.8819)

Добавить новый узел возможно несколькими вариантами. Первый вариант через консоль управления BitrixVM. Второй вариант, это использовать `wrapper_ansible_conf`, инструмент является оберткой для ansible скриптов, который позволяет управлять конфигурацией узлов из командной строки. Добавить новый узел в инвентарь можно одной командой запущенной на главном узле с ролью `mgmt`:

```bash
# /opt/webdir/bin/wrapper_ansible_conf -a add -H bx-db03.org.test -i bx-db03.org.test
```

[Справка по инструменту wrapper_ansible_conf](#wrapper_ansible_conf)

#### [Добавление MySQL Slave](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=37&LESSON_ID=9323&LESSON_PATH=3908.8809.8841.9323)

Из консоли управления `/root/menu.sh` добавляем новый slave.

Создать slave mysql:
```bash
# /opt/webdir/bin/bx-mysql -a slave --server bx-db02.org.test \
                   --cluster_login bx_clusteruser --cluster_password Otus2023! \
                   --replica_login bx_repluser --replica_password Otus2023!
```

[Справка по инструменту bx-mysql](#bx-mysql)

#### [Добавление web-сервера](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=37&LESSON_ID=9359)

Создать web-сервер:
```bash
# /opt/webdir/bin/bx-sites -a create_web -H bx-lb01.org.test --fstype lsync
```

Удалить роль web-сервера:
```bash
# /opt/webdir/bin/bx-sites -a delete_web -H bx02.org.test
```

[Справка по инструменту bx-sites](#bx-sites)

#### [Настройка memcached сервера](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=37&LESSON_ID=9341)

[Справка по инструменту bx-mc](#bx-mc-memcached)

#### [Выход из строя основного или дополнительной ноды web-кластера](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=37&LESSON_ID=3481)

##### Сломался дополнительный веб-сервер

<!---
Падает дополнительная нода bx02.org.test, которая имеет роли: memcached,mysql_slave
Падает дополнительная нода bx03.org.test, которая имеет роли: memcached,mysql_slave,web

--->

Убираем дополнительный сервер через соответствующий пункт меню в `/root/menu.sh`.

ИЛИ

В ручную:
1. Убираем ноду из конфигурации upstream
2. Выключаем синхронизацию lsyncd
3. Если доступ к этой ноде есть, но мастер его не видит, то необходимо выключить синхронизацию и на ноде. Это нужно сделать до того как начнете чистить данные или работать с нодой вне пула.

По шагам:
1. Отключить использование ноды на главном сервере `/etc/nginx/bx/site_enabled/upstream.conf`:
```nginx
upstream bx_cluster {
..
server srv2:8080;
}
```

2. Удалить строку с неиспользуемым сервером.
```bash
systemctl restart nginx
```

3. Отключить синхронизацию файлов на дополнительной ноде: (если нода на связи)
```bash
systemctl stop lsyncd-srv2
systemctl disable lsyncd-srv2
```

4. Отключить синхронизацию файлов:
```bash
systemctl stop lsyncd-srv1
systemctl disable lsyncd-srv1
```

##### Сломался основной сервер

1. Включите конфигурационные файлы балансера:
```bash
ln -sf /etc/nginx/bx/site_avaliable/http* /etc/nginx/bx/site_enabled/
ln -sf /etc/nginx/bx/site_avaliable/upstream.conf /etc/nginx/bx/site_enabled/
```

<!---
REMARKS
Now on master /etc/nginx/bx/site_enabled:
[root@bx01 site_enabled]# ls -lll
итого 0
lrwxrwxrwx 1 bitrix bitrix 50 фев 20 22:43 bx_apache_status.conf -> /etc/nginx/bx/site_avaliable/bx_apache_status.conf
lrwxrwxrwx 1 root   root   52 фев 22 11:16 bx_ext_bx.org.test.conf -> /etc/nginx/bx/site_avaliable/bx_ext_bx.org.test.conf
lrwxrwxrwx 1 bitrix bitrix 47 фев 20 22:43 http_balancer.conf -> /etc/nginx/bx/site_avaliable/http_balancer.conf
lrwxrwxrwx 1 root   root   60 фев 22 11:05 https_balancer_bx.org.test.conf -> /etc/nginx/bx/site_avaliable/https_balancer_bx.org.test.conf
lrwxrwxrwx 1 bitrix bitrix 53 фев 20 23:56 nginx_server_status.conf -> /etc/nginx/bx/site_avaliable/nginx_server_status.conf
lrwxrwxrwx 1 bitrix bitrix 46 фев 20 04:52 pool_manager.conf -> /etc/nginx/bx/site_avaliable/pool_manager.conf
lrwxrwxrwx 1 bitrix bitrix 38 фев 20 04:47 push.conf -> /etc/nginx/bx/site_avaliable/push.conf
lrwxrwxrwx 1 bitrix bitrix 42 фев 20 22:43 upstream.conf -> /etc/nginx/bx/site_avaliable/upstream.conf

===

Check, may be needs:
ln -sf /etc/nginx/bx/site_avaliable/pool_manager.conf /etc/nginx/bx/site_enabled/
--->

2. Уберите основной сервер в upstream (`/etc/nginx/bx/site_enabled/upstream.conf`):
```nginx
upstream bx_cluster {
  ip_hash;

  server srv1:8080;
...

  keepalive 10;
}
```

<!---
/opt/webdir/bin/wrapper_ansible_conf -a del -H bx01.org.test -i bx01.org.test
if result with error than:
/opt/webdir/bin/wrapper_ansible_conf -a forget_host --host bx01.org.test
--->

3. Проверьте, что конфигурация рабочая и пеерезапустите NGINX:
```bash
nginx -t
systemctl restart nginx
```

4. По умолчанию ваш сайт работает на ip-адресе сервера srv1. Самый простой вариант переключить DNS запись на адрес сервера srv2. Заранее нужно продумать это вариант, если время кеширования записи большое, то такое переключение может затянутся на часы или дни.

Расположение архивов бэкапа: `/home/bitrix/www/bitrix/backup/`

<!---
1. Remove A name in infra/var/bind.yml
2. Start ansible-playbook bind_modify.yml
--->

---

### Troubleshooting

###### При повторном использовании скрипта `./bitrix-env.sh` возможно завершение с ошибкой:

```bash
[root@bx-lb01 ~]# ./bitrix-env.sh 
====================================================================
Bitrix Environment for Linux installation script.
Yes will be assumed as a default answer.
Enter 'n' or 'no' for a 'No'. Anything else will be considered a 'Yes'.
This script MUST be run as root, or it will fail.
====================================================================
System update in progress. Please wait.
EPEL repository is already configured on the server.
REMI repository is already configured on the server.
Percona repository is already configured on the server.
Getting Bitrix repository configuration. Please wait.
Bitrix repository has been configured.
System update in progress. Please wait.
Installing php packages. Please wait.
Installing bitrix-env package. Please wait.
Error installing package: bitrix-env
Log file path:  /tmp/bitrix-env-C2wuB.log
```

Записи в логе /tmp/bitrix-env-C2wuB.log:
```log
...
Разрешение зависимостей
There are unfinished transactions remaining. You might consider running yum-complete-transaction, or "yum-complete-transaction --cleanup-only" and "yum history redo last", first to finish them. If those don't work you'll have to try removing/installing packages by hand (maybe package-cleanup can help).
...
...
Transaction check error:
  file /usr/lib64/mysql/plugin/dialog.so from install of Percona-Server-server-57-5.7.40-43.1.el7.x86_64 conflicts with file from package mariadb-libs-1:5.5.68-1.el7.x86_64
```

Решение: удалить пакет, который вызывает конфликт
```bash
$ rpm -qa | grep mariadb-libs
mariadb-libs-1:5.5.68-1.el7.x86_64
$ rpm -ev --nodep mariadb-libs-1:5.5.68-1.el7.x86_64
```

###### После добавления узла в кластер ошибка: There are servers that cannot be used! (Errors: 600 или unk)

###### Вариант 1

Проверить наличие открытого ключа главного сервера на стороне узла в `/root/.ssh/authorized_keys`

Закрытый ключ главного узла в каталоге: `/etc/ansible/.ssh/*.bxkey`

Проверить окружение ansible с проблемным узлом:
```bash
[root@bx01 bin]# ansible -vvv bx-db03.org.test -m ping 
ansible 2.7.9
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /bin/ansible
  python version = 2.7.5 (default, Jun 28 2022, 15:30:04) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
Using /etc/ansible/ansible.cfg as config file
/etc/ansible/hosts did not meet host_list requirements, check plugin documentation if this is unexpected
/etc/ansible/hosts did not meet script requirements, check plugin documentation if this is unexpected
Parsed /etc/ansible/hosts inventory source with ini plugin
META: ran handlers
<10.100.100.32> ESTABLISH SSH CONNECTION FOR USER: None
<10.100.100.32> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o 'IdentityFile="/etc/ansible/.ssh/U7T6PHJhC8.bxkey"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o ConnectTimeout=10 -o ControlPath=/root/.ansible/cp/a4439553e4 10.100.100.32 '/bin/sh -c '"'"'echo ~ && sleep 0'"'"''                                                                                    
<10.100.100.32> (255, '', 'Permission denied (publickey).\r\n')
bx-db03.org.test | UNREACHABLE! => {
    "changed": false, 
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey).", 
    "unreachable": true
}
```

Решение: добавить открытый ключ на узел в файл: `/root/.ssh/authorized_keys`

###### Вариант 2

Также возможно причиной может являеться повреждение БД yum.

Симптом:
```bash
[root@bx-lb01 /]# yum update
ошибка: rpmdb: BDB0113 Thread/process 22614/140346805200704 failed: BDB1507 Thread died in Berkeley DB library
ошибка: ошибка(5) db-30973 из dbenv->failchk: BDB0087 DB_RUNRECOVERY: Fatal error, run database recovery
ошибка: невозможно открыть индекс Packages используя db5 -  (-30973)
ошибка: не могу открыть базу данных Packages в /var/lib/rpm
CRITICAL:yum.main:

Error: rpmdb open failed
```

Решение:
```bash
[root@bx-lb01 /]# mkdir /var/lib/rpm/backup
[root@bx-lb01 /]# cp -a /var/lib/rpm/__db* /var/lib/rpm/backup/
[root@bx-lb01 /]# rm -f /var/lib/rpm/__db.[0-9][0-9]*
[root@bx-lb01 /]# rpm --quiet -qa
[root@bx-lb01 /]# rpm --rebuilddb
[root@bx-lb01 /]# yum clean all
Загружены модули: etckeeper, fastestmirror, merge-conf
Сброс источников:base bitrix epel extras nodesource prel-release-noarch ps-57-release-x86_64 pxb-24-release-x86_64 remi remi-php80 remi-safe updates
Cleaning up list of fastest mirrors
Other repos take up 13 M of disk space (use --verbose for details)
```

---

### Nodes

### Приложения

###### wrapper_ansible_conf

```bash
[root@bx01 bin]# /opt/webdir/bin/wrapper_ansible_conf --help
        wrapper_ansible_conf - Managing MySQL servers in the Bitrix pool

Usage:
        wrapper_ansible_conf --help|-h
        wrapper_ansible_conf [--verbose|-v] [--output|-o json|plain] \
            [-a status|view|bx_info] \
            [-H|--host pool_hostname]
        wrapper_ansible_conf [--verbose|-v] [--output|-o json|plain] \
            [-a create|delete_pool|add|del|forget_host] \
            [-H|--host pool_hostname] [-I|--interface interface_name]
        wrapper_ansible_conf [--verbose|-v] [--output|-o json|plain] \
            [-a change_hostname|change_ip|timezone] \
            [-H|--host pool_hostname]
        wrapper_ansible_conf [--verbose\-v] [--output|-o json|plain] \
            [-a check_network|update_network|bx_reboot|bx_password|bx_update|bx_upgrade] \
            [-H|--host pool_hostname]
        wrapper_ansible_conf [--verbose\-v] [--output|-o json|plain] \
            [-a enable_beta_version|disable_beta_version]
        wrapper_ansible_conf [--verbose\-v] [--output|-o json|plain] \
            [-a sshkey|key] \
            [-H|--host pool_hostname]
         wrapper_ansible_conf [--verbose\-v] [--output|-o json|plain] \
            [-a bx_php_upgrade|bx_php_upgrade_php56|bx_php_rollback_php7[01234]|bx_php_upgrade_php7[01234]] \
            [-H|--host pool_hostname]
          wrapper_ansible_conf [--verbose\-v] [--output|-o json|plain] \
            [-a bx_upgrade_mysql57|bx_upgrade_mysql8]
            [-H|--host pool_hostname]

Options:
    --help|-h
                       Print a brief help message and exits.

    --man|-m
                       Prints the manual page and exits.

    --verbose|-v
                       Enable verbose/debug output.

    -H|--host
                       Server name in the Bitrix pool.

    --action|-a
                       Available options for actions in the Bitrix pool.

    -g|--group pool_group
                       Define the pool group name. Ex. memcached or mysql.

    -i|--ip ipv4
                       Define IPv4 adress that will be used in action.

    -u|--user username
                       Define user login.

    -p|--pass password_string
                       Define user password

    -P|--new new_password_string
                       Define user new password

    -k|--key /path/to/ssh/key Define user OpenSHH key
    -I|--interface interface_name Define network interface name
    -t|--timezone timezone Define timezone name
    --host_id pool_host_id Secure host identifier that used when IP address
    is changed
    --hostname hostname New hostname for the pool server

  Actions:
    view|status
                          Get brief information about all servers or one server in the Bitrix pool.

                          wrapper_ansible_conf -a status --host pool_hostname

                          wrapper_ansible_conf -a status -o json

    bx_info
                          Get all information about one server in the Bitrix pool.

                          wrapper_ansible_conf -a bx_info --host pool_hostname

    create
                          Create Bitrix pool on current host.

                          wrapper_ansible_conf -a create --host pool_hostname \
                              --interface interface_name

    delete_pool
                          Remove Bitrix pool on current host.

                          wrapper_ansible_conf -a delete_pool

    forget_host
                          Delete the server from Bitrix pool without updating configuration on it.
                          This option is used when the server is not available.

                          wrapper_ansible_conf -a forget_host --host pool_hostname

    change_hostname
                          Change hostname.
                          This action changes the host name, but does not change its identifier in the Bitrix pool.

                          wrapper_ansible_conf -a change_hostname --host pool_hostname
                              --hostname new_hostname

    change_ip
                          Change IP address for server in the Bitrix pool,
                          that used for connection to the server. 
                          Update ansible configuration.

                          wrapper_ansible_conf -a change_ip --host pool_hostname \
                              --ip new_ipv4

    update_network
                          This action allows to client inform about the need 
                          to update the IP address in the pool for the specified server
                          Update ansible configuration.
                          Host Id is defined in /etc/ansible/ansible-roles on client.

                          wrapper_ansible_conf -a update_network --host_id host_id \
                              --ip new_ipv4

    check_network
                          This action causes the wizard to check for network change requests.
    
                          wrapper_ansible_conf -a check_network

    enable_beta_version|disable_beta_version
                          Enable or disable Bitrix beta version on the server pool.

                          wrapper_ansible_conf -a enable_beta_version|disable_beta_version

    bx_update|bx_upgrade
                          Update all packages(bx_upgrade) or bitrix-packages(bx_update)
                          on servers in the Bitrix pool.

                          wrapper_ansible_conf -a bx_upgrade [--host pool_hostname]

    bx_passwd
                          Change password for defined user

                          wrapper_ansible_conf -a bx_passwd  --host pool_hostname \
                              --user username

    bx_reboot
                          Reboot server
                          wrapper_ansible_conf -a bx_reboot --host pool_hostname

    timezone
                          Change timezone for all servers in the pool.

                          wrapper_ansible_conf -a timezone \
                              --timezone timezone_name [--php]

    sshkey|key
                          Get current OpenSSH key path.

                          wrapper_ansible_conf -a key

    bx_php_upgrade*|bx_upgrade_mysql*
                          Upgrade php or/and mysql version on the server

                          wrapper_ansible_conf -a bx_upgrade_mysql57

    add|del
                          Add or delete host to the Bitrix group. 
                          If the host configuration is not in the Bitrix pool, it will be created.

                          wrapper_ansible_conf -a add --host pool_hostname --ip ip_address

    copy
                          Copy OpenSHH key to the server

                          wrapper_ansible_conf -a copy --key /path/to/file \
                              --ip ip_address --pass ROOT_PASSWORD

    pw
                          Update root password on the server

                          wrapper_ansible_conf -a pw --pass CURRENT_PASSWORD \
                              --new NEW_PASSWORD --ip ip_address
```

###### bx-sites

Параметры для web-кластера:
```bash
Usage: bx-sites [-a create_web|delete_web|web1|web2] -H hostname [--fstype lsyncd|csync2]
Options:
  -a create_web   - create web-server role
  -a delete_web   - delete web-server role
  -a web1|web2
  --fstype        - lsyncd default
```

```bash
[root@bx01 bin]# /opt/webdir/bin/bx-sites --help
Usage: bx-sites [-vh] [-a status] 
Options:
  -h|--help       - show this message
  -v|--verbose    - enable verbose mode.
  -a|--action     - site(s) management actions: status, email, cron, https, create, delete, web
  -s|--site       - site name (default_name is 'default')
  -d|--dbname     - dbname for site(s) 
  -t|--type       - site type, kernel or link (default type is link)
  -u|--user       - create mysql login for site access
  -p|--password   - create logins's password fro site
  -r|--root       - basename of rootdir for site
  -H|--hostname   - add|remove host from web cluster
  --smtphost      - smtp host for email
  --smtpport      - smtp port on server ( default: 25 )
  --smtpuser      - login user for smtp connection
  --password      - password user for smtp connection
  --smtptls       - enable or disable TLS for smtp connection ( default: off )
  --email         - sender address for site
  --enable        - enable options
  --disable       - disable options
  --minute        - munites when task run ( can be *, 0-60 )
  --hour          - hours (0-23)
  --day           - month day (1-31)
  --month         - month (1-12)
  --weekday       - week day (1-7)
  --ntlm_domain   - netbios domain name (ex. TEST)
  --ntlm_fqdn     - full domain name (ex. TEST.EXAMPLE.ORG)
  --ntlm_ads      - domain password server (ex. DC1.TEST.EXAMPLE.ORG)
  --ntlm_login    - domain admin user (default: Administrator)
  --ntlm_password - password for domain user
  --kernel_root   - path to directory with kernel site
  --kernel_site   - kernel site name (optional)
 Ex.
  * get information about site(s) that live on the system
 bx-sites -o json
  * get information about defined site name
 bx-sites -o json -a status -s alice
  * get information about all site with th same kernel
 bx-sites -o json -a list -d sitemanager0
  * create email settings for site
 bx-sites -o json -a email --smtphost=smtp.yandex.ru \
  --smtpuser='ivan@yandex.ru' --password=XXXXXXXXXX \
  --email='ivan@yandex.ru' --smtptls -s alice
  * disable|enable cron tasks for site
 bx-sites -o json -a cron -s alice --disable
 bx-sites -o json -a cron -s alice --enable
  * disable|enable https for site
 bx-sites -o json -a https -s alice --disable
 bx-sites -o json -a https -s alice --enable
  * create site (kernel)
 bx-sites -o json -a create -s alice.bx -t kernel -d alicedb -u aliceuser -p XXXXXXXX -r alice
  or
 bx-sites -o json -a create -s alice.bx -t kernel
  * create site (link), that linked to extternal kernel in /home/bitrix/share
 bx-sites -o json -a create -s alice.bx -r alice
  * create external kernel directory structure and database
 bx-sites -o json -a create -s share --kernel_root /home/bitrix/share
  * delete site (link or kernel)
 bx-sites -o json -a delete -s alice.bx
  * add host to web group ( create web cluster configuration if second host )
 bx-sites -o json -a web -H vm1 --enable
  * remove host from web group
 bx-sites -o json -a web -H vm2 --disable
  * create backup for kernel ( backup all sites and db)
 bx-sites -o json -a backup -d sitemanager0 --enable \
  --minute=10 --hour=23 --day='*' --month='*' --weekday=7 
  * delete backup for kernel ( backup all sites and db)
 bx-sites -o json -a backup -d sitemanager0 --disable
  * add host to domain and enable NTLM auth on default site
 bx-sites -o json -a ntlm \
  --ntlm_domain=TEST --ntlm_fqdn=TEST.EXAMPLE.ORG --ntlm_ads=DC1.TEST.EXAMPLE.ORG \
  --ntlm_login=admin --ntlm_password=XXXXXXXXXXXXXXX --database=sitemanager0
  * update site settings for NTLM usage
 bx-sites -o json -a ntlm_update --database=sitemanager0 
  * get status host in the domain
 bx-sites -o json -a ntlm_status
  * disable|enable restart xmppd or smtpd service by the cron
 bx-sites -o json -a service --service=xmptd --enable --site=default
  * enable|disable usage composite cache by the nginx service
 bx-sites -o json -a composite --enable --site=default
 bx-sites -o json -a composite --disable --site=default
```

###### bx-mysql

```bash
[root@bx01 bin]# /opt/webdir/bin/bx-mysql --help
        bx-mysql - Managing MySQL servers in the Bitrix pool

Usage:
        bx-mysql --help|-h
        bx-mysql [--verbose|-v] [--output|-o json|plain] [-a status|list] \
            [-s|--server pool_hostname]
        bx-mysql [--verbose|-v] [--output|-o json|plain] [-a master|slave|remove] \
            [-s|--server pool_hostname] \
            [--cluster_login cluster_login --cluster_password cluster_password] \
            [--replica_login replica_login --replica_password replica_password]
        bx-mysql [--verbose|-v] [--output|-o json|plain] [-a update|change_password|client_config] \
            [-s|--server pool_hostname] \
            [--password_file /path/to/file]
        bx-mysql [--verbose\-v] [--output|-o json|plain] [-a stop_service|start_service] \
            [-s|--server pool_hostname]

Options:
    --help|-h
                Print a brief help message and exits.

    --man|-m
                Prints the manual page and exits.

    --verbose|-v
                Enable verbose/debug output.

    --server|-s
                Server name in the Bitrix pool.

    --action|-a
                Available options for actions in the Bitrix pool.

    --password_file
                File with MySQL password. Mandatory option for changing root password.

    --cluster_login and --cluster_password
                MySQL login and password that used by Bitrix cluster module.

    --replica_login and --replica_password
                MySQL login and password that used by MySQL replication process.

  Actions:
    status
               Get mysql status in a specific server.

               bx-mysql -a status --server pool_hostname

    list
               Get mysql status for all mysql servers in the Bitrix pool.

               bx-mysql -a list

    slave
               Create MySQL replication on the specified pool server.
    
               bx-mysql -a slave --server pool_hostname \
                   --cluster_login cluster_login --cluster_password cluster_password \
                   --replica_login replica_login --replica_password replica_password

    master
               Change MySQL master.

               bx-mysql -a master --server pool_hostname

    remove
               Delete MySQL service on the specified pool server.
               Only slave-server can be deleted.

               bx-mysql -a remove --server pool_hostname

    change_password
               Update MySQL root password.
               Password value is saved in password file, 
               it will be deleted upfter update process.

               bx-mysql -a change_password --server pool_hostname \
                   --password_file /path/to/password_file

    stop_service|start_service
               Start and stop MySQL service on the specified pool server

               bx-mysql -a stop_service --server pool_hostname
```

###### bx-mc (memcached)

```bash
[root@bx01 bin]# ./bx-mc --help
Usage: bx-mc [-vh] [-a status|list|create|remove|update] [-s server_ip] 
Options:
  -h|--help    - show this message
  -v|--verbose - enable verbose mode.
  -a|--action  - memcached action: list|status|create|remove|update
  -s|--server  - mysql server
 Ex.
  * get status of all memcached servers
 bx-mc -o json
  * get status for one server
 bx-mc -a status -s vm1
  * create memcached instance on the server
 bx-mc -a create -s vm2
  * remove memcached instance from the server
 bx-mc -a remove -s vm2
  * update configuration on all memcached servers
 bx-mc -a update
```

### Notes

https://habr.com/ru/company/netangels/blog/326400/

https://habr.com/ru/post/524688/

https://netpoint-dc.com/blog/ha-lemp-galera/

https://blog.mailon.com.ua/ceph-ansible-%D0%B1%D1%8B%D1%81%D1%82%D1%80%D0%BE-%D0%B8-%D0%B1%D0%B5%D0%B7-%D0%BE%D1%88%D0%B8%D0%B1%D0%BE%D0%BA-%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-ceph-%D0%BA%D0%BB%D0%B0%D1%81/
