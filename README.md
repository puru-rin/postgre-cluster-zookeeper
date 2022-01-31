# postgre-cluster-zookeeper

# Creating a PostgreSQL cluster using docker swarm + patroni + zookeeper + haproxy


## Creating a service in a docker swarm environment

- Create a docker swarm service:

```sh
docker swarm init
```

- View services to find out the hostname, which will be needed later:
```sh
docker node ls
```
| ID | HOSTNAME | STATUS | README | MANAGER STATUS | ENGINE VERSION |
| ------ | ------ | ------ | ------ | ------ | ------ |
| wt5zphck9xv0qb5zzt7qoh09b | console | Ready | Active | Leader | 20.10.2 |


## Deploying a zookeeper cluster

- Create a directory where the configuration files will be stored:

```sh
mkdir /etc/zookeeper
```
- Create a zookeeper config file
```sh
touch /etc/zookeeper/docker-compose-zookeeper.yml
```
> content in  /zookeeper/ folder in docker-compose-zookeeper.yml file

- Deploy zookeeper

```sh
docker stack deploy --compose-file docker-compose-zookeeper.yml patroni
```

- checking that everything works, we look at the list of services
```sh
docker service ls
```
| ID|NAME |MODE |REPLICAS | IMAGE |PORTS |
| ------ | ------ | ------ | ------ | ------ | ------ |
| sqdvq9jawpr1 | patroni_zoo2 |replicated | 1/1 |  zookeeper:3.4 | *:2192->2181/tcp |
| kaxm002rvt9n |  patroni_zoo3 | replicated | 1/1 |zookeeper:3.4 | *:2193->2181/tcp |
| ctsqg3oyiy4x |  patroni_zoo1 | replicated | 1/1 | zookeeper:3.4 |*:2191->2181/tcp |

- We can ping services and check logs
```sh
mntr | nc localhost 2191
docker service logs {$ID}
```

## Working with patroni

- Create a new directory

```sh
mkdir /etc/patroni
```
- Go to it, create a `patroni.yml` file 
> content in /patroni/ folder in patroni.yml file

`patroni.yml` is the main configuration file, we will copy it to the docker image, so any changes made to it require an image rebuild.

- Create file `patroni-entrypoint.sh`
> content in /patroni/ folder in /patroni-entrypoint.sh file

This script is required because it will not possible to start the service with patroni without knowing the ip address in the script. In the case when the host turns out to be a docker container, we first need to find out what ip this container received, only after that we can start patroni.

- Create `Dockerfile` for patroni 
> content in /patroni/ folder in this repository

- Create docker-compose-patroni.yml
> content is also in /patroni/ folder

- Create local directories and set the rights (the docker-compose-patroni mounts the file from the dockerfile /data/patroni)

```sh
sudo mkdir /patroni1
sudo chown 999:999 /patroni1
sudo chmod 700 /patroni1	
sudo mkdir /patroni2
sudo chown 999:999 /patroni2
sudo chmod 700 /patroni2
sudo mkdir /patroni3
sudo chown 999:999 /patroni3
sudo chmod 700 /patroni3
```
- create docker image

```sh
docker build -t patroni-test .
```

- Deploy patroni cluster

```sh
docker stack deploy --compose-file docker-compose-patroni.yml patroni
```
After deployment, you can check if the servers are running and look at the id of the containers

```sh
docker ps
```
- Go to inside the container

```sh
docker exec -ti {$id} /bin/bash
```
- Check information about zookeeper servers
```sh
patronictl -c /etc/patroni.yml list patroni
```

- We see something like:

| Member   | Host       | Role    | State   | TL | Lag in MB |
| ------ | ------ | ------ | ------ | ------ | ------ |
| patroni1 | 10.0.1.218 | Replica | running |  1 |         0 |
| patroni2 | 10.0.1.216 | Leader  | running |  1 |           |
| patroni3 | 10.0.1.217 | Replica | running |  1 |         0 |

## HAProxy installation
-  Create a config file `haproxy.cfg` in a new directory
- Create a `Dockerfile` in the same directory for haproxy
- Create a `docker-compose-haproxy.yml` file
> Contents for file in this repository in /haproxy/ folder

- Create an image from a dockerfile

```sh
docker build -t haproxy-patroni
```
- Deploy haproxy set
```sh
docker stack deploy --compose-file docker-compose-haproxy.yml haproxy
```

 When HAProxy starts, it will be possible to view cluster statistics in the container

```sh
docker ps | grep haproxy
docker exec -ti {{id}} /bin/bash
hatop -s /var/run/haproxy/haproxy.sock (inside container)
```
