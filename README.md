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
