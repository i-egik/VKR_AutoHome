# untitled-house

Система управления IoT-устройствами умного дома с REST API, фоновым опросом через Modbus TCP/RTU, историей значений и OAuth2-авторизацией.

**Стек:** Java 17 · Spring Boot 4 · SQLite + Flyway · OpenAPI 3.0 (code generation) · j2mod (Modbus) · Docker Compose

---

## Архитектура

### Стенд (C4 Container)

Стенд состоит из двух сервисов, базы данных и эмулятора Modbus-устройства:

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
LAYOUT_WITH_LEGEND()
title Стенд untitled-house — C4-схема контейнеров
Person(user, "Пользователь / тест", "Обращается к REST API")
System_Boundary(stand, "Docker-compose стенд") {
    Container(main, "house-main", "Spring Boot 4 / Java 17", "Основной сервис\nПорт: 8080")
    Container(secondary, "house-secondary", "Spring Boot 4 / Java 17", "Вторичный сервис\nПорт: 8081")
    ContainerDb(main_db, "house-main.db", "SQLite", "subsystems, devices,\ndevice_values, events, triggers")
    ContainerDb(secondary_db, "house-secondary.db", "SQLite", "Данные вторичного сервиса")
    Container(modbus, "modbus-emulator", "oitc/modbus-server", "Эмулятор Modbus TCP\nПорт: 502")
}
Rel(user, main, "REST API", "HTTP/JSON")
Rel(user, secondary, "REST API", "HTTP/JSON")
Rel(main, main_db, "JPA / Flyway", "JDBC")
Rel(secondary, secondary_db, "JPA / Flyway", "JDBC")
Rel(main, modbus, "ReadInputRegisters", "Modbus TCP :502")
Rel(secondary, main, "EXTERNAL_REST / OAuth2", "HTTP :8080")
@enduml
```

> Исходники диаграмм: [`docs/c4-container.puml`](docs/c4-container.puml), [`docs/c4-component.puml`](docs/c4-component.puml)

### Компоненты house-main (C4 Component)

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
LAYOUT_WITH_LEGEND()
title house-main — C4-схема компонентов
Person(user, "Клиент / тест")
Container_Boundary(main, "house-main") {
    Component(api, "API Delegates\n(Devices, Subsystems,\nEvents, Triggers)", "Spring MVC", "REST-контроллеры из OpenAPI")
    Component(device_svc, "DeviceService\nDeviceLifecycleService", "Spring Service", "CRUD и жизненный цикл")
    Component(value_svc, "DeviceValueService", "Spring Service", "История значений")
    Component(polling, "DevicePollingService", "Spring Component", "Фоновый опрос устройств")
    Component(connectors, "ModbusTcpConnector\nModbusRtuConnector\nFileDeviceConnector...", "Spring Component", "Чтение данных с устройств")
    Component(security, "OAuth2 / JWT", "Spring Security", "Авторизация")
    ComponentDb(repo, "JPA Repositories", "Spring Data", "SQLite")
}
Rel(user, api, "REST", "HTTP/JSON")
Rel(api, device_svc, "CRUD")
Rel(api, value_svc, "findByDeviceId\nfindLast")
Rel(polling, connectors, "read(device)")
Rel(polling, value_svc, "save(reading)")
Rel(device_svc, repo, "JPA")
Rel(value_svc, repo, "JPA")
@enduml
```

### Последовательность опроса устройства

```plantuml
@startuml
title Фоновый опрос Modbus-устройства
participant "DevicePollingService" as polling
participant "ModbusTcpConnector" as conn
participant "modbus-emulator:502" as emul
participant "DeviceValueService" as svc
database "device_values" as db
polling -> conn: read(device)
conn -> emul: ReadInputRegisters(register, count)
emul --> conn: [values]
conn --> polling: DeviceReading(value, "reg")
alt saveHistory == true
    polling -> svc: save(deviceId, reading)
    svc -> db: INSERT INTO device_values
end
@enduml
```

> Подробная схема: [`docs/sequence-polling.puml`](docs/sequence-polling.puml)

### Последовательность интеграционного теста

```plantuml
@startuml
title DeviceValuesIntegrationTest — поток выполнения
actor "Test" as test
participant "MockMvc" as mvc
participant "DevicesApiDelegate" as api
participant "DeviceValueService" as svc
database "SQLite (in-memory)" as db
test -> mvc: POST /v1/subsystems
mvc -> db: INSERT subsystems
test -> mvc: POST /v1/devices
mvc -> db: INSERT devices
test -> svc: save(deviceId, DeviceReading("42","reg"))
svc -> db: INSERT device_values
test -> mvc: GET /v1/devices/{id}/values
mvc -> api: getDeviceValues(id, last=null)
api -> db: SELECT * ORDER BY recorded_at DESC
db --> api: [{value:"42", unit:"reg"}]
api --> test: 200 OK ✓
@enduml
```

> Подробная схема: [`docs/sequence-integration-test.puml`](docs/sequence-integration-test.puml)

---

## Быстрый старт

### Требования

- Docker и Docker Compose
- (для локальной разработки) Java 17+, Maven 3.9+

### Запуск стенда

```bash
docker-compose up -d --build
```

После запуска доступны:

| Сервис | URL | Описание |
|--------|-----|----------|
| house-main | http://localhost:8080 | Основной сервис |
| house-secondary | http://localhost:8081 | Вторичный сервис |
| house-main health | http://localhost:8080/actuator/health | Health-check |

Первый запрос займёт ~30 секунд — Flyway применяет миграции, затем загружаются начальные данные.

### Проверка работоспособности

```bash
# Health-check
curl http://localhost:8080/actuator/health

# Список подсистем
curl http://localhost:8080/v1/subsystems

# Устройства
curl http://localhost:8080/v1/devices

# История значений Modbus-датчика (заполняется автоматически каждые 10 сек)
curl http://localhost:8080/v1/devices/22222222-2222-2222-2222-222222222222/values

# Последнее значение
curl "http://localhost:8080/v1/devices/22222222-2222-2222-2222-222222222222/values?last=true"
```

---

## API

Полная спецификация: [`src/main/resources/house.yaml`](src/main/resources/house.yaml) (OpenAPI 3.0).

HTTP-файлы с примерами запросов: [`http-requests/`](http-requests/).

### Основные ресурсы

| Метод | URL | Описание |
|-------|-----|----------|
| `GET` | `/v1/subsystems` | Список подсистем |
| `POST` | `/v1/subsystems` | Создать подсистему |
| `GET` | `/v1/devices` | Список устройств |
| `POST` | `/v1/devices` | Зарегистрировать устройство |
| `GET` | `/v1/devices/{id}` | Информация об устройстве |
| `PUT` | `/v1/devices/{id}` | Обновить устройство |
| `DELETE` | `/v1/devices/{id}` | Удалить устройство |
| `POST` | `/v1/devices/{id}/start` | Запустить опрос |
| `POST` | `/v1/devices/{id}/stop` | Остановить опрос |
| `GET` | `/v1/devices/{id}/values` | История значений |
| `GET` | `/v1/devices/{id}/values?last=true` | Последнее значение |
| `GET` | `/v1/devices/{id}/values/{valueId}` | Конкретное значение |
| `GET` | `/v1/events` | Список событий |
| `POST` | `/v1/events` | Зарегистрировать событие *(требует JWT)* |
| `GET` | `/v1/triggers` | Список триггеров |
| `POST` | `/v1/triggers` | Зарегистрировать триггер *(требует JWT)* |

### Пример: создание Modbus-устройства

```json
POST /v1/devices
{
  "title": "Датчик температуры",
  "name": "Температура кухни",
  "description": "Modbus-датчик на кухне, регистры 0–2",
  "subsystem-id": "11111111-1111-1111-1111-111111111111",
  "communication-type": "modbus",
  "device-type": "NETWORK",
  "protocol": "TCP_IP",
  "device-format": "MODBUS_TCP",
  "address": "192.168.1.50",
  "port": 502,
  "modbus-unit-id": 1,
  "modbus-register-address": 0,
  "modbus-register-count": 3,
  "polling-sec": 30,
  "save-history": true
}
```

При `modbus-register-count > 1` значение в истории — пробел-разделённая строка: `"42 43 44"`.

### OAuth2-авторизация

Защищённые endpoints (`POST /v1/events`, `POST /v1/triggers`) требуют Bearer-токена.

```bash
# Получение токена (client_credentials)
curl -X POST http://localhost:8080/oauth2/token \
  -H "Authorization: Basic c3Vic3lzdGVtLWNsaWVudDpzdWJzeXN0ZW0tc2VjcmV0" \
  -d "grant_type=client_credentials&scope=house.events%20house.triggers"

# Credentials: subsystem-client / subsystem-secret
```

---

## Конфигурация

| Переменная окружения | Описание | Пример |
|---------------------|----------|--------|
| `DB_PATH` | Путь к файлу SQLite | `/data/house-main.db` |
| `DB_INIT_SQL` | SQL-скрипт инициализации данных | `/app/scripts/init-main.sql` |
| `SPRING_CONFIG_ADDITIONAL_LOCATION` | Дополнительный конфиг Spring | `file:/app/config/application-main.yaml` |

Конфигурации сервисов: [`config/application-main.yaml`](config/application-main.yaml), [`config/application-secondary.yaml`](config/application-secondary.yaml).

### Типы устройств и форматы

| `device-type` | `device-format` | Описание |
|---------------|-----------------|----------|
| `NETWORK` | `MODBUS_TCP` | Modbus TCP (j2mod) |
| `NETWORK` | `MODBUS_RTU` | Modbus RTU через serial |
| `FILE` | `RAW` | Чтение из файла |
| `NETWORK` | `TIPI` | TIPI-протокол *(не реализован)* |
| `NETWORK` | `RAW` | Сырое TCP-соединение |

### Подсистемы

| `type` | Описание |
|--------|----------|
| `INTERNAL` | Локальные устройства |
| `EXTERNAL_REST` | Другой экземпляр сервиса (M2M OAuth2) |

---

## Разработка

### Сборка и тесты

```bash
# Сборка
mvn compile

# Тесты (101 тест, покрытие ≥ 80%)
mvn test

# Сборка Docker-образа
docker build -t untitled-house .
```

### Структура проекта

```
src/main/java/ru/mifi/house/
├── domain/
│   ├── device/          # DeviceEntity, DeviceService, DeviceServiceImpl
│   │   └── value/       # DeviceValueEntity, DeviceValueService
│   ├── subsystem/       # SubsystemEntity, SubsystemService
│   ├── event/           # EventEntity, EventService
│   └── trigger/         # TriggerEntity, TriggerService
├── infrastructure/
│   └── device/
│       ├── DevicePollingService   # Фоновый опрос
│       ├── DeviceConnector        # Интерфейс коннектора
│       ├── DeviceConnectorFactory # Выбор коннектора по device_format
│       └── connector/             # ModbusTcpConnector, FileDeviceConnector...
├── web/                 # API Delegates, DeviceMapper
├── security/            # OAuth2 конфигурация
└── UntitledHouseApplication.java

src/main/resources/
├── house.yaml           # OpenAPI 3.0 спецификация (источник кодогенерации)
└── db/migration/        # Flyway миграции V1–V13

docs/
├── c4-container.puml    # C4 Container — стенд
├── c4-component.puml    # C4 Component — компоненты house-main
├── sequence-polling.puml        # Последовательность опроса
└── sequence-integration-test.puml  # Последовательность интеграционного теста

http-requests/           # HTTP-файлы для тестирования API в IDE
changes/                 # Patch-файлы изменений (Changes-01..Changes-17)
```

### Добавление нового endpoint

1. Описать операцию в [`src/main/resources/house.yaml`](src/main/resources/house.yaml)
2. Выполнить `mvn generate-sources` для регенерации API-кода
3. Реализовать метод в соответствующем `*DelegateImpl`
4. Добавить тест

### База данных

SQLite + Flyway. Миграции в `src/main/resources/db/migration/`:

| Миграция | Описание |
|----------|----------|
| V1–V6 | Создание таблиц: subsystems, devices, events, triggers, oauth2_* |
| V7 | Поля modbus_unit_id, modbus_register_address |
| V8 | Поля polling_sec, status |
| V9 | Поле save_history |
| V10 | Таблица device_values |
| V11 | Поле modbus_register_count |
| V12 | Поле name (имя для интерфейса) |
| V13 | Поле description (описание для интерфейса) |

---

## Лицензия

Apache 2.0
