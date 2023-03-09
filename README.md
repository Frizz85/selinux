# Практика с SELinux. SELinux - когда все запрещено

## Инструкция по выполнению домашнего задания:


1. Запустить nginx на нестандартном порту 3-мя разными способами:

    * переключатели setsebool;
    * добавление нестандартного порта в имеющийся тип;
    * формирование и установка модуля SELinux.
    К сдаче:
    * README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.

    * развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
    * выяснить причину неработоспособности механизма обновления зоны (см. README);
    * предложить решение (или решения) для данной проблемы;
    * выбрать одно из решений для реализации, предварительно обосновав выбор;
    * реализовать выбранное решение и продемонстрировать его работоспособность.
   
## 0. Создание виртуальной машины

```
vagrant up
```

Ошибка запуска nginx при установке:

![files](img/1.JPG)

Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.

## 1. Запуск nginx на нестандартном порту 3-мя разными способами

```
vagrant ssh
sudo -i
```

Проверка, что в ОС отключен файервол: 
```
systemctl status firewalld
```

![files](img/2.JPG)

Проверка, что конфигурация nginx настроена без ошибок: 
```
nginx -t
```

![files](img/3.JPG)

Провека режима работы SELinux: 
```
getenforce
```

![files](img/4.JPG)

### Разрешение в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

Нахождение в логах (/var/log/audit/audit.log) информацию о блокировании порта:

```
grep /var/log/audit/audit.log -e '4881'
```

![files](img/5.JPG)

Установка audit2why

```
yum -q provides audit2why
```

![files](img/6.JPG)


```
yum install policycoreutils-python
```

Информация о запрете:
```
grep /var/log/audit/audit.log -e '4881' | audit2why
```

![files](img/7.JPG)

Изменение параметра:

```
setsebool -P nis_enabled 1
systemctl restart nginx
systemctl status nginx
```

![files](img/8.JPG)

Проверка работопособности nginx в браузере:

![files](img/9.JPG)

Проверка статуса параметра можно с помощью команды:

```
getsebool -a | grep nis_enabled
```

![files](img/10.JPG)

Возврат запрета работы nginx на порту 4881 обратно:
```
setsebool -P nis_enabled off
```

### Разрешение в SELinux работы nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип

Поиск имеющегося типа, для http трафика: 
```
semanage port -l | grep http
```
![files](img/11.JPG)

Добавление порта в тип http_port_t: 
```
semanage port -a -t http_port_t -p tcp 4881
semanage port -l | grep http_port_t
```
![files](img/12.JPG)

Проверка работы nginx:

![files](img/13.JPG)

Удаление нестандартного порта из имеющегося типа можно с помощью команды: 
```
semanage port -d -t http_port_t -p tcp 4881
```

### Разрешение в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
Попробуем снова запустить nginx:

```
systemctl start nginx
```
![files](img/15.JPG)

Nginx не запуститься, так как SELinux продолжает его блокировать. Посмотрим логи SELinux, которые относятся к nginx:

```
grep nginx /var/log/audit/audit.log
```
![files](img/16.JPG)


Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 

```
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
```
![files](img/17.JPG)

Выполнение команды предложенной утилитой audit2allow:
```
semodule -i nginx.pp
```

Запуск и проверка запуска nginx:
```
systemctl start nginx
systemctl status nginx
```

![files](img/18.JPG)

Просмотр всех установленных модулей: 
```
semodule -l
```

Удаление модуля: 
```
semodule -r nginx
```

## 2. Обеспечение работоспособности приложения при включенном SELinux

Клонирование репозитория: 
```
git clone https://github.com/mbfx/otus-linux-adm.git
```

Сздание окружения
```
cd otus-linux-adm/selinux_dns_problems
vagrant up
```

Проверка ВМ с помощью команды: 
```
vagrant status
```

![files](img/19.JPG)

Подключимся к клиенту: 
```
vagrant ssh client
```
Попробуем внести изменения в зону: 
```
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.
Подключение к серверу ns01 и проверка логов SELinux:

![files](img/20.JPG)

В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.

Для этого воспользуемся утилитой audit2why: cat /var/log/audit/audit.log | audit2why
![files](img/21.JPG)

Проверим данную проблему в каталоге /etc/named:
```
ls -laZ /etc/named
```
![files](img/22.JPG)

Изменим тип контекста безопасности для каталога /etc/named: 
```
sudo semanage fcontext -l | grep named
sudo chcon -R -t named_zone_t /etc/named
```

Попробуем снова внести изменения с клиента: 
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
dig www.ddns.lab
```

![files](img/24.JPG)

Перегружаем окружения и выполняем проверку:

```
dig @192.168.50.10 www.ddns.lab
```

![files](img/26.JPG)
