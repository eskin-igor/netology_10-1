# netology_10-1
# Домашнее задание к занятию «Disaster Recovery. FHRP и Keepalived»

## Задание 1

* Дана схема для Cisco Packet Tracer, рассматриваемая в лекции.
* На данной схеме уже настроено отслеживание интерфейсов маршрутизаторов Gi0/1 (для нулевой группы)
* Необходимо аналогично настроить отслеживание состояния интерфейсов Gi0/0 (для первой группы).
* Для проверки корректности настройки, разорвите один из кабелей между одним из маршрутизаторов и Switch0 и запустите ping между PC0 и Server0.
* На проверку отправьте получившуюся схему в формате pkt и скриншот, где виден процесс настройки маршрутизатора.

## Решение 1

Начальные настройки роутеров:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-0.PNG)
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-1.PNG)

Маршруты прохождения запросов до внесения правок.  
В направлении от PC  до Server:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-2.PNG)

В обратном направлении:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-3.PNG)

Влияния отказов на прохождение запросов от PC до Server.  
Тестирование отказа Router1 на интерфейсе Gi0/0, успешный ответ на запрос:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-4.PNG)

Тестирование отказа Router1 на интерфейсе Gi0/1, успешный ответ на запрос:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-5.PNG)

Тестирование отказа Router2 на интерфейсе Gi0/0, неуспешный ответ на запрос:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-6.PNG)

Тестирование отказа Router2 на интерфейсе Gi0/1, успешный ответ на запрос:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-7.PNG)

Причина такой работы в том, что отсутствует отслеживание состояния интерфейсов Gi0/0 со стороны Gi0/1.  
До настроим Router1 и Router2.
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-8.PNG)

Повторим ситуацию с ранее неуспешным ответом на запрос.  
Тестирование отказа Router2 на интерфейсе Gi0/0, теперь запрос отработан успешно:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-1-9.PNG)

[Файл с настройками сети.](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/hsrp_advanced_Eskin.pkt)

## Задание 2
* Запустите две виртуальные машины Linux, установите и настройте сервис Keepalived как в лекции, используя пример конфигурационного файла.
* Настройте любой веб-сервер (например, nginx или simple python server) на двух виртуальных машинах
* Напишите Bash-скрипт, который будет проверять доступность порта данного веб-сервера и существование файла index.html в root-директории данного веб-сервера.
* Настройте Keepalived так, чтобы он запускал данный скрипт каждые 3 секунды и переносил виртуальный IP на другой сервер, если bash-скрипт завершался с кодом, отличным от нуля (то есть порт веб-сервера был недоступен или отсутствовал index.html). Используйте для этого секцию vrrp_script
* На проверку отправьте получившейся bash-скрипт и конфигурационный файл keepalived, а также скриншот с демонстрацией переезда плавающего ip на другой сервер в случае недоступности порта или файла index.html

## Решение 2

Установлен nginx и keepalived по адресу 192.168.1.39.  
Данный хост будет MASTER:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-2-2.PNG)

Также установлен nginx и keepalived по адресу 192.168.1.116.  
Данный хост будет BACKUP:
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-2-1.PNG)

Bash-скрипт для проверки доступности порта веб-сервера nginx и наличие файла index.html в root-директории данного веб-сервера.  
[check_nginx.sh:](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/check_nginx.sh)  
```
#!/bin/bash  
if [[ $(netstat -ant | grep LISTEN | grep :80) ]] && [[ -f /var/www/html/index.nginx-debian.html ]]; then  
  exit 0  
else  
  sudo systemctl stop keepalived  
fi  
```
Настройки Keepalived для запуска скрипта каждые 3 секунды и переноса виртуального IP на другой сервер, если bash-скрипт завершится с кодом, отличным от нуля.  
[Для MASTER:](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/keepalived_for_MASTER.conf)  
```  
vrrp_script check_nginx {  
        script "/etc/keepalived/check_nginx.sh"  
	interval 3  
}  
vrrp_instance VI_1 {  
        state MASTER  
        interface enp0s3  
        virtual_router_id 15  
        priority 255  
        advert_int 1  
  
        virtual_ipaddress {  
              192.168.1.250/24  
        }  
        track_script {  
              check_nginx
        }
}
```
  
[Для BACKUP:](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/keepalived_for_BACKUP.conf)  
```  
vrrp_instance VI_1 {
        state BACKUP
        interface enp0s3
        virtual_router_id 15
        priority 254
        advert_int 1
        virtual_ipaddress {
              192.168.1.250/24
        }
}
```
  
### Демонстрация переезда плавающего ip на другой сервер в случае недоступности порта.

Подключимся на виртуальный ip-адрес 192.168.1.250.
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-2-3.PNG)

Убьём процесс на 80 порту сервера 192.168.1.39:  
sudo kill -9 $(sudo lsof –t -i:80)  
Обновим страницу с адресом 192.168.1.250.   
Как видим виртуальный ip переехал на хост 192.168.1.116  
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-2-6.PNG)

### Демонстрация переезда плавающего ip на другой сервер в случае недоступности файла index.html.

Подключимся на виртуальный ip-адрес 192.168.1.250.  
Удалим файл /var/www/html/index.nginx-debian.html на сервере 192.168.1.39.  
Виртуальный ip переехал на хост 192.168.1.116.  
![](https://github.com/eskin-igor/netology_10-1/blob/main/10-1/10-1-2-6.PNG)


