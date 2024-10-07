# Порядок действий 

Файлы конфигурации монтируются в контейнеры. 

У мастера и реплики должны быть одинаковые `max_wal_senders`.

Отличаются адреса, которые они слушают. 

Мастер: `listen_addresses = 'localhost,172.21.0.2'`
Реплика: `listen_addresses = 'localhost,172.21.0.3'`

Также отличаются `wal_level`.

Ip-адреса жестко прописаны в `docker-compose.yml` чтобы наверняка.

## Запускаем все контейнеры

```bash
docker compose up -d
```

## Делаем бекап мастера

Подключаемся через pgslave к pgmaster и делаем бекап сервера

```bash
docker exec -it pgslave /bin/bash
```

Далее делаем бекап мастера. В этой команде есть важный параметр -R. Он означает, что PostgreSQL-сервер также создаст пустой файл standby.signal. Несмотря на то, что файл пустой, само наличие этого файла означает, что этот сервер — реплика.

```bash
pg_basebackup -P -R -X stream -c fast -h 172.21.0.2 -U replicator -D /backup
```

Выходим из pqslave

```
exit
```

## Разворачиваем бекап на реплике

Останавливаем реплику

```bash
docker compose stop pgslave
```

Очищаем папку данных реплики и копируем бекап с мастера

```bash
sudo rm -rf ./postgres_data_slave/*
sudo cp -a ./postgres_backup/* ./postgres_data_slave/
```

Запускаем реплику

```bash
docker compose start pgslave
```

## Проверяем работу реплики

Создадим данные на мастере

```bash
docker exec -it pgmaster /bin/bash
su - postgres
psql -c "CREATE TABLE test_table (id INT, name TEXT);"
psql -c "INSERT INTO test_table (id, name) VALUES (1, 'test');"
psql -c "INSERT INTO test_table (id, name) VALUES (2, 'foobar');"
exit
exit
```

Проверим данные на реплике

```bash
docker exec -it pgslave /bin/bash
su - postgres
psql -c "SELECT * FROM test_table;"
```