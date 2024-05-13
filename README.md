### OTUS Linux Professional Lesson #18 | Subject: SELinux
#### ЦЕЛЬ ДЗ: Диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется
#### ОПИСАНИЕ ДЗ:
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

2. Обеспечить работоспособность приложения при включенном selinux:
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems; 
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность

#### ОПИСАНИЕ ВЫПОЛНЕНИЯ ДЗ

#### ЗАДАНИЕ 1. Запустить nginx на нестандартном порту 3-мя разными способами
Разворачиваем виртуальную машину из Vagrantfile с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.
Смотрим статус nginx:
```
[vagrant@selinux ~]$ systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Mon 2024-05-13 05:32:48 UTC; 6h ago
  Process: 4667 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 4666 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
```
```
[root@selinux vagrant]# journalctl _SYSTEMD_UNIT=nginx.service
-- Logs begin at Mon 2024-05-13 05:31:56 UTC, end at Mon 2024-05-13 12:24:11 UTC. --
May 13 05:32:48 selinux nginx[4667]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 13 05:32:48 selinux nginx[4667]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
May 13 05:32:48 selinux nginx[4667]: nginx: configuration file /etc/nginx/nginx.conf test failed
```
Видим что nginx не запустился по причине "Permission denied", потому что он пытается запуститься на порту 4881, но Selinux это запрещает

Решить это можно несколькими способами.

__СПОСОБ 1. Переключатели setsebool__

Находим в /var/log/audit.log информацию о блокировании порта:
```
type=AVC msg=audit(1715578368.673:1650): avc:  denied  { name_bind } for  pid=4667 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
Отправляем эту строчку утилите audit2why:
```
[root@selinux vagrant]# grep 1715578368.673:1650 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1715578368.673:1650): avc:  denied  { name_bind } for  pid=4667 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
Утилита audit2why показывает почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled. 
Включим параметр nis_enabled и перезапустим nginx:
```
[root@selinux vagrant]# setsebool -P nis_enabled on
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-05-13 12:48:19 UTC; 8s ago
  Process: 25771 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 25769 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 25768 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 25773 (nginx)
   CGroup: /system.slice/nginx.service
           ├─25773 nginx: master process /usr/sbin/nginx
           └─25775 nginx: worker process

May 13 12:48:19 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 13 12:48:19 selinux nginx[25769]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 13 12:48:19 selinux nginx[25769]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 13 12:48:19 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Видим что nginx работает.

Проверить статусы параметров:
```
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Отключить параметр:
```
[root@selinux vagrant]# setsebool -P nis_enabled off
```

__СПОСОБ 2. Добавление нестандартного порта в имеющийся тип__

Ищем имеющийся тип для http траффика:
```
[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t:
```
[root@selinux vagrant]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux vagrant]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
Проверяем работу nginx:
```
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-05-13 12:55:03 UTC; 4s ago
  Process: 25818 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 25816 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 25815 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 25820 (nginx)
   CGroup: /system.slice/nginx.service
           ├─25820 nginx: master process /usr/sbin/nginx
           └─25822 nginx: worker process

May 13 12:55:03 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 13 12:55:03 selinux nginx[25816]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 13 12:55:03 selinux nginx[25816]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 13 12:55:03 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Видим что nginx работает.

Удалить порт из типа:
```
[root@selinux vagrant]# semanage port -d -t http_port_t -p tcp 4881
```

__СПОСОБ 3. Формирование и установка модуля SELinux__

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:
```
root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль:
```
[root@selinux vagrant]# semodule -i nginx.pp
```
Проверяем работу nginx:
```
[root@selinux vagrant]# systemctl start nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-05-13 13:00:15 UTC; 13s ago
  Process: 25865 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 25863 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 25862 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 25867 (nginx)
   CGroup: /system.slice/nginx.service
           ├─25867 nginx: master process /usr/sbin/nginx
           └─25869 nginx: worker process

May 13 13:00:15 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 13 13:00:15 selinux nginx[25863]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 13 13:00:15 selinux nginx[25863]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 13 13:00:15 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Видим что nginx работает.

После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.

Удалить модуль:
```
[root@selinux vagrant]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```
Посмотреть установленные модули:
```
[root@selinux vagrant]# semodule -l
abrt	1.4.1
accountsd	1.1.0
acct	1.6.0
afs	1.9.0
aiccu	1.1.0
aide	1.7.1
...
```

#### ЗАДАНИЕ 2. Обеспечить работоспособность приложения при включенном selinux.

Выполним клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git

Перейдём в каталог со стендом: 
```
cd selinux_dns_problems
```
Развернём 2 ВМ с помощью vagrant: 
```
vagrant up
```
Подключимся к клиенту и попробуй внести изменения в зону dns:
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab   
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
Как видим, изменения внести не получилось.

Смотрим логи SELinux на клиенте:
```
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```
Пусто.

Смотрим логи SELinux на сервере:
```
[root@ns01 vagrant]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1715608488.544:2949): avc:  denied  { create } for  pid=6970 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
Ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.

Посмотрим в каталог /etc/named:
```
[root@ns01 vagrant]# ll -aZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
> [!IMPORTANT]
> Проблема заключается в том, что конфигурационные файлы зон лежат не в /var/named/, а в /etc/named. Для /etc/named/ установлен контекст etc_t. А политикой SELinux заперещены действия сервиса named с контекстом named_t над каталогом с контекстом etc_t. Решение - изменить контекст каталога /etc/named на корректный.

Смотрим какой контекст у каталога с зонами по умолчанию:
```
[root@ns01 vagrant]# semanage fcontext -l | grep named
... 
/var/named(/.*)?                                   all files          system_u:object_r:__named_zone_t__:s0 
...
```
Меняем контекс каталога /etc/named на named_zone_t:
```
[root@ns01 vagrant]# chcon -R -t named_zone_t /etc/named
[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Попробуем снова внести изменения с клиента. Видим, что все работает:
![Screenshot from 2024-05-13 19-36-26](https://github.com/bonyakevich-e/otus_lp_lesson_18_selinux/assets/114911797/c9f6edf1-78db-4808-9eb1-c48c0e531e7b)
![Screenshot from 2024-05-13 19-37-49](https://github.com/bonyakevich-e/otus_lp_lesson_18_selinux/assets/114911797/c38bb1d6-5770-4d7d-8db1-efde7336f7df)
![Screenshot from 2024-05-13 19-44-59](https://github.com/bonyakevich-e/otus_lp_lesson_18_selinux/assets/114911797/a51aa9ca-d3a2-4b4c-a467-ba1834b75af0)


Для того, чтобы вернуть правила обратно, можно ввести команду:
```
[root@ns01 vagrant]# restorecon -v -R /etc/named
```


