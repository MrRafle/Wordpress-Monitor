# WordPress Monitor — контейнерный стенд мониторинга

Учебный проект курсовой работы по дисциплине **Мониторинг и наблюдаемость** (МАИ, кафедра 307).

Контейнерная инфраструктура WordPress с полным контуром наблюдаемости: метрический мониторинг через **Zabbix** и централизованный сбор логов через **ELK Stack**.

---

## Состав стенда

| Контейнер | Образ | Порт | Роль |
|---|---|---|---|
| zabbix-db | mariadb:11.4 | — | БД Zabbix Server |
| zabbix-server | zabbix/zabbix-server-mysql:ubuntu-7.4-latest | 10051 | Ядро мониторинга |
| zabbix-web | zabbix/zabbix-web-nginx-mysql:ubuntu-7.4-latest | **8080** | Web UI Zabbix |
| nginx-custom | nginx:latest + zabbix-agent2 | **81** | Веб-сервер + агент |
| mysql-wp | mariadb:11.4 + zabbix-agent2 | — | БД WordPress + агент |
| wordpress | wordpress:php8.3-apache + zabbix-agent2 | **82** | CMS WordPress + агент |
| elasticsearch | elasticsearch:8.13.0 | 9200 | Хранилище логов |
| kibana | kibana:8.13.0 | **5601** | Визуализация логов |
| filebeat | elastic/filebeat:8.13.0 | — | Сбор логов Docker |

---

## Требования

- Docker Engine 24.0+
- Docker Compose Plugin 2.20+
- RAM: не менее 4 ГБ свободной
- Disk: не менее 10 ГБ свободного

---

## Быстрый старт

### 1. Клонировать репозиторий

```bash
git clone https://github.com/MrRafle/Wordpress-Monitor.git
cd Wordpress-Monitor
```

### 2. Установить права на конфиг Filebeat

```bash
sudo chown root:root filebeat/filebeat.yml
sudo chmod 644 filebeat/filebeat.yml
```

### 3. Собрать образы

```bash
docker compose build
```

### 4. Инициализировать БД Zabbix (только при первом запуске)

```bash
# Поднять только БД
docker compose up -d zabbix-db

# Дождаться статуса healthy
docker compose ps zabbix-db

# Залить схему Zabbix
docker run --rm \
  --network wordpress-monitor_zabbix-net \
  zabbix/zabbix-server-mysql:ubuntu-7.4-latest \
  bash -c "zcat /usr/share/doc/zabbix-server-mysql/create.sql.gz | \
           mysql -hzabbix-db -uzabbix -pzabbix_pass zabbix"

# Проверить результат (должен появиться пользователь Admin)
docker exec -it zabbix-db mariadb -uzabbix -pzabbix_pass zabbix \
  -e "SELECT userid, username FROM users;"
```

### 5. Запустить всю инфраструктуру

```bash
docker compose up -d
docker compose ps
```

---

## Проверка сервисов

| URL | Что должно открыться |
|---|---|
| http://localhost:8080 | Страница входа Zabbix (Admin / zabbix) |
| http://localhost:81 | Nginx — тестовая страница |
| http://localhost:82 | WordPress |
| http://localhost:9200 | Elasticsearch JSON |
| http://localhost:5601 | Kibana |

---

## Настройка мониторинга в Zabbix

1. Открыть http://localhost:8080, войти под Admin / zabbix
2. Перейти в **Data collection -> Hosts -> Create host**
3. Добавить три хоста:

| Host name | DNS (Interfaces) | Port | Шаблоны |
|---|---|---|---|
| nginx-custom | nginx-custom | 10050 | Linux by Zabbix agent, Nginx by Zabbix agent |
| wordpress | wordpress | 10050 | Linux by Zabbix agent, Apache by Zabbix agent |
| mysql-wp | mysql-wp | 10050 | Linux by Zabbix agent, MySQL by Zabbix agent |

> В поле Interfaces выбрать тип **Agent**, переключить **Connect to -> DNS**

Через 1–2 минуты иконка ZBX напротив каждого хоста позеленеет.

---

## Настройка Kibana

1. Открыть http://localhost:5601
2. Перейти в **Management -> Stack Management -> Kibana -> Data Views**
3. Нажать **Create data view**:
   - Name: `Docker Logs`
   - Index pattern: `docker-logs-*`
   - Timestamp field: `@timestamp`
4. Перейти в **Discover**, выбрать Data View: Docker Logs
5. Добавить фильтр: `container.labels.com_docker_compose_project = wordpress-monitor`

---

## Остановка

```bash
# Остановить без удаления данных
docker compose down

# Полный сброс (удалить volumes)
docker compose down -v
```

---

## Структура репозитория

```
.
├── docker-compose.yml
├── filebeat
│   └── filebeat.yml
├── mysql-wp
│   ├── Dockerfile
│   ├── supervisord.conf
│   └── zabbix_agent2.conf
├── nginx-custom
│   ├── Dockerfile
│   ├── html
│   ├── supervisord.conf
│   └── zabbix_agent2.conf
├── README.md
└── wordpress
    ├── Dockerfile
    ├── supervisord.conf
    └── zabbix_agent2.conf
```

---

## Известные особенности

- **Zabbix Agent 2** слушает на IPv6 (`::`) — порт виден в `/proc/net/tcp6`, не в `/proc/net/tcp`. Это нормально, Zabbix Server подключается корректно.
- **Параметры `AllowRoot` и `User`** не поддерживаются в Agent 2 — их наличие в конфиге вызывает ошибку `invalid parameter`.
- **filebeat.yml** должен принадлежать root с правами 644 — иначе Filebeat не запустится.
- **Схема БД Zabbix** инициализируется вручную при первом деплое (шаг 4).
