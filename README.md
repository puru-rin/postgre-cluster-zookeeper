# postgre-cluster-zookeeper
Создание кластера PostgreSQL с использованием docker swarm + patroni + zookeeper + haproxy
Работа в рамках тестового стенда 10.130.0.31
Создание сервиса в среде docker swarm
Создаём сервис docker swarm:
docker swarm init
Просмотр сервисов, чтобы узнать хостнейм, которое понадобится в дальнейшем:
docker node ls
ID	HOSTNAME	STATUS	STATUS	MANAGER STATUS	ENGINE VERSION
wt5zphck9xv0qb5zzt7qoh09b 	console	Ready	Active	Leader	20.10.2
Разворачиваем кластер zookeeper
Создаём директорию, в которой будут храниться файлы конфигураций:
mkdir /etc/puru/zookeeper
создаём файл конфигурации зукипера
touch /etc/puru/zookeeper/docker-compose-zookeeper.yml (содержимое в файле)
Деплоим zookeeper
docker stack deploy --compose-file docker-compose-zookeeper.yml patroni




Проверяем, что всё работает, смотрим на список сервисов
docker service ls
ctsqg3oyiy4x   patroni_zoo1       replicated   1/1        zookeeper:3.4            *:2191->2181/tcp
sqdvq9jawpr1   patroni_zoo2       replicated   1/1        zookeeper:3.4            *:2192->2181/tcp
kaxm002rvt9n   patroni_zoo3       replicated   1/1        zookeeper:3.4            *:2193->2181/tcp
Можем сделать пинг сервисов и проверку логов
mntr | nc localhost 2191
docker service logs {{id}}

Работа с patroni
Создаём отдельную директорию
Mkdir /etc/puru/patroni-test
Переходим в неё, создаём файл patroni.yml (содержимое в файле)
Patroni.yml - основной конфигурационный файл, его  мы будем копировать в докер образ, так что любые изменения, внесенные в него, требуют ребилда образа.

Создаём файл patroni-entrypoint.sh (содержимое в файле)
Смысл данного скрипта в том, что без него не получится запустить сервис с patroni, не зная ip адрес. В случае, когда хост оказывается докер контейнер, нам нужно сначала узнать, какой ip этот контейнер получил, только после этого сможем запустить patroni.
Создаём Dockerfile (содержимое в файле)
Создаём файл docker-compose-patroni.yml (содержимое в файле)
Создаём локальные директории и выставляем права (композ патрони файл монтирует из докерфайла /data/patroni
sudo mkdir /patroni1
sudo chown 999:999 /patroni1
sudo chmod 700 /patroni1		
sudo mkdir /patroni2
sudo chown 999:999 /patroni2
sudo chmod 700 /patroni2
sudo mkdir /patroni3
sudo chown 999:999 /patroni3
sudo chmod 700 /patroni3
Создаём докер образ 
docker build -t patroni-test .
Деплоим патрони-кластер
docker stack deploy --compose-file docker-compose-patroni.yml patroni
После деплоя можно проверить запущены ли сервера и смотрим на id контейнеров
docker ps
Заходим внутрь контейнера
docker exec -ti {{id}} /bin/bash



Смотрим информацию о серверах зукипера:
patronictl -c /etc/patroni.yml list patroni
Видим что-то похожее на:
| Member   | Host       | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7008484060183318559) 
| patroni1 | 10.0.1.218 | Replica | running |  1 |         0 |
| patroni2 | 10.0.1.216 | Leader  | running |  1 |           |
| patroni3 | 10.0.1.217 | Replica | running |  1 |         0 |
Смотрим информацию о серверах зукипера:
patronictl -c /etc/patroni.yml list patroni

Настройка HAProxy 
Создаём конфиг файл haproxy.cfg в новой директории
Создаём Dockerfile в той же директории для haproxy
Создаём файл docker-compose-haproxy.yml
Создаём образ из докерфайла 
docker build -t haproxy-patroni
Деплоим сет хапрокси
docker stack deploy --compose-file docker-compose-haproxy.yml haproxy

Когда HА Proxy запустится, можно будет в контейнере посмотреть статистику кластера
docker ps | grep haproxy
docker exec -ti {{id}} /bin/bash
hatop -s /var/run/haproxy/haproxy.sock (внутри контейнера)
