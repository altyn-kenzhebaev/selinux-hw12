# Selinux
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/selinux-hw12.git`
В текущей директории появится папка с именем репозитория. В данном случае selinux-hw12. Ознакомимся с содержимым:
```
cd selinux-hw12
ls -l
otus-linux-adm
README.md
Vagrantfile
```
Здесь:
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
- otus-linux-adm - папка для работы над 2-м заданием
Запускаем ВМ:
```
vagrant up
```
Во время запуска nginx не запустился и вывел следующую ошибку:
```
    selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Wed 2023-05-24 09:03:47 UTC; 11ms ago
    selinux:   Process: 2506 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2505 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux: 
    selinux: May 24 09:03:47 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: May 24 09:03:47 selinux nginx[2506]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: May 24 09:03:47 selinux nginx[2506]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: May 24 09:03:47 selinux nginx[2506]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: May 24 09:03:47 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: May 24 09:03:47 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: May 24 09:03:47 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: May 24 09:03:47 selinux systemd[1]: nginx.service failed
```
## 1-ое задание: Запуск nginx на нестандартном порту 3-мя разными способами
### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
```
[root@selinux ~]# grep denied /var/log/audit/audit.log 
type=AVC msg=audit(1684919027.605:684): avc:  denied  { name_bind } for  pid=2506 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
С помощью утилиты audit2why выясняем причину блокировки:
```
[root@selinux ~]# yum install policycoreutils-python
[root@selinux ~]# grep denied /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1684919027.605:684): avc:  denied  { name_bind } for  pid=2506 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled:
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-05-24 09:18:12 UTC; 5s ago
  Process: 2641 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2638 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2637 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2643 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2643 nginx: master process /usr/sbin/nginx
           └─2645 nginx: worker process

May 24 09:18:12 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 24 09:18:12 selinux nginx[2638]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 24 09:18:12 selinux nginx[2638]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 24 09:18:12 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
    state: reloaded
```
Проверяем доступность порта утилитой nc:
```
[root@selinux ~]# yum install netcat -y
[root@selinux ~]# nc -zv 127.0.0.1 4881
Connection to 127.0.0.1 4881 port [tcp/*] succeeded!
```
Отключаем nis_enabled для проработки следующих заданий:
```
setsebool -P nis_enabled off
```
### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип
Поиск типа порта для http трафика::
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t::
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
```
Проверяем:
```
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-05-24 09:25:14 UTC; 1min 37s ago
  Process: 2695 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2692 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2691 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2697 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2697 nginx: master process /usr/sbin/nginx
           └─2700 nginx: worker process

May 24 09:25:14 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
May 24 09:25:14 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 24 09:25:14 selinux nginx[2692]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 24 09:25:14 selinux nginx[2692]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 24 09:25:14 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Удаляем нестандартный порт из имеющегося типа для проработки следующего задания:
```
semanage port -d -t http_port_t -p tcp 4881
```
### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
Попытка снова запустить nginx:
```
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# nc -zv 127.0.0.1 4881
nc: connect to 127.0.0.1 port 4881 (tcp) failed: Connection refused
```
Смотрим логи:
```
[root@selinux ~]# grep denied /var/log/audit/audit.log
type=AVC msg=audit(1684919027.605:684): avc:  denied  { name_bind } for  pid=2506 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=AVC msg=audit(1684920586.285:755): avc:  denied  { name_bind } for  pid=2724 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 
```
[root@selinux ~]# grep denied /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
[root@selinux ~]# semodule -i nginx.pp
```
Попробуем снова запустить nginx и проверим доступность порта:
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-05-24 09:36:45 UTC; 4s ago
  Process: 2755 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2753 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2752 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2757 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2757 nginx: master process /usr/sbin/nginx
           └─2759 nginx: worker process

May 24 09:36:45 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 24 09:36:45 selinux nginx[2753]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 24 09:36:45 selinux nginx[2753]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 24 09:36:45 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@selinux ~]# nc -zv 127.0.0.1 4881
Connection to 127.0.0.1 4881 port [tcp/*] succeeded!
```
## 2-ое задание: Обеспечение работоспособности приложения при включенном SELinux
Для этого задание нужно перейти в папку selinux_dns_problems и запустить стенд:
```
cd otus-linux-adm/selinux_dns_problems
vagrant up
```
В результате получаем 2 ВМ:
```
$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```
Попробуем внести изменения в зону с client:
```
[vagrant@client ~]$  nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
Изменения внести не получилось. Давайте посмотрим логи SELinux на клиенте client и на сервере ns01, чтобы понять в чём может быть проблема
На клиенте:
```
[root@client ~]# grep denied /var/log/audit/audit.log | audit2why
Nothing to do
```
На сервере:
```
[root@ns01 ~]# grep denied /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1684921523.109:1931): avc:  denied  { create } for  pid=5205 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1684921559.063:1932): avc:  denied  { create } for  pid=5205 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
Проверим данную проблему в каталоге /etc/named:
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Изменим тип контекста безопасности для каталога /etc/named, и проверим:
```
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Попробуем снова внести изменения с клиента: 
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58857
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed May 24 09:56:35 UTC 2023
;; MSG SIZE  rcvd: 96

```
Как мы видим изменения внести удалось