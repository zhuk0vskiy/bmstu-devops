# Л/Р №1

Развернуть локально стенд из нескольких виртуальных машин (ВМ). Описание стенда:
1.  ВМ1 с двумя сетевыми интерфейсами.
    1-й сетевой интерфейс подключен к сети интернет, так же как и хостовая ОС
    2-й сетевой интерфейс подключен к локальной сети внутри гипервизора
2.  ВМ2 с одним сетевым интерфейсом. 
    интерфейс подключен к локальной сети внутри гипервизора

На виртуальных машинах должен быть установлен любой из дистрибутивов Linux. (рекомендуется Ubuntu22-24 server)


## Задание 1. 

Требуется настроить ВМ1 так, чтобы она выполняла функцию пограничного маршрутизатора,
 где WAN - это интерфейс 1, а LAN - интерфейс 2.
Шлюз по умолчанию для ВМ2 это IP интерфейса 2 от ВМ1.
После успешной настройки с ВМ2 должен быть доступ в интернет.


## Задание 2. 

На ВМ2 развернуть APACHE на порту 80.
На ВМ1 развернуть NGINX на порту 80.
Требуется настроить виртуальные машины так, чтобы при обращении с хостовой (где хостятся ВМ) машины на IP адрес 
ВМ1 1-го сетевого интерфейса должна выводиться дефолтная страница Apache, развернутого на ВМ2. 
nginx как front-end к apache!

## Задание 3.
Настроить проброс портов. portforwarding.
При обращении на порт 2222 (или любой другой порт на ваш выбор)  wan интерфейса вм1 запрос должен  улететь на 22 порт вм2 и мы должны получить доступ к ssh vm2.

Реализовать используя iptables.

# Решения

## Задание 1.

Качаем VirtualBox и iso образ ubuntu server (обязательно смотрите на архитектуру процессора, у кого мак на arm - нужно соответстенно arm версию)

Создаем две виртуалки. На одной делаем сетевые адаптеры *сетевой мост* и *внутренняя сеть*, на второй *внутренняя сеть*

За ходим на первую ВМ.
Вводим `ip a` и убеждаемся, что на интерфейсе адаптера *сетевой мост* есть айпишник (будет начинаться на 192.168.1.*).
Назначаем айпишник для интерфейса внутренней сети
`sudo ip addr add <ip-адрес>/<маска сети> dev <интерфейс внутренней сети>`
Теперь заходим под root
`sudo echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf` и включаем ip форвардинг 
`sudo sysctl -p`
и настраиваем NAT
`sudo iptables -t nat -A POSTROUTING -o <интерфейс сетевого моста> -j MASQUERADE`
Сохраняем правила iptables
`sudo apt-get install iptables-persistent`
`sudo netfilter-persistent save`

Заходим на вторую ВМ
Назначаем айпишник для интерфейса внутренней сети
`sudo ip addr add <ip-адрес>/<маска сети> dev <интерфейс внутренней сети>` и gateway `sudo ip route add default via <ip-адрес первом ВМ>`. 
Добавляем днс, чтоб apt смог скачивать пакеты `echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf`

Проверяем что все работает `ping 8.8.8.8` и проверяем apt `sudo apt update`

## Задание 2.

Заходим на первую ВМ
Скачиваем nginx `sudo apt install nginx`.
Меняем в */etc/nginx/sites-available/default* в **location / {}** `try_files $uri $uri/ =404;` на `proxy_pass http://<ip-адрес второй ВМ>;` для переадресации.
Перезапускаем nginx `sudo systemctl restart nginx`

Заходим на вторую ВМ
Скачиваем apache `sudo apt install apache2`

На хостовой такче в браузере заходим по адресу первой ВМ (тот что по сетевому мосту выдается) и там должна быть страница apache

## Задание 3.

Заходим на первую ВМ
Перенаправляем входящие подключения
`sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination <ip-адрес второй ВМ>.:22`.
Разрешаем форвардинг пакетов
`sudo iptables -A FORWARD -p tcp -d <ip-адрес второй ВМ> --dport 22 -j ACCEPT`

Заходим на вторую ВМ
Скачиваем ssh сервер
`sudo apt install ssh`
и проверяем что он работает
`sudo systemctl status ssh`

С хостовой ОС подлюкаемся по ssh к первой ВМ по сетевому мостц
`ssh <username>@<ip-адрес> -p 2222`

