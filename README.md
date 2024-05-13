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

СПОСОБ 1. Переключатели setsebool

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
СПОСОБ 2. Добавление нестандартного порта в имеющийся тип
