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

### web interface

not needed for backup/restore testing

```bash
docker run --name zabbix-web-nginx-mysql -t \
    -p 8888:8080 \
    -e DB_SERVER_HOST="mysql-server-test" \
    -e MYSQL_USER="zabbix" \
    -e MYSQL_PASSWORD="zabbix_pwd" \
    -e ZBX_SERVER_HOST="zabbix-server-mysql" \
    -e PHP_TZ="Europe/Riga" \
    --network=zabbix-net \
    --restart unless-stopped \
    -d zabbix/zabbix-web-nginx-mysql:alpine-6.0-latest
```

## postgresql

```bash
docker network create zabbix-net

docker run --name psql-server-test -t \
    -p 5432:5432 \
    -e POSTGRES_DB="zabbix" \
    -e POSTGRES_USER="zabbix" \
    -e POSTGRES_PASSWORD="zabbix_pwd" \
    --network=zabbix-net \
    -v psql-data22:/var/lib/postgresql/data \
    --restart unless-stopped \
    -d postgres:13-alpine

docker run --name zabbix-server-pgsql -t \
    -e DB_SERVER_HOST="psql-server-test" \
    -e POSTGRES_USER="zabbix" \
    -e POSTGRES_PASSWORD="zabbix_pwd" \
    --network=zabbix-net \
    --restart unless-stopped \
    -d zabbix/zabbix-server-pgsql:alpine-6.0-latest
```

## postgresql + timescaledb

```bash
docker network create zabbix-net

docker run --name timescaledb-server-test -t \
    -p 5432:5432 \
    -e POSTGRES_DB="zabbix" \
    -e POSTGRES_USER="zabbix" \
    -e POSTGRES_PASSWORD="zabbix_pwd" \
    --network=zabbix-net \
    -v tdb-data22:/var/lib/postgresql/data \
    --restart unless-stopped \
    -d timescale/timescaledb:latest-pg13

docker run --name zabbix-server-pgsql -t \
    -e DB_SERVER_HOST="timescaledb-server-test" \
    -e POSTGRES_USER="zabbix" \
    -e POSTGRES_PASSWORD="zabbix_pwd" \
    --network=zabbix-net \
    --restart unless-stopped \
    -d zabbix/zabbix-server-pgsql:alpine-6.0-latest
```


# clean env

```bash
docker rm -f zabbix-server-mysql mysql-server-test zabbix-web-nginx-mysql
docker rm -f zabbix-server-pgsql psql-server-test timescaledb-server-test zabbix-web-nginx-psql
docker volume rm mysql-data22 psql-data22 tdb-data22
docker network rm zabbix-net
```

# simple test commands

| desc | cmd |
|---|---|
| mysql | ./zabbix-dump -H ... -p zabbix_pwd -Z |
| mysql, stdout | ./zabbix-dump -H ... -p zabbix_pwd -Z -o - |
| psql | ./zabbix-dump -H ... -t psql -p zabbix_pwd -Z |
| psql, custom format | ./zabbix-dump -H ... -t psql -p zabbix_pwd -Z -C |
