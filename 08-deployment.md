# 08. Развертывание

## Целевая среда

MVP разворачивается в двух типах сред:

- **полевой контур** вдоль трассы: DAS-интеррогаторы, камеры, видеосерверы и edge-узлы на участках;
- **центральный контур**: Kubernetes-кластер с API, workers, Kafka, сервисами аналитики, АРМ и хранилищами.

Для локальной разработки допускается Docker Compose с эмуляторами edge-узлов, камер и Kafka.

## Deployment-диаграмма

```mermaid
C4Deployment
    title Развертывание MVP DAS Monitor ВСМ

    Deployment_Node(field, "Полевой контур вдоль трассы", "Участки ВСМ") {
        Container(das, "DAS-интеррогаторы", "Полевое оборудование", "Съем сигнала с оптоволокна")
        Container(cameras, "Камеры и видеосерверы", "Video", "Видеоархив и онлайн-поток")
        Container(edge, "Edge-узлы", "Linux, Go/Python", "Предобработка, буферизация, отправка событий")
    }

    Deployment_Node(k8s, "Центральный контур", "Kubernetes") {
        Container(web, "АРМ оператора", "React/TypeScript", "Web UI")
        Container(api, "Backend API", "Go", "Команды, карточки, роли")
        Container(ml, "ML workers", "Python", "Классификация событий")
        Container(severity, "Сервис критичности", "Go", "Приоритеты и инциденты")
        Container(video, "Сервис видео", "Go", "Запрос видеофрагментов")
        Container(twin, "Сервис цифрового двойника", "Go/Python", "Состояние участков")
    }

    Deployment_Node(data, "Слой данных", "Stateful services") {
        Container(kafka, "Kafka", "Cluster", "Потоки событий")
        ContainerDb(pg, "PostgreSQL/PostGIS", "DB", "Источник истины")
        ContainerDb(tsdb, "TimescaleDB", "DB", "Временные ряды")
        ContainerDb(s3, "S3/MinIO", "Object storage", "Видео, DAS-фрагменты, модели")
    }

    Deployment_Node(user, "Рабочее место", "Браузер") {
        Container(browser, "Браузер оператора", "Web", "АРМ")
    }

    Rel(das, edge, "DAS stream")
    Rel(edge, kafka, "mTLS / event stream")
    Rel(browser, web, "HTTPS")
    Rel(web, api, "HTTPS API")
    Rel(api, pg, "SQL")
    Rel(api, s3, "Presigned access / internal API")
    Rel(ml, kafka, "Consumer/producer")
    Rel(ml, tsdb, "Write features")
    Rel(ml, s3, "Read model, write fragments")
    Rel(severity, kafka, "Consumer/producer")
    Rel(video, cameras, "Video API")
    Rel(twin, pg, "SQL")
    Rel(twin, tsdb, "Read trends")

    UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="2")
```

## Развертываемые процессы

| Процесс | Тип | Масштабирование |
|---|---|---|
| Backend API | Stateless | Горизонтально по HTTP-нагрузке |
| АРМ оператора | Static web / frontend | Через web-сервер или CDN в защищенном контуре |
| ML workers | Stateless workers | Горизонтально по лагу Kafka |
| Сервис критичности | Stateless consumer | Горизонтально с учетом partitioning |
| Сервис видео | Stateless | Горизонтально по числу запросов к видеосерверам |
| Сервис цифрового двойника | Stateful logic, stateless runtime | Горизонтально осторожно, с блокировками по участку |
| Edge-узел | Локальный процесс | Масштабируется числом участков и интеррогаторов |
| Cleanup job | CronJob | По расписанию, один активный экземпляр |

## Stateful-компоненты

- Kafka cluster.
- PostgreSQL/PostGIS.
- TimescaleDB.
- S3/MinIO.
- Локальный буфер edge-узла при потере связи.

Stateful-компоненты должны иметь резервное копирование, мониторинг места, политики восстановления и отдельные права доступа.

## Конфигурация и секреты

| Секрет / настройка | Где хранить | Комментарий |
|---|---|---|
| TLS-сертификаты edge-узлов | Secret store / Kubernetes Secrets | Нужны для доверенной передачи событий |
| Пароли БД | Secret store | Не хранить в репозитории |
| Ключи S3/MinIO | Secret store | Раздельные права для чтения моделей и записи артефактов |
| Токены видеосерверов | Secret store | Ограничить доступ сервисом видео |
| Параметры retention | ConfigMap / таблица настроек | Меняются без пересборки приложения |
| Активная версия модели | PostgreSQL metadata | Не задавать вручную в переменных окружения |

## Отличия окружений

| Окружение | Особенности |
|---|---|
| Локальное | Docker Compose, эмулятор DAS, fake video API, локальный MinIO |
| Тестовое | Kubernetes namespace, тестовые топики Kafka, синтетические сигналы, набор камер-эмуляторов |
| Предпромышленное | Подключение к пилотному участку или потоку записанных сигналов, реальные edge-узлы |
| Production | Полная карта участков, реальные роли, резервное копирование, алерты, регламенты поддержки |

## Обновление без потери задач

- API и workers обновляются rolling update.
- Kafka consumer должен фиксировать offset только после успешной записи состояния.
- Миграции БД выполняются перед выпуском версии приложения, совместимой со старой и новой схемой.
- Edge-узел при обновлении сохраняет локальный буфер.
- Новая модель сначала регистрируется как `candidate`, затем активируется явно.
