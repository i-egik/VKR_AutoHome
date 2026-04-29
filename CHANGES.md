# Журнал изменений

Все значимые изменения в проекте документируются в этом файле.

Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.0.0/),
версии следуют [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added

- Задача 16: подробные Javadoc-комментарии на русском языке для всех Java-классов
  - Main-класс, enum-ы (`DeviceConnectionType`, `DeviceFormat`, `DeviceProtocol`, `SubsystemType`)
    с описанием каждой константы и сценария применения (патч 19-P01)
  - Domain-сущности: все поля и JPA lifecycle callbacks задокументированы (патч 19-P01)
  - Репозитории: class Javadoc + описание кастомных методов и JPQL-запросов (патч 19-P02)
  - Интерфейсы сервисов: полные @param/@return/@throws (патч 19-P02)
  - Реализации сервисов: class Javadoc с описанием алгоритмов, {@inheritDoc} где уместно (патч 19-P03)
  - Инфраструктурный слой: коннекторы (File, Network, Serial, Modbus TCP/RTU, TIPI),
    `DeviceConnectorFactory`, `DevicePollingService` — алгоритмы, fallback-значения,
    константы таймаутов (патч 19-P04)
  - Web-слой: `DeviceMapper`, все делегаты REST API — маппинг, авторизация, граничные случаи (патч 19-P05)
  - Security: `AuthorizationServerConfig`, `ResourceServerConfig` — алгоритм выбора ключей,
    жизненный цикл токенов, модель доступа (патч 19-P05)

- Задача 15: README.md с описанием проекта и инструкциями
  - Обзор проекта, стек технологий
  - Встроенные PlantUML-диаграммы (C4 Container, C4 Component, 2 sequence-диаграммы)
    со ссылками на файлы в `docs/`
  - Быстрый старт: `docker-compose up`, health-check, примеры curl
  - Таблица API-endpoints, пример создания Modbus-устройства
  - OAuth2-авторизация (curl-пример получения токена)
  - Таблицы конфигурации, типов устройств, Flyway-миграций
  - Структура проекта, инструкции по разработке и добавлению новых endpoint'ов

- Задача 14: Поле description — описание устройства для отображения в интерфейсе
  - `V13__add_device_description.sql`: колонка `description TEXT` в таблице `devices`
  - `house.yaml`: поле `description` в схемах `Device`, `DeviceCreate`, `DeviceUpdate`
  - `DeviceEntity`: поле `description`; `DeviceServiceImpl.update`: patch-поддержка
  - `DeviceMapper`: маппинг `description` в `toDto`/`fromCreate`/`fromUpdate`
  - `DeviceDescriptionIntegrationTest` (4 теста): создание с описанием, без описания,
    обновление, независимое хранение описаний разных устройств

- Задача 13: Поле name — имя устройства для отображения в интерфейсе
  - `V12__add_device_name.sql`: колонка `name TEXT` в таблице `devices`
  - `house.yaml`: поле `name` в схемах `Device`, `DeviceCreate`, `DeviceUpdate`
  - `DeviceEntity`: поле `name`; `DeviceServiceImpl.update`: patch-поддержка
  - `DeviceMapper`: маппинг `name` в `toDto`/`fromCreate`/`fromUpdate`
  - `DeviceNameIntegrationTest` (4 теста): создание с именем, без имени, обновление,
    независимое хранение имён разных устройств

- Задача 12: PlantUML-диаграммы архитектуры стенда
  - `docs/c4-container.puml`: C4 Container — docker-compose стенд (house-main,
    house-secondary, modbus-emulator, два SQLite)
  - `docs/c4-component.puml`: C4 Component — внутренняя архитектура house-main
    (делегаты, сервисы, коннекторы, репозитории, OAuth2)
  - `docs/sequence-polling.puml`: последовательность фонового опроса Modbus-устройства
    (@PostConstruct → scheduleAtFixedRate → ReadInputRegisters → device_values)
  - `docs/sequence-integration-test.puml`: последовательность интеграционного теста
    DeviceValuesIntegrationTest (подготовка, save, GET /values, GET /values?last=true, 404)

- Задача 11: Настройки адреса и количества регистров Modbus-устройств
  - `V11__add_modbus_register_count.sql`: новое поле `modbus_register_count INTEGER DEFAULT 1`
  - `house.yaml`: поля `modbus-unit-id`, `modbus-register-address`, `modbus-register-count`
    добавлены в схемы `Device`, `DeviceCreate`, `DeviceUpdate`
  - `DeviceEntity`: поле `modbusRegisterCount`; `DeviceServiceImpl.update`: патч-поддержка
    трёх новых Modbus-полей
  - `DeviceMapper`: маппинг `modbusUnitId`, `modbusRegisterAddress`, `modbusRegisterCount`
    в `toDto`, `fromCreate`, `fromUpdate`
  - `ModbusTcpConnector`, `ModbusRtuConnector`: используют `modbusRegisterCount` для
    `ReadInputRegistersRequest(register, count)`; при `count > 1` значения возвращаются
    как строка, разделённая пробелами (напр. `"42 43 44"`)
  - `ModbusDeviceSettingsIntegrationTest` (5 тестов): создание устройств с разными
    `modbus-register-address`/`modbus-register-count`, обновление настроек, проверка
    независимого хранения для разных устройств

- Задача 10: Флаг `last` для получения последнего значения устройства
  - `house.yaml`: добавлен query-параметр `last` (boolean, default: false) в
    `GET /v1/devices/{device-id}/values` — при `last=true` возвращает массив из одного
    (последнего по времени) элемента
  - `DeviceValueRepository`: метод `findFirstByDeviceIdOrderByRecordedAtDesc`
  - `DeviceValueService` / `DeviceValueServiceImpl`: метод `findLastByDeviceId`
  - `DevicesDelegateImpl`: обновлена сигнатура `getDeviceValues(UUID, Boolean)`,
    ветка `last=true` возвращает `List.of(lastValue)` или пустой список
  - `DeviceValuesIntegrationTest`: 3 новых теста — `last=true` с историей,
    `last=true` без истории (пустой массив), `last=false` возвращает всю историю
  - `stack.http`: тест 14 — e2e-проверка `?last=true` на Modbus-устройстве

- Задача 9: Интеграционные тесты истории значений Modbus-устройства
  - `DeviceValuesIntegrationTest` (6 тестов): проверка `GET /v1/devices/{id}/values` и
    `GET /v1/devices/{id}/values/{valueId}` — история, сортировка, 404, пустой список
  - `scripts/init-main.sql`: Modbus-устройство теперь стартует с `status=RUNNING`,
    `save_history=1`, `polling_sec=10` — опрос начинается сразу после запуска контейнера
  - `stack.http` (тесты 11–13): e2e-проверка статуса Modbus-устройства, непустой истории
    значений и получения конкретного значения по ID

- Задача 8: Проверка запуска docker-compose стенда и интеграционные тесты
  - Добавлена зависимость `spring-boot-starter-actuator`: health-check `/actuator/health`
    теперь доступен (использовался в health-check docker-compose, ранее возвращал 404)
  - `application.yaml`: конфигурация `management.endpoints.web.exposure.include: health`
  - `Dockerfile`: добавлены `sqlite3` и `curl` в runtime-образ (entrypoint.sh использует
    `sqlite3` для ожидания инициализации схемы Flyway; без него данные не загружались)
  - `house.yaml`: удалён нестандартный обязательный header `Authentication` из POST /events
    и POST /triggers (не использовался делегатами, конфликтовал со стандартным
    `Authorization: Bearer`); добавлен `securitySchemes.bearerAuth`
  - `house.yaml`: поле `data` события изменено с `oneOf` (SubsystemEventAlarmData |
    SubsystemEventNotificationData) на свободный объект (`additionalProperties: true`) —
    устранена ошибка Jackson при десериализации без дискриминатора
  - `EventsDelegateImpl`, `TriggersDelegateImpl`: удалён неиспользуемый параметр
    `authentication` из сигнатур `addEvent`/`addTrigger`
  - `init-main.sql`, `init-secondary.sql`: исправлены значения `communication_type`
    (`'NETWORK'`→`'modbus'`, `'FILE'`→`'i2c'`/`'rs486'`); ранее значения не совпадали
    с enum `Device.CommunicationTypeEnum` и вызывали 500 при `GET /v1/devices`
  - `DeviceMapper`: защитная обработка неизвестных значений `communicationType` (try-catch)
  - Тесты `EventsControllerTest`, `TriggersControllerTest`: убраны заглушки
    `header("Authentication", ...)` — теперь авторизация проходит только через JWT
  - **Результат e2e**: все 10 сценариев из `stack.http` проходят (10/10 PASS)

- Задача 7: Unit-тесты и интеграционные тесты с покрытием ≥ 80%
  - **Unit-тесты** (`@ExtendWith(MockitoExtension.class)`, 79 тестов, 0 ошибок):
    - `SubsystemServiceImplTest` — CRUD подсистем
    - `DeviceServiceImplTest` — CRUD устройств, пагинация, фильтрация
    - `DeviceLifecycleServiceImplTest` — start/stop/restart жизненный цикл
    - `EventServiceImplTest` — сохранение и фильтрация событий
    - `TriggerServiceImplTest` — CRUD триггеров с пагинацией
    - `DeviceValueServiceImplTest` — история значений устройств
    - `DeviceConnectorFactoryTest` — выбор коннектора по формату/типу (9 сценариев)
    - `FileDeviceConnectorTest` — чтение из файла + ошибка отсутствующего файла
    - `TipiConnectorTest` — заглушка протокола TIPI
  - **Интеграционные тесты** (`@SpringBootTest + @AutoConfigureMockMvc`):
    - `SubsystemsControllerTest` — REST CRUD + devices, 404-обработка
    - `DevicesControllerTest` — REST CRUD + start/stop/values
    - `EventsControllerTest` — OAuth2: 401 без токена, 403 неверный scope, 201 с корректным
    - `TriggersControllerTest` — OAuth2: 401/403/201, проверка фильтрации по subsystemId
  - **JaCoCo** — отчёт о покрытии, исключение сгенерированного кода OpenAPI и
    hardware-зависимых коннекторов; итоговое покрытие: **81%** инструкций
  - **HTTP request файлы** (`http-requests/`) для IntelliJ IDEA HTTP Client:
    - `auth.http` — получение OAuth2 токена (client_credentials)
    - `subsystems.http` — CRUD подсистем с assertions
    - `devices.http` — CRUD устройств + start/stop/restart/values
    - `events.http` — POST событий (с/без токена), GET с фильтром
    - `triggers.http` — POST триггеров (авторизация), GET с фильтром
    - `stack.http` — end-to-end тесты docker-compose стенда (house-main:8080 и
      house-secondary:8081): health-check, аутентификация, авторизация (401 без токена,
      201 с токеном), операции с подсистемами и устройствами
  - `http-client.env.json` — переменные окружения `dev` для HTTP Client
  - `spring-security-test` добавлен как test-зависимость для `SecurityMockMvcRequestPostProcessors.jwt()`

- Задача 6: Персистентность истории значений устройств
  - DeviceValueEntity / DeviceValueRepository / DeviceValueService / ServiceImpl
  - Flyway V9: save_history в таблице devices; V10: таблица device_values с индексом
  - house.yaml: поля device-id, unit, recorded-at в DeviceValue; save-history в Device/Create/Update
  - DevicePollingService: при save_history = true сохраняет каждое прочитанное значение
  - DevicesDelegateImpl: GET /devices/{id}/values и GET /devices/{id}/values/{value-id} 
    теперь используют DeviceValueService вместо заглушки
  - DeviceMapper: toValueDto, маппинг saveHistory в create/update/toDto
  - DeviceEntity: поле saveHistory; DeviceServiceImpl: учёт в update

- Задача 5: Background-опрос устройств с управлением жизненным циклом
  - DevicePollingService: планировщик на ScheduledExecutorService (10 потоков),
    управляет ScheduledFuture на каждое устройство, @PostConstruct восстанавливает RUNNING
  - DeviceLifecycleService/Impl: start/stop/restart — обновляет статус в БД и планировщике
  - house.yaml: новые пути POST /devices/{id}/start, /stop, /restart;
    DeviceStatus enum (RUNNING/STOPPED); поле polling-sec в Device/DeviceCreate/DeviceUpdate;
    TIPI в DeviceFormat; поле status в Device
  - DeviceEntity: поля pollingSec, status
  - DeviceService/Impl: методы updateStatus и findByStatus; учёт pollingSec/status в update
  - DeviceRepository: findByStatus(String)
  - DeviceMapper: маппинг pollingSec и status
  - DevicesDelegateImpl: реализация startDevice, stopDevice, restartDevice
  - Flyway V8: ALTER TABLE devices — polling_sec, status

- Задача 4: Docker-compose стенд с эмулятором Modbus и SQLite volumes
  - Dockerfile: многоэтапная сборка (Maven build → JRE runtime)
  - docker-compose.yml: сервисы house-main (8080), house-secondary (8081), modbus-emulator
  - Конфигурации: config/application-main.yaml, config/application-secondary.yaml
  - Скрипт инициализации БД: scripts/entrypoint.sh (ожидает Flyway, затем применяет данные)
  - Начальные данные: scripts/init-main.sql (2 устройства + внешняя подсистема),
    scripts/init-secondary.sql (1 устройство Serial)
  - SQLite хранится в Docker volumes (house-main-data, house-secondary-data)

- Задача 3: Инфраструктура подключения устройств (File, Network, Modbus TCP/RTU, Serial, TIPI)
  - Новый пакет ru.mifi.house.infrastructure.device с паттерном Strategy
  - Интерфейс DeviceConnector и запись DeviceReading
  - Коннекторы: FileDeviceConnector, NetworkDeviceConnector, ModbusTcpConnector,
    ModbusRtuConnector, SerialDeviceConnector, TipiConnector (заглушка)
  - DeviceConnectorFactory — выбор коннектора по DeviceFormat/DeviceType/Protocol
  - DeviceFormat расширен значением TIPI
  - DeviceEntity: новые поля modbusUnitId и modbusRegisterAddress
  - Flyway V7: ALTER TABLE devices — добавление Modbus-полей
  - Новые зависимости: j2mod 3.2.0 (Modbus), jSerialComm 2.10.4 (Serial)

- Задача 2: M2M-аутентификация подсистем через OAuth2 client credentials с refresh token
  - AuthorizationServerConfig: настройка OAuth2 Authorization Server через
    `OAuth2AuthorizationServerConfigurer` (совместимо с Spring Boot 4.x/Spring Security 7.x)
  - JdbcRegisteredClientRepository и JdbcOAuth2AuthorizationService для персистентного
    хранения OAuth2-клиентов и авторизаций в SQLite
  - CommandLineRunner инициализирует клиент "subsystem-client" (scope: house.events, house.triggers)
    с grant types: client_credentials + refresh_token (TTL: 1 час / 30 дней)
  - ResourceServerConfig: защита POST /v1/triggers scope house.triggers (в дополнение к events)
  - Flyway V5: таблица oauth2_registered_client
  - Flyway V6: таблицы oauth2_authorization и oauth2_authorization_consent

- Задача 1: Базовый DDD-сервис на основе house.yaml
  - Расширен house.yaml: CRUD-эндпоинты для подсистем и устройств, обогащённые схемы
    (SubsystemCreate/Update, DeviceCreate/Update, SubsystemType, DeviceConnectionType,
    DeviceProtocol, DeviceFormat)
  - Добавлена конфигурация генератора: delegate-паттерн OpenAPI, исключения в
    .openapi-generator-ignore
  - Настроен SQLite (через Flyway-миграции) + Hibernate Community Dialect
  - Flyway-миграции V1–V4: таблицы subsystems, devices, subsystem_events, subsystem_triggers
  - Доменный слой (DDD): SubsystemEntity/Service, DeviceEntity/Service,
    EventEntity/Service, TriggerEntity/Service с JPA-репозиториями
  - Веб-слой: SubsystemsDelegateImpl, DevicesDelegateImpl, EventsDelegateImpl,
    TriggersDelegateImpl — реализации делегатов OpenAPI
  - Глобальный обработчик EntityNotFoundException
  - Файл локализации i18n/messages.properties
  - Тестовая конфигурация с SQLite in-memory
