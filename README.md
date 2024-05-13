### OTUS Linux Professional Lesson #15 | Subject: SELinux
#### ЦЕЛЬ ДЗ: Диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется
#### ОПИСАНИЕ ДЗ:
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
К сдаче:
- README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux:
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems; 
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность

#### ОПИСАНИЕ ВЫПОЛНЕНИЯ ДЗ:
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
Видим что nginx не запустился по причине "Permission denied", потому что он пытается запуститься на порту 4881, но Selinux это запрущает

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
