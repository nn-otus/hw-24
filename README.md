## ДЗ-24 ##
##### Цель домашнего задания #####
Научиться создавать пользователей и добавлять им ограничения
##### Описание домашнего задания #####
Ограничить доступ к системе для всех пользователей, кроме группы администраторов, в выходные дни (суббота и воскресенье), за исключением праздничных дней.

⭐️ Задание со звездочкой
Предоставить определённому пользователю доступ к Docker и право перезапускать Docker-сервис.

### Настройка запрета для всех пользователей (кроме группы admin) логина в выходные дни (Праздники не учитываются) ###

1. Подключаемся к нашей созданной ВМ: 
```
[nn@alma96srv07 ~]$ ssh nn@u24srv03
ED25519 key fingerprint is SHA256:pXy8JTN5ogjjG/XKdxDUTHGjImPhuII1DqS9E296vXg.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:1: u24-clone3
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'u24srv03' (ED25519) to the list of known hosts.
nn@u24srv03's password:
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.15.0-061500-generic x86_64)
...
To restore this content, you can run the 'unminimize' command.
Last login: Wed Oct  1 08:08:48 2025 from 192.168.250.250
nn@u24srv03:~$

```
2. Переходим в root-пользователя: sudo -i
```
nn@u24srv03:~$ su -
Password:
root@u24srv03:~#

```
3. Создаём пользователей otusadm и otus: sudo useradd otusadm && sudo useradd otus
```
root@u24srv03:~# useradd otusadm
root@u24srv03:~# useradd otus
root@u24srv03:~#
```
4. Создаём пользователям пароли:
```
root@u24srv03:~# passwd otusadm
New password:
Retype new password:
passwd: password updated successfully
root@u24srv03:~#
root@u24srv03:~# passwd otus
New password:
Retype new password:
passwd: password updated successfully
root@u24srv03:~#
```
5. Создаём группу admin: sudo groupadd -f admin
```
root@u24srv03:~#  groupadd -f admin
root@u24srv03:~#
```
6. Добавляем пользователей root и otusadm в группу admin:   

(Добавление пользователя otusadm в группу не делает его админом)
```
root@u24srv03:~# usermod otusadm -a -G admin && usermod root -a -G admin
root@u24srv03:~#
root@u24srv03:~# cat /etc/group | grep admin
admin:x:1004:otusadm,root

```
 Проверяем, что вновь созданные пользователи могут подключаться по SSH к нашей ВМ
 ```
$ whoami
otusadm
$
...
$ whoami
otus
 ```
8. Создаём файл-скрипт /usr/local/bin/login.sh
```
root@u24srv03:~# nano  /usr/local/bin/login.sh
#!/bin/bash
# PAM-скрипт для контроля доступа по группе и дню недели
# Логирование через journal и отдельный файл

user="$PAM_USER"
host="$PAM_RHOST"
day=$(date +%u)  # 1=Пн ... 6=Сб, 7=Вс
LOGFILE="/var/log/pam-login.log"

# Функция логирования
log() {
    local MSG="$1"
    # systemd journal
    logger -t pam-login "user=$user host=$host msg=\"$MSG\""
    # отдельный лог-файл
    echo "$(date '+%F %T') user=$user host=$host msg=\"$MSG\"" >> $LOGFILE
}

# Проверяем, состоит ли пользователь в группе admin
if id -nG "$user" | grep -qw "admin"; then
    log "доступ разрешён (пользователь в группе admin)"
    exit 0
fi

# Пользователь НЕ в группе admin. Проверка выполняется в понедельник, добавляем в условия $day -eq 1 
if [[ $day -eq 6 || $day -eq 7 ||  $day -eq 1 ]]; then
    log "доступ запрещён (выходной, пользователь не в группе admin)"
    exit 1
else
    log "доступ разрешён (будний день, пользователь не в группе admin)"
    exit 0
fi
# ************ END ***********
```
9. Добавляем права на исполнение файла: chmod +x /usr/local/bin/login.sh
```
root@u24srv03:~# chmod +x /usr/local/bin/login.sh
root@u24srv03:~# ls -l !$
ls -l /usr/local/bin/login.sh
-rwxr-xr-x 1 root root 719 Oct  6 12:43 /usr/local/bin/login.sh
root@u24srv03:~#
```
10. Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:
```
*********
root@u24srv03:~# cat /etc/pam.d/sshd
auth     include    common-auth
auth     required   pam_exec.so debug /usr/local/bin/login.sh

account  include    common-account
account  required   pam_nologin.so

password include    common-password

session  include    common-session
session  optional   pam_motd.so

root@u24srv03:~#

```
11. Проверяем возможность логина пользователей otus и otusadm
```

root@u24srv03:~#
root@u24srv03:~# ssh otus@u24srv03
...
otus@u24srv03's password:
Permission denied, please try again.

$ ssh otusadm@u24srv03
...
otusadm@u24srv03's password:
Could not chdir to home directory /home/otusadm: No such file or directory
$ whoami && hostname
otusadm
u24srv03
$
```
Смотрим лог: 
```
root@u24srv03:~# journalctl -t pam-login -n 2
Oct 06 14:24:48 u24srv03 pam-login[26302]: user=otus host=192.168.240.122 msg="доступ запрещён (выходной, пользователь не в группе admin)"
Oct 06 14:26:00 u24srv03 pam-login[26311]: user=otusadm host=192.168.240.122 msg="доступ разрешён (пользователь в группе admin)"
root@u24srv03:~#
```
## ДЗ-24 выполнено
