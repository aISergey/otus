#### Подготовка теста
- создадим новый кластер:
```
sudo pg_createcluster --locale ru_RU.UTF-8 --start 17 otus_1
```
- ещё раз проверим список работающих кластеров, видим рабочий порт нового кластера - 5433:
```
sudo pg_lsclusters
```
- заходим в новый кластер:
```
su postgres \
psql -w -p5433
```
- создадим базу данных testdb:
```
create database testdb; \
\c testdb
```
- создаём схему **testnm**: ```CREATE SCHEMA testnm;```
- новая таблица **testnm.t1**: ```CREATE TABLE testnm.t1(c1 integer);```
- добавим данные в **testnm.t1**: ```INSERT INTO testnm.t1 values(1);```
- новая роль **readonly**: ```CREATE role readonly;```
- права **readonly** на подключение к **testdb**: ```grant connect on DATABASE testdb TO readonly;```
- права **readonly** на использование схемы **testnm**: ```grant usage on SCHEMA testnm to readonly;```
- права **readonly** на SELECT всех таблиц схемы **testnm**: ```grant SELECT on all TABLEs in SCHEMA testnm TO readonly;```
- 

<div class="text text_p-small text_default learning-markdown js-learning-markdown"><ol>
<li>создайте пользователя testread с паролем test123</li>
<li>дайте роль readonly пользователю testread</li>
<li>зайдите под пользователем testread в базу данных testdb</li>
<li>сделайте select * from t1;</li>
<li>получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)</li>
<li>напишите что именно произошло в тексте домашнего задания</li>
<li>у вас есть идеи почему? ведь права то дали?</li>
<li>посмотрите на список таблиц</li>
<li>подсказка в шпаргалке под пунктом 20</li>
<li>а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)</li>
<li>вернитесь в базу данных testdb под пользователем postgres</li>
<li>удалите таблицу t1</li>
<li>создайте ее заново но уже с явным указанием имени схемы testnm</li>
<li>вставьте строку со значением c1=1</li>
<li>зайдите под пользователем testread в базу данных testdb</li>
<li>сделайте select * from testnm.t1;</li>
<li>получилось?</li>
<li>есть идеи почему? если нет - смотрите шпаргалку</li>
<li>как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку</li>
<li>сделайте select * from testnm.t1;</li>
<li>получилось?</li>
<li>есть идеи почему? если нет - смотрите шпаргалку</li>
<li>сделайте select * from testnm.t1;</li>
<li>получилось?</li>
<li>ура!</li>
<li>теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);</li>
<li>а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?</li>
<li>есть идеи как убрать эти права? если нет - смотрите шпаргалку</li>
<li>если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды</li>
<li>теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);</li>
<li>расскажите что получилось и почему  </li>
</ol>
</div>
