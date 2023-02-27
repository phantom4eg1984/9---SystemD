# Домашнее задание по теме: Инициализация системы. Systemd.
## 1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig.

В методичке для выполнения домашнего задания сказано что ```SElinux``` и ```firewalld``` отключены.

Отключаем SElinux:
```
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sudo reboot
```

Отключаем firewalld:
```
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

Создадим файл конфигурации ```/etc/sysconfig/watchlog``` в котором мы зададим путь к файлу лога и ключевое слово которое мы будем мониторить.
```
sudo tee /etc/sysconfig/watchlog << EOF
LOG=/var/log/messages
KEYWORD=ALERT
EOF
```

Создадим скрипт ```/opt/watchlog.sh``` который будет мониторить наш файл и искать ключевое слово ```ALERT``` которое мы указали в переменной ```KEYWORD``` в нашем конфигурационном файле ```/etc/sysconfig/watchlog```.
```
#!/bin/bash
if [ -f /etc/sysconfig/watchlog ]; then
    . /etc/sysconfig/watchlog
else
    echo "/etc/sysconfig/watchlog not found!"
    exit 1
fi

if ! [ -f "$LOG" ]; then
    echo "Log file $LOG not found!"
    exit 1
fi

tail -n0 -f "$LOG" | while read LINE; do
    echo "$LINE" | grep "$KEYWORD" &> /dev/null
    if [ $? = 0 ]; then
        logger "Watchdog: Keyword $KEYWORD found in $LOG"
    fi
done
```

Этот скрипт читает значения из файла конфигурации ```/etc/sysconfig/watchlog```, проверяет наличие лог-файла и ищет ключевое слово в лог-файле. Если ключевое слово найдено, он записывает сообщение в системный лог.

Сделаем файл исполняемым:
```
sudo chmod +x /opt/watchlog.sh
```

Создадим юнит для сервиса. Создадим файл ```/etc/systemd/system/watchlog.service``` со следующим содержимым:
```
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh
StandardOutput=syslog
```

Создадим юнит для таймера. Создадим файл ```/etc/systemd/system/watchlog.timer```
```
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

Это определение сервиса, в котором мы указываем, что он должен запускаться как простой сервис, загружая значения переменных из файла ```/etc/sysconfig/watchlog```, и что его запуском занимается файл ```/opt/watchlog.sh```.

Теперь мы можем запустить наш сервис и добавить его в автозагрузку:
```
sudo systemctl start watchlog.timer
sudo systemctl enable watchlog.timer
```

Проверим работу сервиса. Добавим в лог-файл /var/log/messages строку с ключевым словом ALERT:
```
sudo echo "ALERT: Something happened" >> /var/log/messages
sudo echo "ALERT: Something happened again" >> /var/log/messages
```
Проверим системный лог, чтобы убедиться, что наш сервис работает:
```
sudo tail /var/log/messages
```

В выводе мы можем увидеть такие сообщения:
```
[root@phantom4eg ~]# tail -n 2 /var/log/messages 
ALERT: Something happened
ALERT: Something happened again
```
Проверим статус службы:
```
[root@phantom4eg ~]# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 30 second
   Loaded: loaded (/etc/systemd/system/watchlog.timer; enabled; vendor preset: disabled)
   Active: active (elapsed) since Mon 2023-02-27 00:34:18 UTC; 1min 3s ag
  Trigger: n/a
Feb 27 00:34:18 phantom4eg systemd[1]: Started Run watchlog script every 30 second.
```

Чтобы увидеть журнал службы, выполним:
```
[root@phantom4eg ~]# journalctl -u watchlog.timer
-- Logs begin at Mon 2023-02-27 00:27:41 UTC, end at Mon 2023-02-27 00:36:40 UTC. --
Feb 27 00:34:18 phantom4eg systemd[1]: Started Run watchlog script every 30 second.
```

Мы можем увидеть, что служба периодически запускается и проверяет лог-файл на предмет наличия ключевого слова ```ALERT```.

## 2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.

Справка:
```
spawn-fcgi - это утилита, которая позволяет запускать FastCGI-приложения, такие как PHP, Ruby on Rails и другие, в качестве отдельных процессов. 
Она позволяет запускать приложения в фоновом режиме, под контролем системы и с автоматическим перезапуском в случае падения или завершения процесса. 
Это удобно для обеспечения отказоустойчивости и устранения сбоев в работе веб-приложений.
```

```/etc/rc.d/init.d/spawn-fcgi``` - init скрипт который требяется переписать.

Для этого необходимо раскомментировать строки с переменными в файле ```/etc/sysconfig/spawn-fcgi```
```
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```

Создадим unit-файл с именем ```/etc/systemd/system/spawn-fcgi.service```:
```
[Unit]
Description=Spawn-fcgi startup service
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

Запускаем сервис и добавляем его в автозагрузку:
```
systemctl start spawn-fcgi.service 
systemctl enable spawn-fcgi.service 
```

Проверяем статус нашего сервиса:
```
[root@phantom4eg ~]# systemctl status spawn-fcgi.service 
● spawn-fcgi.service - Spawn-fcgi startup service
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-02-27 09:13:39 UTC; 1min 11s ago
```

## 3. Дополнить Unit-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами:

Для начала требуется отредактировать шаблон в конфигурации файла окружения ```/usr/lib/systemd/system/httpd.service```. Добавим в секцию [Service] следующую строчку:
```
EnvironmentFile=/etc/sysconfig/httpd-%I
```

В файлах окружения ```/etc/sysconfig/httpd-first``` и ```/etc/sysconfig/httpd-second``` задаем параметры для запуска веб-сервера с необходимым для нас конфигурационным файлом:
```
/etc/sysconfig/httpd-first:
OPTIONS=-f conf/first.conf
/etc/sysconfig/httpd-second:
OPTIONS=-f conf/second.conf
```

В каталоге ```/etc/httpd/conf``` создадим два конфига, за основу возьмем конфигурационный файл ```httpd.conf```, скопируем его 2 раза:
```
cp httpd.conf first.conf
cp httpd.conf second.conf
```

Конфигурационный файл ```first.conf``` оставим без изменений, в конфиге ```second.conf``` добавим 2 опции:
```
PidFile /var/run/httpd-second.pid
Listen 8080
```

Запустим инстансы:
```
systemctl start httpd@first
systemctl start httpd@second
```

Проверим что оба порта прослушиваются:
```
[root@phantom4eg conf]# ss -tnlup | grep httpd
tcp   LISTEN 0      128                *:8080            *:*    users:(("httpd",pid=3577,fd=4),("httpd",pid=3576,fd=4),("httpd",pid=3575,fd=4),("httpd",pid=3572,fd=4))
tcp   LISTEN 0      128                *:80              *:*    users:(("httpd",pid=3355,fd=4),("httpd",pid=3354,fd=4),("httpd",pid=3353,fd=4),("httpd",pid=3350,fd=4))
```
