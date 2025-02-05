# Задание 1. Анализ и планирование

### 1. Описание функциональности монолитного приложения

**Управление отоплением:**

- Пользователи могут включать/выключать отопление
- Пользователи могут устанавливать желаемую температуру
- Система поддерживает включение/выключение датчиков
- Система поддерживает установку температуры на датчиках пользователей

**Мониторинг температуры:**

- Пользователи могут смотреть текущую температуру в системе отопления через веб-интерфейс
- Система предоставляет интерфейс для получения данных о температуре с датчиков пользователей

### 2. Анализ архитектуры монолитного приложения

Система "Тёплый дом" - это монолитное приложение, написанное на Java. Для хранения данных используется СУБД PostgreSQL. Взаимодействие пользователя с системой осуществляется через веб-интерфейс.
Администратор, скорее всего, добавляет новых клиентов в систему, напрямую подключаясь к БД (тк явного api для этого нет).
Датчики получают данные от сервера тоже через http запросы.

### 3. Определение доменов и границы контекстов

- Домен "Управление устройствами"
  * Контекст "публикация данных с датчиков"
  * Контекст "установка температуры"
  * Контекст "управления состоянием датчиков/приборов"
- Домен "Мониторинг устройств"
  * Контекст "просмотр температуры"
  * Контекст "просмотр состояния датчиков/приборов"

### **4. Проблемы монолитного решения**

- При появлении нагрузки в какой-то части функционала придется масштабировать всю систему полностью.
- Данные с датчиков поступают в систему в виде синхронных запросов, что может спровоцировать задержки.
- Судя по api, система не имеет POST запроса для добавления оборудования нового клиента. Значит данные вносятся напрямую в БД. Это может вызвать серьезные проблемы, такие как потеря части данных или всех данных.
- При добавлении новых пользователей в систему время отклика будет неуклонно расти, тк. данные от датчиков будут все сильнее и сильнее нагружать сервер и канал связи.
- В команде на данный момент 5 человек, разрешать конфликты в коде будет все тяжелее.

### 5. Визуализация контекста системы — диаграмма С4

Для просмотра диаграммы в markdown использую плагин для Visual Studio Code
https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title Контекстная диаграмма системы "Тёплый дом"

Person(customer, "Клиент", "Пользователь системы 'Тёплый дом'", $sprite="person")
System(smart_home, "Система 'Тёплый дом'", "Управляет приборами и датчиками пользователей. Предоставляет информацию о текущем состоянии датчиков.")
Person(admin, "Администратор", "Подключает клиента к системе")
System_Ext(sensors, "Датчики/Приборы", "Управляются сервером. Работают на стороне клиента.")

Rel(customer, sensors, "Использует")
Rel(sensors, smart_home, "Управление осуществляется сервером")
Rel(admin, smart_home, "Вносит данные о датчиках клиента в базу данных")
Rel(customer, smart_home, "Просматривает/изменяет температуру через веб-интерфейс, включает/выключает отопление")

@enduml
```

# Задание 2. Проектирование микросервисной архитектуры

**Диаграмма контейнеров (Containers)**

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title Диаграмма контейнеров системы "Тёплый дом"

Person(customer, "Клиент", "Пользователь системы 'Тёплый дом'", $sprite="person")
System_Ext(sensors, "Датчики/Приборы", "Публикуют свои показания/статус")
Container_Ext(notification_gateway, "Notification Gateway", "", "Отправляет sms")

Container_Boundary(smart_home, "Система 'Тёплый дом'") {
    Container(auth_mf, "Микрофронт Авторизация", "React, Module Federation", "Часть веб-интерфейса пользователя, отвечающая за авторизацию")
    Container(sensor_mf, "Микрофронт Управление датчиками", "React, Module Federation", "Часть веб-интерфейса пользователя, отвечающая за вкл/выкл приборов, установку температуры, открытие/закрытие ворот, подключение новых приборов/датчиков")
    Container(monitoring_mf, "Микрофронт Мониторинг", "React, Module Federation", "Часть веб-интерфейса пользователя, отвечающая за наблюдение за системой (графики, статичстика, показания, сбои)")

    Container(api_gw, "Api Gateway", "Kusk", "Проксирует все запросы пользователей")
    Container(sensor_api_gw, "Sensor Api Gateway", "Kusk", "Проксирует запросы датчиков")
    ContainerQueue(senson_kafka, "Sensor Kafka", "Kafka", "Собирает информацию с датчиков")
    ContainerQueue(notifacation_kafka, "Notification Kafka", "Kafka", "Собирает сообщения для клиентов")
    ContainerDb(telemetry_db, "Telemetry DB", "PostgreSQL", "Хранит информацию с датчиков")
    ContainerDb(user_db, "User DB", "PostgreSQL", "Хранит данные о пользователях")
    ContainerDb(sensor_db, "Sensor DB", "PostgreSQL", "Хранит данные о приборах/датчиках")
    ContainerDb(message_db, "Message DB", "PostgreSQL", "Хранит данные о шаблонах сообщений для клиентов")

    Container(auth_mcs, "Микросервис Авторизация", "Java, Spring Boot", "Автризация пользователя, выдача токена")
    Container(sensor_mcs, "Микросервис Управление датчиками", "Java, Spring Boot", "CRUD для датчиков пользователя")
    Container(telemetry_mcs, "Микросервис Телеметрия", "Java, Spring Boot", "Принимает запросы с датчиков")
    Container(monitoring_mcs, "Микросервис Мониторинг", "Java, Spring Boot", "Предоставляет пользователю данные о показателях датчиков/приборов во временной перспективе. Формирует сообщение клиенту в экстренных случаях")
    Container(notification_mcs, "Микросервис Нотификация", "Java, Spring Boot", "Шлет sms клиенту")

    Rel(customer, auth_mf, "Rest")
    Rel(customer, monitoring_mf, "Rest + JWT")
    Rel(customer, sensor_mf, "Rest + JWT")

    Rel(api_gw, auth_mcs, "Rest")
    Rel(api_gw, sensor_mcs, "Rest + JWT")
    Rel(api_gw, monitoring_mcs, "Rest + JWT")

    Rel(sensor_api_gw, telemetry_mcs, "Rest")
    Rel(telemetry_mcs, senson_kafka, "Публикует")
    Rel(monitoring_mcs, senson_kafka, "Слушает")
    Rel(sensor_mcs, senson_kafka, "Слушает")

    Rel(monitoring_mcs, telemetry_db, "")
    Rel(auth_mcs, user_db, "")
    Rel(sensor_mcs, sensor_db, "")
    Rel(notification_mcs, message_db, "")

    Rel(auth_mf, api_gw, "Rest")
    Rel(monitoring_mf, api_gw, "Rest + JWT")
    Rel(sensor_mf, api_gw, "Rest + JWT")

    Rel(sensors, sensor_api_gw, "Rest")
    Rel(notification_gateway, customer, "Sms")
    Rel(notification_mcs, notification_gateway, "binary")
    Rel(notifacation_kafka, notification_mcs, "Слушает")
    Rel(monitoring_mcs, notifacation_kafka, "Публикует ошибку датчика/прибора")
    Rel(auth_mcs, notifacation_kafka, "Публикует статус логина")
    Rel(sensor_mcs, sensors, "Установка температуры/выкл/вкл")
}

@enduml
```

**Диаграмма компонентов (Components)**

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Диаграмма компонентов контейнера "Микросервис авторизации"

Container_Ext(api_gw, "Api Gateway", "Kusk", "Проксирует все запросы пользователей")
ComponentQueue_Ext(notifacation_kafka, "Notification Kafka", "Kafka", "Собирает сообщения для клиентов")
Container_Boundary(auth, "Микросервис авторизации") {
    ComponentDb(user_db, "User DB", "PostgreSQL", "Хранит данные о пользователях")
    Component(service, "Сервис Аутентификации", "Spring Mvc Rest Controller", "Производит авторизацию и аутентификацию пользователя")
    Component(jwt_service, "Сервис Валидации", "Java", "Производит проверку и генерацию JWT")
}

Rel(api_gw, service, "/login")
Rel(api_gw, jwt_service, "/validate")
Rel(service, user_db, "")
Rel(service, jwt_service, "")
Rel(service, notifacation_kafka, "Сообщение (успех/неуспех)")

@enduml
```

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Диаграмма компонентов контейнера "Микросервис Управление датчиками"

Container_Ext(api_gw, "Api Gateway", "Kusk", "Проксирует все запросы пользователей")
ComponentQueue_Ext(senson_kafka, "Sensor Kafka", "Kafka", "Собирает информацию с датчиков")
System_Ext(sensors, "Датчики/Приборы", "Ожидание команды от сервера")
Container_Boundary(sensor, "Микросервис Управления датчиками") {
    ComponentDb(sensor_db, "Sensor DB", "PostgreSQL", "Хранит данные о датчиках/приборах")
    Component(service, "Сервис Мониторинга", "Spring Mvc Rest Controller", "CRUD для датчиков/приборов")
    Component(telemetry_listener, "Telemetry Listener", "Java", "Вычитывает топик из kafka и получает самые последние данные с датчиков для обновления в БД")
}

Rel(api_gw, service, "")
Rel(service, sensor_db, "")
Rel(telemetry_listener, senson_kafka, "")
Rel(service, telemetry_listener, "")
Rel(service, sensors, "[async] установка температуры/выкл/вкл")

@enduml
```

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Диаграмма компонентов контейнера "Микросервис Мониторинг"

Component_Ext(api_gw, "Api Gateway", "Kusk", "Проксирует все запросы пользователей")
ComponentQueue_Ext(senson_kafka, "Sensor Kafka", "Kafka", "Собирает информацию с датчиков")

ComponentQueue_Ext(notifacation_kafka, "Notification Kafka", "Kafka", "Собирает сообщения для клиентов")
Container_Boundary(monitoring, "Микросервис Мониторинг") {
    ContainerDb(telemetry_db, "Telemetry DB", "PostgreSQL", "Хранит исторические данные с датчиков")
    Component(service, "Monitoring Service", "Spring Boot Mvc Controller", "Принимает запросы, возвращает ответы")
}

Rel(api_gw, service, "")
Rel(service, telemetry_db, "")
Rel(service, senson_kafka, "Вычитывает телеметрию")
Rel(service, notifacation_kafka, "Публикует сообщения о ЧП (выкл отопления, пожар)")

@enduml
```

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Диаграмма компонентов контейнера "Микросервис Телеметрия"

Component_Ext(api_gw, "SensorApi Gateway", "Kusk", "Проксирует запросы датчиков")
ComponentQueue_Ext(senson_kafka, "Sensor Kafka", "Kafka", "Собирает информацию с датчиков")

Container_Boundary(telemetry, "Микросервис Телеметрия") {
    Component(service, "Monitoring Service", "Spring Boot Mvc Controller", "Принимает запросы от датчиков")
}

Rel(api_gw, service, "")
Rel(service, senson_kafka, "Публикует телеметрию")

@enduml
```

```puml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Диаграмма компонентов контейнера "Микросервис Нотификация"

Component_Ext(notification_gateway, "Notification Gateway", "", "Отправляет sms")
ComponentQueue_Ext(notifacation_kafka, "Notification Kafka", "Kafka", "Собирает сообщения для клиентов")

Container_Boundary(notify, "Микросервис Нотификация") {
    Component(service, "Notification Service", "Spring Boot", "Принимает сообщения для клиентов, форматирует и отправляет")
    Component(formatting, "Formatting Service", "Bean", "Форматирует сообщения согласно шаблону")
    ContainerDb(message_db, "Message DB", "PostgreSQL", "Хранит данные о шаблонах сообщений для клиентов")
}


Rel(service, notification_gateway, "Публикует сообщения")
Rel(service, notifacation_kafka, "Слушает сообщения для клиентов")
Rel(formatting, message_db, "")
Rel(service, formatting, "")

@enduml
```

**Диаграмма кода (Code)**

```puml
@startuml
title Диаграмма кода компонента Управления датчиками

top to bottom direction

!includeurl https://raw.githubusercontent.com/RicardoNiepel/C4-PlantUML/master/C4_Component.puml

class SmartHomeController {
  + service: SmartHomeService
  + delete()
  + register()
  + update()
  + list(clientId: uuid): List<Sensor>
}

class SmartHomeService {
  + repository: SmartHomeRepository
  + delete()
  + register()
  + update()
  + list(clientId: uuid): List<Sensor>
}

class TelemetryListener {
  + service: SmartHomeService
  + listen(telemetry: Sensor)
}

class SmartHomeRepository {
  + delete()
  + register()
  + update()
  + list(clientId: uuid): List<Sensor>
}

struct Sensor {
  + id: uuid
  + status: Status
  + currentValue: Double
}

SmartHomeController --> SmartHomeService
SmartHomeService -> SmartHomeRepository : CRUD
TelemetryListener --> SmartHomeService: update
@enduml
```

```puml
@startuml
title Диаграмма кода компонента Мониторинг

top to bottom direction

!includeurl https://raw.githubusercontent.com/RicardoNiepel/C4-PlantUML/master/C4_Component.puml

class MonitoringController {
  + service: MonitoringService
  + makeReport()
}

class MonitoringService {
  + repository: TelemetryRepository
  + update()
  + saveTelemetry()
  + makeReport()
}

class TelemetryListener {
  + service: MonitoringService
  + listen(telemetry: SensorData)
}

class TelemetryRepository {
  + update()
  + saveTelemetry()
  + findByParams(sensorId: uuid): List<SensorData>
}

struct SensorData {
  + id: uuid
  + date: timastamp
  + value: Double
}

MonitoringController --> MonitoringService
MonitoringService -> TelemetryRepository
TelemetryListener --> MonitoringService: saveTelemetry

@enduml
```

# Задание 3. Разработка ER-диаграммы

@startuml
entity Client {
    * id : uuid
    --
    + name : Strin
    + surname : String
    + username : String
    + passwordHash : String
}

entity Sensor {
    * id : uuid
    --
    + name : String
    + type_id : SensorType
}

entity SensorType {
    * id : uuid
    --
    + name : String
}

entity SensorData {
    * id : uuid
    --
    + sensor_id : uuid
    + date: timestamp
    + status : Status
    + value : Double
}

entity ClientSensors {
    * client_id : uuid
    * sensor_id : uuid
    --
}

ClientSensors }o--|| Client
ClientSensors }o--|| Sensor
Sensor ||--o{ SensorData
SensorType ||--o{ Sensor

@enduml