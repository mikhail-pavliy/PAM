# PAM
# 1. Запретить всем пользователям, кроме группы admin логин в выходные(суббота и воскресенье), без учета праздников
запретим конкретному пользователю, пока без привязки к группе.
Чтобы у пользователя не было возможности войти в систему не только через SSH, но и локально для этого необходимо откорректировать два файла.
nano /etc/pam.d/sshd  и nano /etc/pam.d/login
```ruby
[vagrant@PAM ~]$ cat /etc/pam.d/sshd
#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_access.so 
account    required     pam_time.so
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
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
```ruby
[vagrant@PAM ~]$ cat /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       substack     system-auth
account    required     pam_time.so
account    required     pam_access.so
auth       include      postlogin
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
```
Далее заходим в файл /etc/security/time.conf и добавляем в конце файла строку, запретив вход по выходным *;*;test_user;Wk
Добавляем пользователя командой useradd test_user и задаем ему пароль, passwd test_user
Заходим сегодня, в четверг, когда у нас работает правило. 
```ruby
[vagrant@PAM ~]$ ssh test_user@localhost
test_user@localhost's password:
[test_user@PAM ~]$
```
Как видно все работает, поменяем немного правило изменив Wk на Thu
```ruby
[vagrant@PAM ~]$ date
Thu Jan 14 11:55:07 UTC 2021
```
```ruby
[vagrant@PAM ~]$ ssh test_user@localhost
test_user@localhost's password:
Authentication failed.
```
# Создадим ограничения на группу admin 
  1. Создаем группу groupadd admin
  2. Добавим нашего пользователя test_user в ранее созданную группу usermod -aG admin test_user
  3. Установим модуль pam_script
  ```ruby
  [root@PAM vagrant]# cat /etc/pam.d/sshd
#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_access.so
auth       required     pam_script.so # Добавили эту строку
account    required     pam_time.so
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
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
Правим  файл /etc/pam_script
``` ruby
#!/bin/bash

if [[ `grep $PAM_USER /etc/group | grep 'admin'` ]]
then
exit 0
fi
if [[ `date +%u` > 5 ]]
then
exit 1
fi
```
