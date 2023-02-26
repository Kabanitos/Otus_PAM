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
