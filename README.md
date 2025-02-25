# Домашнее задание Vagrant-стенд c DNS

## Цель домашнего задания  
Создать домашнюю сетевую лабораторию. Изучить основы DNS, научиться работать с технологией Split-DNS в Linux-based системах

### Описание домашнего задания  

Что нужно сделать?

1. взять стенд https://github.com/erlong15/vagrant-bind    
- добавить еще один сервер client2    
- авести в зоне dns.lab имена:    
  web1 - смотрит на клиент1    
  web2  смотрит на клиент2     
- завести еще одну зону newdns.lab    
- завести в ней запись    
- www - смотрит на обоих клиентов    

2. настроить split-dns    
- клиент1 - видит обе зоны, но в зоне dns.lab только web1    
- клиент2 видит только dns.lab    

Дополнительное задание    
* настроить все без выключения selinux     

Формат сдачи ДЗ - vagrant + ansible    


## Введение    
DNS(Domain Name System, Служба доменных имён) -  это распределенная система, для получения информации о доменах. DNS используется для сопоставления IP-адресов и доменных имён.    
Сопостовления IP-адресов и DNS-имён бывают двух видов:     
Прямое (DNS-bмя в IP-адрес)    
Обратное (IP-адрес в DNS-имя)    

Доменная структура DNS представляет собой древовидную иерархию, состоящую из узлов, зон, доменов, поддоменов и т.д. «Вершиной» доменной структуры является корневая зона. Корневая (root) зона обозначается точкой. Далее следуют домены первого уровня (.com, ,ru, .org и т. д.) и т д.    

В DNS встречаются понятия зон и доменов:    
Зона — это любая часть дерева системы доменных имён, размещаемая как единое целое на некотором DNS-сервере.      
Домен – определенный узел, включающий в себя все подчинённые узлы.      

Давайте разберем основное отличие зоны от домена. Возьмём для примера ресурс otus.ru — это может быть сразу и зона и домен, однако, при использовании зоны otus.ru мы можем сделать отдельную зону mail.otus.ru, которая будет управляться не нами. В случае домена так сделать нельзя...     

FQDN (Fully Qualified Domain Name) - полностью указанное доменное имя, т.е. от корневого домена. Ключевой идентификатор FQDN - точка в конце имени. Максимальный размер FQDN — 255 байт, с ограничением в 63 байта на каждое имя домена. Пример FQDN: mail.otus.ru.     

Вся информация о DNS-ресурсах хранится в ресурсных записях. Записи хранят следующие атрибуты:    
Имя (NAME) - доменное имя, к которому привязана или которому принадлежит данная ресурсная область, либо IP-адрес. При отсутствии данного поля, запись ресурса наследуется от предыдущей записи.      
TTL (время жизни в кэше) - после указанного времени запись удаляется, данное поле может не указываться в индивидуальных записях ресурсов, но тогда оно должно быть указано в начале файла зоны и будет наследоваться всеми записями.    
Класс (CLASS) - определяет тип сети (в 99% используется IN - интернет)    
Тип (TYPE) - тип записи, синтаксис и назначение записи     
Значение (DATA)      

Типы рекурсивных записей:    
А (Address record) - отображают имя хоста (доменное имя) на адрес IPv4     
AAAA - отображает доменное имя на адрес IPv6     
CNAME (Canonical name record/псевдоним) - привязка алиаса к существующему доменному имени
MX (mail exchange) - указывает хосты для отправки почты, адресованной домену. При этом поле NAME указывает домен назначения, а поле DATA приоритет и доменное имя хоста, ответственного за приём почты. Данные вводятся через пробел    
NS (name server) -  указывает на DNS-сервер, обслуживающий данный домен.      
PTR (pointer) -  Отображает IP-адрес в доменное имя     
SOA (Start of Authority/начальная запись зоны) - описывает основные начальные настройки зоны.      
SRV (server selection) — указывает на сервера, обеспечивающие работу тех или иных служб в данном домене (например  Jabber и Active Directory).    

Для работы с DNS (как клиенту) в linux используют утилиты dig, host и nslookup    
Также в Linux есть следующие реализации DNS-серверов:    
bind    
powerdns (умеет хранить зоны в БД)    
unbound (реализация bind)    
dnsmasq     
и т д.     

Split DNS (split-horizon или split-brain) — это конфигурация, позволяющая отдавать разные записи зон DNS в зависимости от подсети источника запроса. Данную функцию можно реализовать как с помощью одного DNS-сервера, так и с помощью нескольких DNS-серверов…    
       
# Выполнение:  

### 1. С помощью vagrant развернул тестовый стенд из четырех виртуальных машин:
|Имя|IP-адрес|
|-|-|
|ns01|192.168.57.10|
|ns02|192.168.57.11|
|Client1|192.168.57.15|
|Client2|192.168.57.16|    

### 2. Создал и запустил playbook dns.yml    

```shell
root@ansible:/home/vagrant/ansible# ansible-playbook dns.yml

PLAY [Configure bind dns] *************************************************************************************

TASK [Gathering Facts] ****************************************************************************************
ok: [ns01]
ok: [client1]
ok: [client2]
ok: [ns02]
...
PLAY RECAP ****************************************************************************************************
client1                    : ok=6    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
client2                    : ok=6    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
ns01                       : ok=9    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
ns02                       : ok=8    changed=6    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```    
Playbook отработал без ошибок    

### 3. Проверка зоны dns.lab    
На клиентах запустил проверку зоны командой "dig"     

*client1*    
```shell
[vagrant@client1 ~]$ dig web1.dns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web1.dns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51634
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.57.15

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns02.dns.lab.
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 10:31:09 UTC 2025
;; MSG SIZE  rcvd: 127
```    
*client2*    
```shell
[vagrant@client2 ~]$ dig web2.dns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web2.dns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4509
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.                  IN      A

;; ANSWER SECTION:
web2.dns.lab.           3600    IN      A       192.168.57.16

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns01.dns.lab.
dns.lab.                3600    IN      NS      ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 1 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 10:38:19 UTC 2025
;; MSG SIZE  rcvd: 127
```
Из вывода команды "dig" вижу что в секции ANSWER SECTION от обоих машин есть ответ    
  

### 4. Проверка зоны newdns.lab    
Выполняю проверку работы зоны с client1    
*client1*    
```shell
[vagrant@client1 ~]$ dig www.newdns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.newdns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23006
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.newdns.lab.                        IN      A

;; ANSWER SECTION:
www.newdns.lab.         3600    IN      A       192.168.57.15
www.newdns.lab.         3600    IN      A       192.168.57.16

;; AUTHORITY SECTION:
newdns.lab.             3600    IN      NS      ns02.dns.lab.
newdns.lab.             3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 12:31:52 UTC 2025
;; MSG SIZE  rcvd: 149
```    
Из вывода команды "dig" в секции ANSWER SECTION вижу ответ от обоих машин    

### 5. Настройка Split-DNS    
Цель:     
client1 видит обе зоны (dns.lab и newdns.lab), но в dns.lab только web1.     
client2 видит только зону dns.lab    
Для настройки запустил автоматизацию ansible-playbook split-dns.yml    

```shell
root@ansible:/home/vagrant/ansible# ansible-playbook split-dns.yml

PLAY [Configure Split-DNS] ************************************************************************************

TASK [Gathering Facts] ****************************************************************************************
ok: [ns01]
ok: [client1]
ok: [client2]
ok: [ns02]

TASK [Copy bind config for split-dns] *************************************************************************
skipping: [client1]
skipping: [client2]
changed: [ns01]
changed: [ns02]

TASK [Copy dns.lab.client1 zone config] ***********************************************************************
skipping: [ns02]
skipping: [client1]
skipping: [client2]
changed: [ns01]

TASK [Restart named service] **********************************************************************************
changed: [client1]
changed: [client2]
changed: [ns01]
changed: [ns02]

PLAY RECAP ****************************************************************************************************
client1                    : ok=2    changed=1    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
client2                    : ok=2    changed=1    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
ns01                       : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
ns02                       : ok=3    changed=2    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```    
Playbook отработал без ошибок    

*client1*    
```shell
[vagrant@client1 ~]$ dig web1.dns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web1.dns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21240
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.57.15   #<---------ответ

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns01.dns.lab.
dns.lab.                3600    IN      NS      ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 3 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 12:49:31 UTC 2025
;; MSG SIZE  rcvd: 127
------------------------------------------------------------------------------------------------------------
[vagrant@client1 ~]$ dig web2.dns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web2.dns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 17003  #<--------------------ошибка
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.                  IN      A

;; AUTHORITY SECTION:
dns.lab.                600     IN      SOA     ns01.dns.lab. root.dns.lab. 2507202401 3600 600 86400 600

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 12:51:07 UTC 2025
;; MSG SIZE  rcvd: 87
-------------------------------------------------------------------------------------------------------------
[vagrant@client1 ~]$ dig www.newdns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.newdns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57827
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.newdns.lab.                        IN      A

;; ANSWER SECTION:
www.newdns.lab.         3600    IN      A       192.168.57.16    #<---------ответ
www.newdns.lab.         3600    IN      A       192.168.57.15    #<---------ответ

;; AUTHORITY SECTION:
newdns.lab.             3600    IN      NS      ns02.dns.lab.
newdns.lab.             3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 12:53:05 UTC 2025
;; MSG SIZE  rcvd: 149
```    

*client2*
```shell
[vagrant@client2 ~]$ dig web1.dns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web1.dns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31328
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.57.15     #<---------ответ

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns02.dns.lab.
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 12:55:29 UTC 2025
;; MSG SIZE  rcvd: 127
--------------------------------------------------------------------------------------
[vagrant@client2 ~]$ dig web2.dns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web2.dns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38445
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.                  IN      A

;; ANSWER SECTION:
web2.dns.lab.           3600    IN      A       192.168.57.16     #<---------ответ

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns02.dns.lab.
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 12:56:31 UTC 2025
;; MSG SIZE  rcvd: 127
-------------------------------------------------------------------------------------
[vagrant@client2 ~]$ dig www.newdns.lab @192.168.57.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.newdns.lab @192.168.57.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 54359                #<--------------------ошибка
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.newdns.lab.                        IN      A

;; AUTHORITY SECTION:
.                       10724   IN      SOA     a.root-servers.net. nstld.verisign-grs.com. 2025022500 1800 900
 604800 86400

;; Query time: 1 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 13:01:32 UTC 2025
;; MSG SIZE  rcvd: 118
```    

Итог:    
1. Зона dns.lab настроена с записями web1 и web2.    
2. Зона newdns.lab создана с записью www, указывающей на оба клиента.      
3. Split-DNS работает:      
 - client1 видит web1.dns.lab и www.newdns.lab.     
 - client2 видит только всю зону dns.lab (web1 и web2).      
Все проверки успешно пройдены    

### 6. Настройка BIND с включенным SELinux    

На всех машинах включил SELinux

```shell
sudo setenforce 1
```    

#### 6.1. Настройка контекстов SELinux для BIND    
Проверил контексты файлов зон    
```shell
[root@ns01 ~]# ls -Z /etc/named/named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       /etc/named/named.dns.lab

[root@ns01 ~]# ls -Z /etc/named/named.newdns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       /etc/named/named.newdns.lab
```     

#### 6.2. Исправил контексты
```shell
[root@ns01 ~]# chcon -t named_zone_t /etc/named/named.newdns.lab
[root@ns01 ~]# chcon -t named_zone_t /etc/named/named.dns.lab
[root@ns01 ~]# chcon -t named_conf_t /etc/named.conf
[root@ns01 ~]# restorecon -Rv /etc/named
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
```    

Перезапустил службу named   
```shell
[root@ns01 ~]# systemctl restart named
[root@ns01 ~]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2025-02-25 20:29:18 UTC; 10s ago
  Process: 24263 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS
)
  Process: 24277 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 24275 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF
"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 24279 (named)
   CGroup: /system.slice/named.service
           └─24279 /usr/sbin/named -u named -c /etc/named.conf

Feb 25 20:29:18 ns01 named[24279]: running
Feb 25 20:29:18 ns01 systemd[1]: Started Berkeley Internet Name Domain (DNS).
Feb 25 20:29:18 ns01 named[24279]: zone dns.lab/IN/client1: sending notifies (serial 2507202401)
Feb 25 20:29:18 ns01 named[24279]: zone newdns.lab/IN/client1: sending notifies (serial 2507202401)
Feb 25 20:29:18 ns01 named[24279]: zone dns.lab/IN/client2: sending notifies (serial 2507202401)
Feb 25 20:29:18 ns01 named[24279]: zone newdns.lab/IN/default: sending notifies (serial 2507202401)
Feb 25 20:29:18 ns01 named[24279]: zone 57.168.192.in-addr.arpa/IN/default: sending notifies (serial 2507202401)
Feb 25 20:29:18 ns01 named[24279]: zone dns.lab/IN/default: sending notifies (serial 2507202401)
Feb 25 20:29:28 ns01 named[24279]: resolver priming query complete
Feb 25 20:29:28 ns01 named[24279]: managed-keys-zone/default: Unable to fetch DNSKEY set '.': timed out
```    

#### 6.3. Проверил    

*client1*   
```shell
[root@client1 ~]# dig web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web1.dns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42504
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.57.15       #<---------ответ

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns02.dns.lab.
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 20:29:57 UTC 2025
;; MSG SIZE  rcvd: 127
------------------------------------------------------------------------
[root@client1 ~]# dig www.newdns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.newdns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30292
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.newdns.lab.                        IN      A

;; ANSWER SECTION:
www.newdns.lab.         3600    IN      A       192.168.57.16        #<---------ответ
www.newdns.lab.         3600    IN      A       192.168.57.15        #<---------ответ

;; AUTHORITY SECTION:
newdns.lab.             3600    IN      NS      ns01.dns.lab.
newdns.lab.             3600    IN      NS      ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 20:31:54 UTC 2025
;; MSG SIZE  rcvd: 149
```    
*client2*   
```shell
[root@client2 ~]# dig web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> web2.dns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37730
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.                  IN      A

;; ANSWER SECTION:
web2.dns.lab.           3600    IN      A       192.168.57.16         #<---------ответ

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns02.dns.lab.
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.57.10
ns02.dns.lab.           3600    IN      A       192.168.57.11

;; Query time: 0 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 20:33:15 UTC 2025
;; MSG SIZE  rcvd: 127
--------------------------------------------------------------------
[root@client2 ~]# dig www.newdns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.newdns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 41513            #<--------------------ошибка
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.newdns.lab.                        IN      A

;; AUTHORITY SECTION:
.                       10800   IN      SOA     a.root-servers.net. nstld.verisign-grs.com. 2025022501 1800 900 604800 86400

;; Query time: 933 msec
;; SERVER: 192.168.57.10#53(192.168.57.10)
;; WHEN: Tue Feb 25 20:33:02 UTC 2025
;; MSG SIZE  rcvd: 118
```   

__________________   

end
