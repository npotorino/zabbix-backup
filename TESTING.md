
# create test env

## mysql

```bash
docker network create zabbix-net

docker run --name mysql-server-test -t \
    -p 3306:3306 \
    -e MYSQL_DATABASE="zabbix" \
    -e MYSQL_USER="zabbix" \
    -e MYSQL_PASSWORD="zabbix_pwd" \
    -e MYSQL_ROOT_HOST="%" \
    -e MYSQL_ROOT_PASSWORD="root_pwd" \
    --network=zabbix-net \
    -v mysql-data22:/var/lib/mysql \
    --restart unless-stopped \
    --cap-add=sys_nice \
    -d mariadb \
    --character-set-server=utf8 \
    --collation-server=utf8_bin \
    --default-authentication-plugin=mysql_native_password \
    --disable-log-bin

docker run --name zabbix-server-mysql -t \
    -e DB_SERVER_HOST="mysql-server-test" \
    -e MYSQL_DATABASE="zabbix" \
    -e MYSQL_USER="zabbix" \
    -e MYSQL_PASSWORD="zabbix_pwd" \
    -e MYSQL_ROOT_PASSWORD="root_pwd" \
    --network=zabbix-net \
    --restart unless-stopped \
    -d zabbix/zabbix-server-mysql:alpine-6.0-latest

```

## postgresql

```bash
docker network create zabbix-net

...to be defined...

docker run --name zabbix-server-pgsql -t \
    ...to be defined... \
    --network=zabbix-net \
    --restart unless-stopped \
    -d zabbix/zabbix-server-pgsql:alpine-6.0-latest
```

# clean env

```bash
docker rm -f zabbix-server-mysql zabbix-server-pgsql mysql-server-test
docker volume rm mysql-data22
docker network rm zabbix-net
```
