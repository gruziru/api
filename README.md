# gruzi.ru client api

## 1. Общие сведения

**Эндпоинт API**

* POST {HOST}/oboz2-order-client-api/v1/messages
* GET {HOST}/oboz2-order-client-api/v1/messages
* POST {HOST}/oboz2-security-auth-tokenizer/v1/login
* POST {HOST}/oboz2-security-auth-tokenizer/v1/refresh
* GET {HOST}/oboz2-dictionary-resource-types-crud/v1/internal/dropdown_with_title

### 1.1. Bearer-авторизация и адреса

1. Во всех запросах должен присутствовать заголовок `Authorization` со значением `Bearer ${ТОКЕН}`
2. Токен (если явно не указано иное) выдается поддержкой Грузи.ру и его значение различается для разных контуров
3. Адреса контуров Грузи.ру
   1. Тестовый контур `https://preprod.gruzi.ru` - доступен только через vpn Грузи.ру
   2. Продуктив `https://go.gruzi.ru`  доступен без VPN

### 1.2. Валидация

1. Если в заголовках запроса отсутствует [токен](#11-bearer-авторизация-и-адреса), то метод возвращает ответ 401 - "Обратившийся не авторизован" и прерывает обработку сообщения. 
2. Если [токен](#11-bearer-авторизация-и-адреса) есть, но он не валиден, возвращается 403 - "Нет доступа"
3. Проверяет заполнение обязательных полей и форматы полей согласно JSON-схеме `openapi.yaml` и возвращает 400 - "Прочие ошибки", если хотя бы одно обязательное поле не заполнено.<BR/>См. [примеры](#5-пример-ответа).
4. Далее проверяется формат данных, если отсутствуют или имеют некорректный тип поля или структуры, возвращается 400 - "Прочие ошибки"<BR/>См. [примеры](#5-пример-ответа).
5. При наличии ошибок нескольких типов формирует в ответе два блока Error с детализацией ошибки каждого типа.
6. Обработка не выполняется при любых ошибках валидации
7. Если в запросе есть заголовок `Oboz-API-errorcode` с любым целочисленным значением, то при любых ошибках вместо кода ответа 400 должен возвращаться код ответа, равный значению заголовка `Oboz-API-errorcode`.
8. Если валидация прошла успешно, возвращает 202 - "Принято к обработке"

### 1.3. API Для работы с заказами

1. У API единственный эндпоинт, который получает POST-запросы с массивом входящих сообщений различного типа.
2. В разделе [Типы входящих сообщений](#2-типы-входящих-сообщений) дана краткая характеристика всем типам, а подробное описание приведено в `openapi.yaml` в формате Open API 3
3. При возникновении в системе Грузи.ру событий и изменений по заказам, созданным через API, Грузи.ру оповещает систему партнера об этих событиях, отправляя POST-запросы с данными событий на callback url, который устанавливается при помощи [UpdateApiSettingsMessage](#21-updateapisettingsmessage)
4. В разделе [Типы исходящих событий](#3-типы-исходящих-событий) дана краткая характеристика всем исходящим событиям, а подробное описание в формате Open API 3 приведено в `openapi.yaml`
5. В callback-запросах передается только один заголовок `Content-Type: application/json`. При необходимости авторизовывать запросы, токен авторизации нужно "зашивать" в URL.

### 1.4 API для работы со справочниками

API справочников требуют клиентской JWT-авторизации, то есть, для взаимодействия с ними требуется получать клиентский JWT-токен, используя логин и пароль от UI платформы Грузи.ру.

#### 1.4.1 Получение пользовательского токена

Для получения токена авторизации выполняется запрос `POST /oboz2-security-auth-tokenizer/v1/login`, в теле которого передается логин и пароль пользователя. В ответ синхронно возвращается токен, который и используется для авторизации запросов к подсистеме справочников через HTTP-заголовок `Authorization: 'Bearer {{token}}'`.

Пример запроса:

```bash
curl --location 'https://{{server}}/oboz2-security-auth-tokenizer/v1/login' \
--header 'Content-Type: application/json' \
--data-raw '{
                "email": "flastname@domain.dom",
                "password": "qwertypoiuyt"
            }'
```

#### 1.4.2 Получение справочника типов транспортных средств

Для получения полного справочника типов транспортных средств выполняется `GET /oboz2-dictionary-resource-types-crud/v1/internal/dropdown_with_title`, который возвращает JSON-массив всех типов ТС с их uuid и наименованиями в системе Грузи.ру.

Пример запроса:

```bash
curl --location 'https://{{server}}/oboz2-dictionary-resource-types-crud/v1/internal/dropdown_with_title' \
--header 'Authorization: Bearer {{token}}' \
--header 'Accept-Language: RU'
```

## 2. Типы входящих сообщений

См. [примеры запросов](#4-примеры-запросов)

### 2.1. UpdateApiSettingsMessage

1. Используется для обновления эндпоинта, на который Грузи.ру отправляет исходящие сообщения
2. Грузи.ру периодически отправляет POST-запросы с данными исходящих событий

### 2.2. CreateOrderMessage

1. Создание нового перевозочного заказа (заявка на перевозку)

### 2.3. CancelOrderMessage

1. Отмена ранее созданного перевозочного заказа
2. Отменить заказ можно только в статусах `NEW`, `ON_DOER_SEARCH`

### 2.4. CreateTrackingOrderMessage

1. Создание трекингового заказа (заявка на трекинг)

### 2.5. UpdateOrderMessage

1. Обновление данных ранее созданного заказа
2. Изменения принимаются только, пока заказа в статусах `NEW`, `ON_DOER_SEARCH`, `CANCELED`
3. Изменить можно только плановое время подачи и маршрут. Для всех остальных изменений требуется отменить заказ и создать новый.

## 3. Типы исходящих событий

1. `OrderCreatedEventMessage` - Результат обработки входящего запроса [CreateOrderMessage](#22-createordermessage)
2. `OrderUpdatedEventMessage` - Результат обработки входящего запроса [UpdateOrderMessage](#25-updateordermessage)
3. `OrderSignedEventMessage` - Заказ подписан экспедитором
4. `OrderResourcesAssignedEventMessage` - Ресурсы на заказ назначены
5. `OrderCancelledEventMessage` - Заказ отменён перевозчиком, заказчиком или экспедитором. В том числе - в результате обработки [CancelOrderMessage](#23-cancelordermessage)
6. `OrderExecutionStartedEventMessage` - Исполнение заказа начато
7. `OrderExecutedEventMessage` - Заказ исполнен
8. `OrderExecutionConfirmedEventMessage` - Исполнение заказа подтверждено
9. `TripTrackingEtaEventMessage` - ETA для точки разгрузки
10. `TripTrackingArrivedEventMessage` - Прибытие в точку разгрузки по треку
11. `TripManualArrivedEventMessage` - Прибытие в точку разгрузки по данным логиста
12. `TripUnloadingStartedEventMessage` - Начало разгрузки
13. `TripUnloadingEndedEventMessage` - Окончание разгрузки
14. `TripReasonOfLateForArrivingEventMessage` - Причина опоздания на точку разгрузки
15. `TrackingOrderCreatedEventMessage` - Трекинговый заказ создан [CreateTrackingOrderMessage](#24-createtrackingordermessage)
16. `TrackingEtaEventMessage` - ETA для точки разгрузки
17. `TrackingArrivedEventMessage` - Прибытие в точку разгрузки по треку
18. `IntegrationSpotOrderStatusEventMessage` - Изменен статус спотового заказа

## 4. Примеры запросов

<details>
<summary>Создание заказа на перевозку</summary>

```json5
[
   {
    "header": {
      "uuid": "805b4d3d-5f8c-47d6-82dd-e37442627472",
      "version": "1.0",
      "type": "createOrder",
      "createdAt": "2021-03-05T06:23:18Z",
      "senderContractorKpp": "770501001",
      "senderContractorInn": "7705739450",
      "transportOperatorInn": "7726630679",
      "receiverContractorInn": "7726630679",
      "receiverContractorKpp": "771401001",
      "senderSystemCode": "sender_prod_msk1",
      "manualSend": false,
      "isTest": false
    },
    "body": {
      "externalNumber": "R100711813",
      "clientInn": "7705739450",
      "contractorInn": "7726630679",
      "species": "quota",
      "type": "standard",
      "date": "2020-09-23",
      "createdAt": "2020-09-23T08:47:00.333Z",
      "contractNumber": "R100711813",
      "transportation": {
        "way": "ftl",
        "route": [
          {
            "type": "loading",
            "organizationTitle": "ООО Рога и копыта",
            "location": {
              "address": "620000, Россия, Свердловская область, Екатеринбург, Машинная улица, 3",
              "coordinates": {
                "lat": 56.812286,
                "long": 60.630047
              },
              "externalCode": "RUEC0005051350"
            },
            "timeslot": {
              "arrive": "2020-09-23T08:47:00.333Z",
              "departure": "2020-09-23T08:47:00.333Z",
              "timezone": "RUS05"
            },
            "contact": {
              "phoneNumber": "8612774554"
            }
          }
        ],
        "needTracking": false,
        "numberOfVehicles": 0,
        "resource": {
          "truckBodyTypeUuid": "88fac672-1f3c-4171-b912-c8ae45ed7805",
          "loadTypes": [
            "back"
          ]
        },
        "tripTariff": {
          "value": 1000000,
          "currency": "RUR"
        }
      },
      "cargo": {
        "type": "Бакалея",
        "cost": {
          "value": 1000000,
          "currency": "RUR"
        },
        "insurance": {
          "value": 1000000,
          "currency": "RUR"
        },
        "weight": 0,
        "volume": 0,
        "freightUnit": {
          "count": 0,
          "type": "pallet"
        },
        "isDangerous": false,
        "temperatureLimits": {
          "min": 4,
          "max": 18
        },
        "etsngs": [
          "021054"
        ]
      },
      "contactPerson": {
        "fio": "Сидоров Иван Васильевич",
        "phone": "74957778899"
      },
      "comment": "Перевозка по договору №1 от 2020-09-22",
      "shipment": [
        "R100821571"
      ],
      "externalId": "R100711813",
      "needCreateAsDraft" : false
    }
  }
]
```

</details>

<details>
<summary>Создание заказа на трекинг груза</summary>

```json5
[
  {
    "header": {
      "uuid": "6F9619FF-8B00-D011-B42D-00CF4FC964FF",
      "version": "1.0",
      "type": "createTrackingOrder",
      "createdAt": "2020-09-23T08:47:00.333Z",
      "senderContractorInn": "7705739450",
      "senderContractorKpp": "771401001",
      "transportOperatorInn": "7726630679",
      "receiverContractorInn": "7726630679",
      "receiverContractorKpp": "7726630679",
      "senderSystemCode": "sender_prod_msk1",
      "manualSend": false,
      "isTest": true
    },
    "body": {
      "externalId": "R100711813",
      "externalNumber": "R100711813",
      "clientInn": "7705739450",
      "contractorInn": "7726630679",
      "date": "2020-09-23",
      "contractNumber": "R100711813",
      "route": [
        {
          "type": "loading",
          "location": {
            "address": "620000, Россия, Свердловская область, Екатеринбург, Машинная улица, 3",
            "coordinates": {
              "lat": 56.812286,
              "long": 60.630047
            },
            "externalCode": "RUEC0005051350"
          },
          "plannedTimeOfArrival": "2020-09-23T08:47:00.333Z",
          "plannedEndTimeOfArrival": "2020-09-23T08:47:00.333Z",
          "parkingRadius": 3000,
          "targetRadius": 1500
        }
      ],
      "doerInn": "123456789",
      "vehicleNo": "A 000 AA 797",
      "trailerNo": "AA 00-00 77",
      "driver": {
        "fio": "Иванов Иван Иванович",
        "phone": "8-910-000-00-00"
      },
      "tariff": {
        "value": 1000000,
        "currency": "RUR"
      },
      "contactPerson": {
        "fio": "Сидоров Иван Васильевич",
        "phone": "74957778899"
      },
      "comment": "Перевозка по договору №1 от 2020-09-22",
      "trackingStartDt": "2020-05-19T21:00:00.000Z",
      "trackingEndDt": "2020-05-19T21:00:00.000Z"
    }
  }
]
```

</details>

<details>
<summary>Отмена заказа по запросу из внешней системы</summary>

```json5
[
{
  "header": {
    "uuid": "805b4d3d-5f8c-47d6-82dd-e37442627472",
    "version": "1.0",
    "type": "cancelOrder",
    "createdAt": "2021-03-05T06:23:18Z",
    "senderContractorKpp": "770501001",
    "senderContractorInn": "7705739450",
    "transportOperatorInn": "7726630679",
    "receiverContractorInn": "7726630679",
    "receiverContractorKpp": "771401001",
    "senderSystemCode": "sender_prod_msk1",
    "manualSend": false,
    "isTest": false
  },
  "body": {
    "externalId": "11223344",
    "externalNumber": "11223344"
  }
}
]
```

</details>


## 5. Пример ответа

<details>
<summary>Принято к обработке</summary>

```json5
202 	
Сообщения приняты в обработку
```

</details>

<details>
<summary>Ошибка валидации обязательных полей</summary>

```json5
{
    "messageUuid": "%header.uuid%",  
    "errorCode": "field_is_absent",
    "errorMessage": "Не заполнены обязательные поля секции header: uuid, type"
}
```
</details>

<details>
<summary>Ошибка валидации формата данных</summary>

```json5
{
    "messageUuid": "%header.uuid%",  
    "errorCode": "incorrect_format_of_field",
    "errorMessage": "Некорректный формат полей секции header: uuid, type"
}
```
</details>