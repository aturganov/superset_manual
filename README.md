# superset_manual ver 1.4.2

Загрузка кода новой версии
https://superset.apache.org/docs/installation/installing-superset-using-docker-compose

# Инсталляция в докере (сборка коробки). Версия 1.4.2 
git checkout 1.4.2
TAG=1.4.2 docker-compose -f docker-compose-non-dev.yml pull
# В случае windows установить строчку в docker-compose-non-dev.yml
x-superset-image: &superset-image apache/superset:1.4.2

# Запуск докера
TAG=1.4.2 docker-compose -f docker-compose-non-dev.yml up -d

# Перенос данных
# Создаем дамп базы данных в старой версии
docker exec -it {container_id_old} pg_dump  -U superset superset -f /var/lib/postgresql/backup.sql
# Копирование дампа на локальную папку
docker cp {container_id_old}:/var/lib/postgresql/backup.sql ./

# Копирание дампа в новый контейнер
docker cp ./backup.sql {container_id_1.4.2}:/var/lib/postgresql/
# Редактирование базы в новом контейнере 
docker exec -it {container_id_1.4.2} psql -U superset
# psql удаление старой схемы (текущих данных по умолчанию)
drop schema public cascade;
create schema public;

# Опционально. Сторонник хранения базы Postgres на внешнем диске

# Останавливаем докер и начинаем донастраивать superset
TAG=1.4.2 docker-compose -f docker-compose-non-dev.yml down
# Редактируем файл. docker-compose-non-dev.yml
# Прописываем локальный путь хранения базы данных Postgres
# Открываем порт в докере 5432 для внешнего доступа
  db:
    env_file: docker/.env-non-dev
    image: postgres:10
    container_name: superset_db
    restart: unless-stopped
    volumes:
      # - db_home:/var/lib/postgresql/data
      - "{local_path}db/superset:/var/lib/postgresql/data"
    ports:
      - 5432:5432
# ... -> комментируем настройку volume (в конце файла) 
  # db_home:
  #   external: false
# Останавливаем докер и начинаем донастраивать superset. Опция -d - запуск в фоновом режиме.
TAG=1.4.2 docker-compose -f docker-compose-non-dev.yml up -d 

# Поднимаем дамп
docker exec -it {container_id_1.4.2} psql -U superset -d superset -f /var/lib/postgresql/backup.sql
# Добавляем поле (если его нет) в базе. При переходе на 1.4.2 может быть такая ситуация. Без нового поля будут ошибки работы настроек загрузки данных
alter table public.dbs add column allow_csv_upload boolean;



# Останавливаем докер и начинаем донастраивать superset
# ! не используйте опцию -v, это приведет к удалению данных в базе
TAG=1.4.2 docker-compose -f docker-compose-non-dev.yml down


#### Приступаем к донастройке коробки Superset

# Добавляем Clickhouse
# В папку ./superset/docker добавляем файл requirements-local.txt со следующим содержимым:
clickhouse-driver==0.2.3
clickhouse-sqlalchemy==0.1.8

# Добавляем опцию CROSS-FILTERS (взаимофильтрация между графиками)
# В папку ./superset/docker/pathonpath_dev добавляем файл superset_config_docker.py со следующим содержимым
FEATURE_FLAGS = {
    "DASHBOARD_CROSS_FILTERS": True,
}

# Поднимаем докер
TAG=1.4.2 docker-compose -f docker-compose-non-dev.yml up -d

# ! Не забывайте делать бэкам базы superset в postgres
