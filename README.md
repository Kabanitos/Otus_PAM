# Описание домашнего задания
1) Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

* дать конкретному пользователю права работать с докером
и возможность рестартить докер сервис
# Настройка запрета для всех пользователей (кроме группы Admin) логина в выходные дни (Праздники не учитываются)
Подключаемся к созданной VM: **vagrant ssh**.
Переходим в root-пользователя: **sudo -i**.
Создаем пользователя otusadm и otus: **sudo useradd otusadm && sudo useradd otus**
Задаем пользователям пароли: **echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus**
Создаем группу admin: **sudo groupadd -f admin**
Добавляем пользователя vagrant,otusadm и root в группу admin:
   + **sudo usermod otusadm -a -G admin**
   + **sudo usermod root -a -G admin**
   + **sudo usermod vagrant -a -G admin**

    
    
```
vagrant ssh 
[vagrant@pam ~]$ sudo -i 
[root@pam ~]# sudo useradd otusadm && sudo useradd otus
[root@pam ~]# echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
Changing password for user otus.
passwd: all authentication tokens updated successfully.
[root@pam ~]# sudo groupadd -f admin
[root@pam ~]# sudo usermod otusadm -a -G admin
[root@pam ~]# sudo usermod root -a -G admin
[root@pam ~]# sudo usermod vagrant -a -G admin
```
После создания пользователей, проверяем что они могут подключаться к нашей VM по SSH. 
```
ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
ED25519 key fingerprint is SHA256:VZ8b+qJcYRzvVWI+nfN2z6MfWCPmdFippkFueyPuDjA.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.57.10' (ED25519) to the list of known hosts.
otus@192.168.57.10's password: 
Permission denied, please try again.
otus@192.168.57.10's password: 
Permission denied, please try again.
otus@192.168.57.10's password: 
Last failed login: Sun Feb 26 13:59:03 UTC 2023 from 192.168.57.1 on ssh:notty
There were 2 failed login attempts since the last successful login.
[otus@pam ~]$
```
```
ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
[otusadm@pam ~]$
```
Мы проверили, что пользователи otus и otusadm имеют доступ к нашей VM по SSH. Далее будем настраивать правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни.
Проверим, что пользователи **root**, **vagrant** и **otusadm** есть в группе admin:

```
cat /etc/group | grep admin
printadmin:x:994:
admin:x:1003:otusadm,root,vagrant
```
Выберим метод PAM-аутентификации, так как у нас используется только ограничение по времени. Метод pam_time  нам не подходит, т.к. данный метод не работает с локальными групами пользователей. В нашей случае лучше написать скрипт контроля и использовать модуль pam_exec.

Создадим скрипт /usr/local/bin/login.sh.

**vi /usr/local/bin/login.sh**
```
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun"]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
```
В данном скрипте описаны все условия, которые нам нужны. Скрипт работает по принципу: 
Если сегодня суббота или воскресенье, то нужно проверить, входит ли пользователь в группу admin, если не входит — то подключение запрещено. При любых других вариантах подключение разрешено.

Добавим права на исполнение файла: **chmod +x /usr/local/bin/login.sh**

Укажем в файле **/etc/pam.d/sshd** модуль **pam_exec** и наш скрипт:

**vi /etc/pam.d/sshd**

```
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
account    required     pam_exec.so /usr/local/bin/login.sh
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```
Настройка завершена. Проверяем, что скрипт отрабатывает коректно.
```
ssh otus@192.168.57.10
otus@192.168.57.10's password: 
/usr/local/bin/login.sh failed: exit code 1
Connection closed by 192.168.57.10 port 22
```
