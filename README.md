# Массивно параллельные кластера PostgreSQL

1. Устанавливаем docker и разворачиваем ADCM:

````
sudo docker pull hub.arenadata.io/adcm/adcm:2.5.1

sudo docker create --name adcm -p 8000:8000 -v /opt/adcm:/adcm/data hub.arenadata.io/adcm/adcm:2.5.1

sudo docker start adcm

# Настроиваем автоматический запуск Docker-контейнера

sudo docker update --restart=on-failure adcm

````

2. Проверяем доступность порта 8000:

````
$ sudo netstat -ntpl | grep 8000

tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      1371/docker-proxy   
tcp6       0      0 :::8000                 :::*                    LISTEN      1377/docker-proxy   

$ curl http://localhost:8000
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" href="/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@300;400;500&family=Roboto:wght@300;400;500&display=swap" rel="stylesheet">
    <title>Arenadata Cluster Manager</title>
....

````

3. Заходим через браузер в ADCM, логин/пароль по умолчанию `admin/admin` и устанавливаем [Arenadata DB](https://docs.arenadata.io/en/ADB/current/get-started/online-install/adb-install/cluster-creation.html).

![Arenadata DB](https://github.com/EugeniaSimonova/postgres2-lesson17/blob/main/img1.png?raw=true)


4. Заходим на мастер-хост и проверяем кластер:

````
jenny@eek-adb-1:~$ sudo su - gpadmin
gpadmin@eek-adb-1:~$ psql -l
                               List of databases
   Name    |  Owner  | Encoding |  Collate   |   Ctype    |  Access privileges  
-----------+---------+----------+------------+------------+---------------------
 adb       | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | =Tc/gpadmin        +
           |         |          |            |            | gpadmin=CTc/gpadmin
 gpperfmon | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | gpadmin=CTc/gpadmin+
           |         |          |            |            | =c/gpadmin
 postgres  | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | =c/gpadmin         +
           |         |          |            |            | gpadmin=CTc/gpadmin
 template1 | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | =c/gpadmin         +
           |         |          |            |            | gpadmin=CTc/gpadmin
(5 rows)

````
 
5. Создадим таблицу на инстансе PostgreSQL:

````
postgres=# CREATE TABLE base_table(                                                     
    id SERIAL PRIMARY KEY,
    time_col timestamptz not null default clock_timestamp(),
    col1 int default random()*1E5,
    col2 int default random()*1E5,
    col3 int default random()*1E5,
    col4 int default random()*1E5,
    col5 int default random()*1E5,
    col6 int default random()*1E5,
    col7 int default random()*1E5,
    col8 int default random()*1E5,
    col9 int default random()*1E5,
    col10 int default random()*1E5,
    col11 text,
    col12 text,
    col13 text);

postgres=# INSERT INTO base_table(time_col, col11, col12, col13) SELECT generate_series(
now() - INTERVAL '24 year',
now(),
INTERVAL '15 second'),
'This is simple text',
'This is simple text',
'This is simple text'
;

INSERT 0 50492161
Time: 445679.396 ms (07:25.679)

postgres=# select pg_total_relation_size('base_table');                               

 pg_total_relation_size 
------------------------
             8656617472
````

6. Создадим таблицу в Greenplum (ArenadataDB):


````
postgres=# CREATE TABLE base_table(                                                     
    id SERIAL PRIMARY KEY,
    time_col timestamptz not null default clock_timestamp(),
    col1 int default random()*1E5,
    col2 int default random()*1E5,
    col3 int default random()*1E5,
    col4 int default random()*1E5,
    col5 int default random()*1E5,
    col6 int default random()*1E5,
    col7 int default random()*1E5,
    col8 int default random()*1E5,
    col9 int default random()*1E5,
    col10 int default random()*1E5,
    col11 text,
    col12 text,
    col13 text
) DISTRIBUTED BY(id);


INSERT INTO base_table(time_col, col11, col12, col13) SELECT generate_series(
now() - INTERVAL '24 year',
now(),
INTERVAL '15 second'),
'This is simple text',
'This is simple text',
'This is simple text'
;

````


7. Выполним запрос на инстансе PostgreSQL:

````
select * from base_table where col3=6049;


    id    |           time_col            | col1  | col2  | col3 | col4  | col5  | col6  | col7  | col8  | col9  | col10 |        col11        |        col12        |        col13        
----------+-------------------------------+-------+-------+------+-------+-------+-------+-------+-------+-------+-------+---------------------+---------------------+---------------------
       20 | 2001-03-17 09:57:35.909326+00 | 28544 | 12247 | 6049 | 41893 | 98235 | 30030 | 61729 | 88452 | 56554 |  8953 | This is simple text | This is simple text | This is simple text
   363795 | 2001-05-19 13:41:20.909326+00 | 97769 | 40704 | 6049 |  5033 |  5614 | 32787 | 86151 | 71040 | 13661 | 30311 | This is simple text | This is simple text | This is simple text
...
(488 rows)

Time: 201991.576 ms (03:21.992)
````

8. Выполним запрос в Arenadata DB:
````
select * from base_table where col3=6049;
  
    id    |           time_col            | col1  | col2  | col3 | col4  | col5  | col6  | col7  | col8  | col9  | col10 |        col11        |        col12        |        col13        
----------+-------------------------------+-------+-------+------+-------+-------+-------+-------+-------+-------+-------+---------------------+---------------------+---------------------
  6141504 | 2023-06-27 00:20:48.943044+00 | 22482 | 49693 | 6049 | 38439 | 40769 |  3725 | 70477 | 92509 | 66175 | 51139 | This is simple text | This is simple text | This is simple text
  8502315 | 2024-08-09 21:03:33.943044+00 | 76187 | 86688 | 6049 | 50358 | 14600 | 89779 | 67854 | 13021 | 25946 | 66616 | This is simple text | This is simple text | This is simple text
  9185221 | 2024-12-06 10:30:03.943044+00 | 62167 |  2696 | 6049 | 85401 | 87852 | 84650 | 51468 | 15978 | 18403 | 13979 | This is simple text | This is simple text | This is simple text
...
(509 rows)

Time: 754.729 ms

````

Запроса в Arenadata DB выполнился в 267 раз быстрее.
