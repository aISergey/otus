#### Установка Debian на ВМ Hyper-H

- Загружаем "ванильный" дистрибутив:
https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/
- Устанавливаем Debian
- Начальные настройки Debian
  - разрешаем вход под root по SSH
  - конфигурируем сеть
  - проверяем репозиторий (в /etc/apt/sources.list)
  - установим локаль для PostgreSQL: ru_RU.UTF-8

#### Установка PostgreSQL
- заходим на страницу с инструкцией:
https://www.postgresql.org/download/linux/debian/
- настроим репозиторий:
> sudo apt install -y postgresql-common \
> sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
- установим 17 версию:
> sudo apt -y install postgresql-17

#### Начальная настройка PostgreSQL
- удаляем кластер:
> sudo pg_dropcluster --stop 17 main
- создаём кластер:
> sudo pg_createcluster --locale ru_RU.UTF-8 --start 17 main
- устанавливаем пароль суперадмина
- разрешаем работать с PostgreSQL по локальной сети (postgresql.conf и pg_hba.conf)
  > проверяем: sudo netstat -plunt |grep postgres \
  (видим: 0 0.0.0.0:5432)
- настроим параметры работы PostgreSQL под текущее "железо"
