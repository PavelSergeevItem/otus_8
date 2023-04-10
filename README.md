***Написать сервис, дополнить unit-файл httpd  .***

**1. Описание задания.**  

Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличие ключевого слова. Файл и слово должны задаваться в /etc/sysconfig.  
Дополнить unit-файл httpd (он же apache2) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.  

**2. Выполенение задания.**  

 **-1 Создание сервиса.**
 Получил корневой доступ `` sudo -i``.
 Cоздал файл с конфигураций для сервиса в директории /etc/sysconfig, из которой сервис будет брать необходимые переменные.  
 `` cat /etc/sysconfig/watchlog``
 
 ```
# Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```
 Потом создал лог файл по адресу /var/log/watchlog.log и написал туда строки, так же туда указал ключевое слово "ALERT".  
 Создал скрпипт, при помощи команды `` cat /opt/watchlog.sh``
 ```
 #!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
```
Добавил права на запуск файл, использую команду `` chmod +x /opt/watchlog.sh``.  
Создал юнит для сервиса. Для этого воспользовался командой ``nano /etc/systemd/system/watchlog``  
Текст юнита:
```
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```
Создал юнит для таймера ``nano /etc/systemd/system/timer``.
Текс юнита: 
```[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```
После запустил timer, при помощи комнады ``systemctl start watchlog.timer``.
Проверил работу. ``tail -f /var/log/messages``  
```
Apr 10 17:48:57 terraform-instance systemd: Starting My watchlog service...
Apr 10 17:48:57 terraform-instance root: Mon Apr 10 17:48:57 +05 2019: I found word,
master!
Apr 10 17:48:57 terraform-instance systemd: Started My watchlog service.
Apr 10 17:49:27 terraform-instance systemd: Starting My watchlog service...
Apr 10 17:49:27 terraform-instance root: Tue Feb 26 16:49:27 +05 2019: I found word,
master!
Apr 10 17:49:27 terraform-instance systemd: Started My watchlog service.
```
Установил spawn-fcgi и необходимые длā него пакеты:
``yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y``  
Далее переписал init скрпипт, используя комнаду `` cat /etc/sysconfig/spawn-fcgi``
Текс скрипта ниже:
```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```
Написал юнит файл. `` cat /etc/systemd/system/spawn-fcgi.service``  
Текс юнит файл:
```
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```
Запустил юнит ``systemctl start spawn-fcgi``
Проверил работу командой ``systemctl status spawn-fcgi``

 **-2 Дополнение unit-файл httpd.**
 

