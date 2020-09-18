# eRx HL7 FHIR Lite API v1.0

Този документ описва интеграционния подход и начин на работа на eRx HL7 FHIR Lite REST API v1.0

### Общо описание за работата с REST API адаптера

Всички отговори на заявки към REST API са пакетирани в обект носещ основна информация за резултата от изпълнението:

```
{
  "Result": any,
  "Success": boolean,
  "Errors" [{Path: string, Message: string, Key: string, Id: string}]
}
``` 
* в атрибута `success` е закодиран крайния резултат на операцията под формата на булева стойност, т.е. дали операцията е успешна или не. 
**Стойността на този атрибут трябва да се разбира и като грешка в бизнес логиката на API.** Например при търсене на рецепта
резултата може да бъде `success: false` в случаите, когато липсват резултатите по зададения идентификатор. При този случай
в резултатния масив на грешките ще присъства една грешка с описание за липсваща рецепта със зададеното id.   
* атрибута `Result` е контейнер за данните върнати от изпълнението на заявката.
* при наличие на грешки в атрибута errors се попълва масив от грешки касаещи изпълнението на заявката. Всяка грешка е под формата на обект
притежаващ атрибутите `Path`, `Message`, `Key` и `Id`. `Path` е незадължителен атрибут указващ пътя на променливата, която е с невалидна стойност. 
Всички валидационни грешки притежават такъв атрибут. Понякога обаче този атрибут може да бъде празен, тъй като грешката на касае и не реферира
конкретни данни от заявката. Атрибута `Message` е информативно поле описващо причината за настъпване на грешката. Езикът на който се връща
отговора зависи от стойността на хидъра `Accept-Language`, който се изпраща в заявката. Примерни стойности могат да бъдат `Accept-Language: bg-BG` 
или `Accept-Language: en-EN`. За сега API поддържа само тези два езика. Атрибута `Id` указва конкретната грешка, която е настъпила
и може да се използва като референция при обследването.

Атрибута `Success` e в резултатната структура и се показва винаги. Атрибута `Errors` също е задължитеиелн и при липсвата на грешки показва празен масив. 

##### Примерен резултат на успешна заявката:
```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": {
    "User": {
      "Token": "<admin user token>"
    }
  },
  "Success": true,
  "Errors" []
}
```

##### Пример резултат наа неуспешна заявка:

```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": null,
  "Success": false,
  "Errors" [
    {
      "Message" "Грешно потребителко име или парола",
      "Key" "{user.login.failed}"
    }
  ]
}

```

##### Други грешки на автентикационната услуга - некасаещи бизнес логиката на системата
* 404 NotFount - грешен URL на заявката. Резултата е HTML съдържание
* 401 Unauthorized - Среща се при автентикираните заявки - означава, че липсват креденции в заяквата.
* 500 Internal Server Error - грешки на софтуера, които излизат извън видимостта на софтуера - например недостатъчни количесто памет и др. 

### Описание на процеса за регистрация на потребител

Регистрацията на краен потребител (Потребител на аптечния софтуер) преминава през две основни стъпки.

1. Първа стъпка - вземане на автентикационен токен за администратор. 
2. Втора стъпка - Създаване на краен потребител с помощта на автентикационния токен на администратора. 

Административния потребител се създава от администраторите на регистъра. 

*Заявките за вход в системата, притежават допълнителен автентикационен хидър (с Basic автентикация) определящ клиентския софтуер, от който се изпращат креденциите.*

Процесът за това е следният:
1. Вземане на автентикационен токен за администратор. Изпълнение на POST заявка със следните данни:
```
Request method:	POST
Request URI:	https://erx2.e-health.bg/auth/login
Proxy:			<none>
Request params:	<none>
Query params:	<none>
Form params:	<none>
Path params:	<none>
Headers:		Accept=*/*
                Accept-Language=bg-BG
				Content-Type=application/json; charset=UTF-8
                Authorization: Basic <omitted> \
Cookies:		<none>
Multiparts:		<none>
Body:
{
    "User": {
        "Data": [
            {
                "Username": "niset",
                "Credentials": [
                    {
                        "Type": "password",
                        "Value": "<omitted>"
                    }
                ]
            }
        ],
        "Action": "LOGIN"
    }
}
```
Резултата от заявката се връща в 200 ОК дори и ако съществуват грешки в бизнес логиката на процеса. 
```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": {
    "User": {
      "Token": "<admin user token>"
    }
  },
  "Success": true,
  "Errors" []
}
```
Обработката на грешки при тази заявка се прави само в случая, че в резултата присъства атрибута `success: false`. 

3. Изполазвайте полученият токен за да изпълните нова заявка за създаване на краен потреботел:
```
Request method: POST
Request URI:    https://erx2.e-health.bg/auth/register
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Headers:                Authorization=Bearer <Административния токен получен от предната заявка>
                                Accept-Language=bg-BG
                                Accept=*/*
                                Content-Type=application/json; charset=UTF-8
Cookies:                <none>
Multiparts:             <none>
Body:
{
    "Auth": {
        "Data": [
            {
                "Username": "test77",
                "Enabled": true,
                "FirstName": "John",
                "LastName": "Doe",
                "Email": "user77@e-health.bg",
                "Credentials": [
                    {
                        "Type": "password",
                        "Value": "<omitted>"
                    }
                ]
            }
        ],
        "Action": "CREATE",
        "MedicRegRq": {
            "DhId": "2201928818",
            "OrganizationName": "ДКЦ ТЕСТ",
            "NhifBranch": "02"
        }
    }
}
```

Резултата от тази заявка ще бъде винаги `200 OK` като в отговора ще се съдържа информация за направената регистрация: 
Пример за успешна регистрация:
```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": {
    "Id": 88,
    "FirstName": "John",
    "LastName": "Doe",
    "Enabled": true,
    "Username": "test77",
    "Email": "user77@e-health.bg",
    "EmailVerified": false,
    "Acknowledged": false,
    "Authorities": [
      {
        "Id": 89,
        "Authority": "ROLE_PHARMACIST",
        "Client": "1"
      }
    ],
    "Properties": [
      {
        "Id": 90,
        "Key": "roleId",
        "Value": "PractitionerRole/18/_history/1"
      }
    ],
    "Provider": "local"
  },
  "Success": true,
  "Errors": []
}
```
Пример резултат за грешна регистрация с мейл, който е дублиран в системата:

```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Success": false,
  "Errors": [
    {
      "Message": "Key (email)=(user73@e-health.bg) already exists.",
      "Id": "dbb1a3f3-6a0e-4ee6-80fd-41c33b5b1f7d"
    }
  ]
}
```

#### Вход в системата за краен потребител

Входа в системата става чрез: 
```
Request method: POST
Request URI:    https://erx2.e-health.bg/auth/login
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Headers:        Accept-Language=bg-BG
                Accept=*/*
                Content-Type=application/json; charset=UTF-8
                Authorization: Basic <omitted> \

Cookies:                <none>
Multiparts:             <none>
Body:

{
    "User": {
        "Data": [
            {
                "Username": "test77",
                "Credentials": [
                    {
                        "Type": "password",
                        "Value": "<omitted>"
                    }
                ]
            }
        ]
    }
}
```
*Отново при изпълнениет на тази заявка се изисква добавяне на Basic автентикация, аналогично на предходната при вземане на токен за администратор*

При изпълнението заявката ще получите отговор 200 ОК, като в резултата ще получите JSON с токен нужен за изпълнението 
на последващите заявки към микроуслугите обслужващи основните бизнес функции на регистъра:
```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": {
    "User": {
      "Token": "<token content>"
    }
  },
  "Success": true,
  "Errors" []
}
```

Запазете текущо генерирания токен за последващите заявки, носеща в себе си информация за потребителя, валидността, достъпа до услугите,
както и допълнителна информация нужна за авторизацията. Той е стрингов низ който е състевен от три части заделени с точка.
При base64 декодитане на втората част се получава JSON обект включващ в себе си информация за времето в което токена е валиден.
Повече информация за характеристиките на данните закодирани в този артефакт можете да намерите тук: https://tools.ietf.org/html/rfc7519
 
#### Заявка "Whoami" - "Кой съм аз"

За да валидирате текущо издадения токен можете да използвате следната заявка:

```
curl --location --request GET 'https://erx2.e-health.bg/user/me' \
--header 'Authorization: Bearer <access token>'
```
Ако токена е валиден, то вие ще получите следният отговор 200 ОК със следното съдържание:

```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": {
    "Id": 88,
    "FirstName": "John",
    "LastName": "Doe",
    "Enabled": true,
    "Username": "test77",
    "Email": "user77@e-health.bg",
    "EmailVerified": false,
    "Acknowledged": false,
    "Authorities": [
      {
        "Id": 89,
        "Authority": "ROLE_MEDIC",
        "Client": "1"
      }
    ],
    "Properties": [
      {
        "Id": 90,
        "Key": "roleId",
        "Value": "PractitionerRole/18/_history/1"
      }
    ],
    "Provider": "local"
  },
  "Success": true,
  "Errors": []
}
```

#### Обработка на грешки в микроуслугата за вход и регистрация

Отговорът на всички грешни заявки е 200 ОК (с изклчение на тези с грешно URL), но със съдържание в което присъства атрибута "errors".
Например грешката при вход в системата е:
```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
  "Result": null,
  "Success": false,
  "Errors" [
    {
      "Message" "Грешно потребителко име или парола",
      "Key" "{user.login.failed}",
      "Id": "9b5924f8-b236-11ea-a311-a3651306a48e"
    }
  ]
}

```

Грешките при регистрация на потребител могат да бъдат:
* Грешка 401 - неоторизиран достъп
* Грешка 404 - получава се само в случаите че URL-то към услугите е грешно. Резултата е HTML съдържание.
* Грешка 500 - случайна грешка при в софтуера.

При резултатите от всички грешки се връща информативно поле Message което може да бъде директно изпратено към крайния потребител.

```
{
  "wrong credentials": "Грешно потребителско име или парола",
  "Wrong client authentication": "Грешни клиентски автентикационни атрибути",
  "invalid request format": "Грешен формат на заявката за вход",
  "Username attribute is mandatory": "Атрибута Username е задължителен",
  "data integrity error": "Грешка в интегриртета на данните. Моля обърнете се към администратора",
  "PractitionerRole is not saved": "Данните ви не са записани във FHIR. Моля обърнете се към администратора на системата",
  "Bad credentials": "Грешно потребителско име или парола",
  "Unauthorized": "Грешни автентикационни параметри",
  "Internal Server Error": "Възникна случайна грешка. Моля обърнете се към администратора на системата"
}
```

#### Микроуслуга за създаване и промяна на рецепти в системата на регистъра
С помощта на наличния токен се изпълняват последващите заявки имащи за цел създаване на рецепти в системата на регистъра. 
Заявката за създаване и промяна на рецепта става с еднотипна по вид структура. Промяната на рецепта всъщност създава нова 
версия на предходната като запазва старата и версия за случаите когато тя се изисква. Заявката представлява
валидна HL7 FHIR структура в олекотен вид. Целта на това "олекотяване" е да се услени интеграционния процес и адаптацията
на системата. Примерна заявка за рецепта представлява:

```
curl --location --request POST 'https://erx2.e-health.bg/fhirlite/prescription?seed=3014' \
--header 'Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOjEyNCwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJyb2xlcyI6W3sicm9sZSI6IlJPTEVfTUVESUMiLCJjbGllbnQiOiJtZWRpYyJ9XSwiaXNzIjoiaHR0cHM6Ly9lbG1lZGlrby5jb20iLCJnaXZlbl9uYW1lIjpudWxsLCJjbGllbnRfaWQiOiJtZWRpYyIsInBpY3R1cmUiOm51bGwsInNjb3BlIjoiKi8qLioiLCJwaG9uZV9udW1iZXIiOm51bGwsImV4cCI6MTYwMDM3MjM4MCwiZmFtaWx5X25hbWUiOm51bGwsImVtYWlsIjpudWxsLCJ1c2VybmFtZSI6Im1lZGljNSJ9.2mIruwdqt24M0HytcJKaqUTMOlz1N5DKg2uS9VuQIoBuzEWM2LOI6-O0uqG7kGcaU1CivCxezz7zHe-4luvLKQ' \
--header 'Content-Type: application/json' \
--data-raw '{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "A0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "category": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                "code": "A"
              }
            ]
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 1,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "B0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "category": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                "code": "A"
              }
            ]
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 2,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "C0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "category": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                "code": "A"
              }
            ]
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 2,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "Practitioner",
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/uin",
            "value": "2300013314"
          }
        ],
        "name": [
          {
            "use": "official",
            "family": "Поляков",
            "given": [
              "Иван",
              "Акимов"
            ],
            "prefix": [
              "д-р"
            ]
          }
        ],
        "telecom": [
          {
            "system": "phone",
            "value": "088888880",
            "use": "mobile"
          },
          {
            "system": "email",
            "value": "ipolyakov@e-health.bg",
            "use": "work"
          }
        ],
        "gender": "male"
      }
    },
    {
      "resource": {
        "resourceType": "PractitionerRole",
        "specialty": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/doctor-speciality-nhif",
                "code": "00",
                "display": "Общопрактикуващ лекар"
              }
            ]
          }
        ]
      }
    },
    {
      "resource": {
        "resourceType": "Patient",
        "extension": [
          {
            "url": "http://terminology.e-health.bg.com/Extension/patient-maternity-flag",
            "valueBoolean": false
          },
          {
            "url": "http://terminology.e-health.bg.com/Extension/patient-pregnancy-flag",
            "valueBoolean": false
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/nnbgr",
            "value": "<omitted>"
          }
        ],
        "birthDate": "1972-03-12T00:00:00.000Z",
        "gender": "male",
        "name": [
          {
            "family": "ТРАЕВ",
            "given": [
              "КОНСТАНТИН",
              "ТРАЙЧЕВ"
            ]
          }
        ]
      }
    },
    {
      "resource": {
        "resourceType": "Encounter",
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/enc-id",
            "value": "006665"
          }
        ],
        "period": {
          "start": "2020-09-01T00:00:00.000Z",
          "end": "2020-09-01T00:10:00.000Z"
        }
      }
    }
  ]
}
'
```
Тялото на заявката е JSON представляващ FHIR структурата Bundle (https://www.hl7.org/fhir/bundle.html).
Това е специфична единица която обединява в себе си множество FHIR ресурси и има за цел изълнение на няколко заявки
в обща атомарна транзакция. Това означава, че при грешка при някои от операциите описани в Bundle ресурса, изъпълнението
на всички останали операции спира. Всяка операция e описана под атрибута _entry_ и представлява описание на един от
ресурсите засегнати в процеса по издавана на рецепта. 
Всеки притежава атрибута _resourceType_. Ресурсните типове на FHIR използвани при издаването на рецептата са следните:

* MedicationRequest - аналогия на един ред от рецепта включващ една лекарство и дозировката към него
* Practitioner - Това е ресурса описащ информация за лекаря издал рецептата
* PractitionerRole - Роля на лекаря с неговата конкретна специалност принадлежаща към лечебното заведение в което се издава рецептата
* Patient - Репрезентация на пациента със всички необходими данни за издаването на рецептата
* Encounter - Репрезентация на прегледа като събитие с идентифиакатор и време на протичане 

Има и още няколко ресурса които са свърани с процеса, но адаптера за FHIR API на регистъра прави използването им прозрачно. 
Това са:

* Organization - Репрезентация на организацията издаваща рецептата. Създава се по време на регистрация на потребителски акаунт и се добавя като референция при издаване на рецептата.
* Location - Репрезентация на локацията на конкретната организация издаваща рецептата. Създава се автоматично по време на регистрация на потребителски акаунт и се добавя като референция при издаване на рецептата.

Кода на лекарствения продукт се специфицира посредством:
```
...
"medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF034",
              "display": "ZYRTEC FILM COATED TABLETS 10MG 10"
            }
          ]
        },
...
```
Атрибута `system` обозначава системата по която се кодира, `code` предствлява кода на лекарството.
Системата е във формат на URL който може да носи в себе си информация за издателя не кодиращата система, версия, както и други атрибути. 

Понастоящем са позволени две системи за кодиране:
 - http://terminology.e-health.bg/CodeSystem/mc-nhif - обозначава лекарствените продукти от позитивния лекарствен списък
 - http://sathealth.com/cs/pt_code - обозначава лекарствените продукти от списъкут изготвен от SatHealth. Пълна спецификация на този списък можете да намерите тук: http://pharmacy.e-health.bg/ml
 
Кодовете от системата на SatHealth започват с префикса PT. В тяхната номенклатура са кодирани, лекарствата от позитивния лекарствен списък, OTC и хранителни добавки.

**При издаване на рецепта по НЗОК е важно да се използва кодовата система http://terminology.e-health.bg/CodeSystem/mc-nhif, дори и ако лекарството
присъства и в кодовата система на SatHealth.**
 
##### Категории рецепти
Категорията на рецептат се специфицира в атрибута category.coding[0].code и може да притежава една от следните стойности:
* W - бяла рецептурна бланка
* G - зелена рецептурна бланка
* Y - жълта рецептурна бланка
* N - рецепта по НЗОК - бланка №5
* A - рецепта по НЗОК - бланка №5а
* B - рецепта по НЗОК - бланка №5б
* C - рецепта по НЗОК - бланка №5в

Ако рецептата е с категории започваща с А, B, C или N, то кодът на лекарството, трябва да бъде задължително от позитивния лекарствен списък по НЗОК.
```
...
    "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "SF083",
              "display": "FLAREX EYE DROPS 0.1% 5ML 1"
            }
          ]
        }
...
``` 
При бялата рецепта обаче в списка с лекарства може да присъстват и такива които са извън позитивния лекарствен списък на НЗОК. 
В този случай кодировката на лекарството е по друга номенклатура специфициран чрез:

```
...
    "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://sathealth.com/cs/pt_code",
              "code": "PT004470",
              "display": "ZYRTEC FILM COATED TABLETS 10MG 10"
            }
          ]
        }
...
``` 
Въпросната номенклатура включва в себе си както лекарствата от позитивния лекарствен списък, така и тези които могат
да бъдат изписвани без лекарско предписание, хранителни добавки и др. Достъпът на номенклатурата е публичерн и свободен.
Повече информация за това можете да намерите тук: https://pharmacy.e-health.bg/ml

#### Разчитане на резултата от изпълнение на заявката 
Резултата от изпълнението на заяквата при успех е следният:
```
{
    "result": {
        "t1": {
            "id": "5ce0a539-2361-4730-af5e-65346aed83b1",
            "resource-type": "Bundle",
            "type": "transaction-response",
            "entry": [
                {
                    "response": {
                        "status": "200 OK",
                        "location": "Patient/252/_history/1",
                        "etag": "1"
                    }
                },
                {
                    "response": {
                        "status": "201 Created",
                        "location": "MedicationRequest/300/_history/1",
                        "etag": "1"
                    }
                },
                {
                    "response": {
                        "status": "201 Created",
                        "location": "MedicationRequest/301/_history/1",
                        "etag": "1"
                    }
                },
                {
                    "response": {
                        "status": "201 Created",
                        "location": "MedicationRequest/302/_history/1",
                        "etag": "1"
                    }
                },
                {
                    "response": {
                        "status": "200 OK",
                        "location": "Practitioner/3/_history/1",
                        "etag": "1"
                    }
                },
                {
                    "response": {
                        "status": "200 OK",
                        "location": "PractitionerRole/292/_history/1",
                        "etag": "1"
                    }
                },
                {
                    "response": {
                        "status": "201 Created",
                        "location": "Encounter/303/_history/1",
                        "etag": "1"
                    }
                }
            ]
        },
        "t2": {
            "resource-type": "Bundle",
            "type": "transaction",
            "entry": [
                {
                    "resource": {
                        "resourceType": "Patient",
                        "extension": [
                            {
                                "url": "http://terminology.e-health.bg.com/Extension/patient-maternity-flag",
                                "valueBoolean": false
                            },
                            {
                                "url": "http://terminology.e-health.bg.com/Extension/patient-pregnancy-flag",
                                "valueBoolean": false
                            }
                        ],
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/nnbgr",
                                "value": "<omitted>"
                            }
                        ],
                        "name": [
                            {
                                "family": "ТРАЕВ",
                                "given": [
                                    "КОНСТАНТИН",
                                    "ТРАЙЧЕВ"
                                ]
                            }
                        ],
                        "gender": "male",
                        "birthDate": "1972-03-12T00:00:00.000Z"
                    },
                    "request": {
                        "method": "POST",
                        "url": "Patient"
                    }
                },
                {
                    "resource": {
                        "resourceType": "MedicationRequest",
                        "extension": [
                            {
                                "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
                                "valueString": "A0"
                            }
                        ],
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/booklet-id",
                                "value": "1520002"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/barcode-id",
                                "value": "ERXE-20200917-124-3014-A"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/mr-id",
                                "value": "ERXE-20200917-124-3014-A0"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/internal",
                                "value": "3014"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/dh-id",
                                "value": "1234567897"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/erx-users",
                                "value": "medic5"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/software-clients",
                                "value": "medic"
                            }
                        ],
                        "status": "active",
                        "intent": "proposal",
                        "category": [
                            {
                                "coding": [
                                    {
                                        "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                                        "code": "A"
                                    }
                                ]
                            }
                        ],
                        "medicationCodeableConcept": {
                            "coding": [
                                {
                                    "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
                                    "code": "RF127",
                                    "display": "Foster"
                                }
                            ]
                        },
                        "subject": {
                            "reference": "urn:uuid:139cc2c4-d4af-436c-91d7-68434ad215d7",
                            "display": "КОНСТАНТИН ТРАЙЧЕВ ТРАЕВ"
                        },
                        "encounter": {
                            "reference": "urn:uuid:07abb08c-bc31-466b-8042-bf4de71aa1b1"
                        },
                        "authoredOn": "2020-09-01T00:00:00.000Z",
                        "requester": {
                            "reference": "urn:uuid:c7066a7c-064f-443d-8211-b763d8c47312",
                            "display": "д-р Иван Акимов Поляков"
                        },
                        "reasonCode": [
                            {
                                "coding": [
                                    {
                                        "system": "http://hl7.org/fhir/sid/icd-10",
                                        "code": "J44.8"
                                    }
                                ]
                            }
                        ],
                        "groupIdentifier": {
                            "system": "http://erx.e-health.bg/ns/barcode-id",
                            "value": "ERXE-20200917-124-3014-A"
                        },
                        "dosageInstruction": [
                            {
                                "text": "SIG",
                                "doseAndRate": [
                                    {
                                        "doseQuantity": {
                                            "value": 3,
                                            "unit": "units"
                                        },
                                        "rateQuantity": {
                                            "value": 2,
                                            "unit": "daily"
                                        }
                                    }
                                ]
                            }
                        ],
                        "dispenseRequest": {
                            "dispenseInterval": {
                                "value": 30,
                                "unit": "days"
                            },
                            "quantity": {
                                "value": 1,
                                "unit": "pack"
                            }
                        },
                        "substitution": {
                            "allowedBoolean": false
                        }
                    },
                    "request": {
                        "method": "PUT",
                        "url": "MedicationRequest?identifier=http://erx.e-health.bg/ns/mr-id|ERXE-20200917-124-3014-A0&identifier=http://erx.e-health.bg/ns/erx-users|medic5"
                    }
                },
                {
                    "resource": {
                        "resourceType": "MedicationRequest",
                        "extension": [
                            {
                                "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
                                "valueString": "B0"
                            }
                        ],
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/booklet-id",
                                "value": "1520002"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/barcode-id",
                                "value": "ERXE-20200917-124-3014-B"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/mr-id",
                                "value": "ERXE-20200917-124-3014-B0"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/internal",
                                "value": "3014"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/dh-id",
                                "value": "1234567897"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/erx-users",
                                "value": "medic5"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/software-clients",
                                "value": "medic"
                            }
                        ],
                        "status": "active",
                        "intent": "proposal",
                        "category": [
                            {
                                "coding": [
                                    {
                                        "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                                        "code": "A"
                                    }
                                ]
                            }
                        ],
                        "medicationCodeableConcept": {
                            "coding": [
                                {
                                    "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
                                    "code": "RF127",
                                    "display": "Foster"
                                }
                            ]
                        },
                        "subject": {
                            "reference": "urn:uuid:139cc2c4-d4af-436c-91d7-68434ad215d7",
                            "display": "КОНСТАНТИН ТРАЙЧЕВ ТРАЕВ"
                        },
                        "encounter": {
                            "reference": "urn:uuid:07abb08c-bc31-466b-8042-bf4de71aa1b1"
                        },
                        "authoredOn": "2020-09-01T00:00:00.000Z",
                        "requester": {
                            "reference": "urn:uuid:c7066a7c-064f-443d-8211-b763d8c47312",
                            "display": "д-р Иван Акимов Поляков"
                        },
                        "reasonCode": [
                            {
                                "coding": [
                                    {
                                        "system": "http://hl7.org/fhir/sid/icd-10",
                                        "code": "J44.8"
                                    }
                                ]
                            }
                        ],
                        "groupIdentifier": {
                            "system": "http://erx.e-health.bg/ns/barcode-id",
                            "value": "ERXE-20200917-124-3014-B"
                        },
                        "dosageInstruction": [
                            {
                                "text": "SIG",
                                "doseAndRate": [
                                    {
                                        "doseQuantity": {
                                            "value": 3,
                                            "unit": "units"
                                        },
                                        "rateQuantity": {
                                            "value": 2,
                                            "unit": "daily"
                                        }
                                    }
                                ]
                            }
                        ],
                        "dispenseRequest": {
                            "dispenseInterval": {
                                "value": 30,
                                "unit": "days"
                            },
                            "quantity": {
                                "value": 2,
                                "unit": "pack"
                            }
                        },
                        "substitution": {
                            "allowedBoolean": false
                        }
                    },
                    "request": {
                        "method": "PUT",
                        "url": "MedicationRequest?identifier=http://erx.e-health.bg/ns/mr-id|ERXE-20200917-124-3014-B0&identifier=http://erx.e-health.bg/ns/erx-users|medic5"
                    }
                },
                {
                    "resource": {
                        "resourceType": "MedicationRequest",
                        "extension": [
                            {
                                "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
                                "valueString": "C0"
                            }
                        ],
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/booklet-id",
                                "value": "1520002"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/barcode-id",
                                "value": "ERXE-20200917-124-3014-C"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/mr-id",
                                "value": "ERXE-20200917-124-3014-C0"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/internal",
                                "value": "3014"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/dh-id",
                                "value": "1234567897"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/erx-users",
                                "value": "medic5"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/software-clients",
                                "value": "medic"
                            }
                        ],
                        "status": "active",
                        "intent": "proposal",
                        "category": [
                            {
                                "coding": [
                                    {
                                        "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                                        "code": "A"
                                    }
                                ]
                            }
                        ],
                        "medicationCodeableConcept": {
                            "coding": [
                                {
                                    "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
                                    "code": "RF127",
                                    "display": "Foster"
                                }
                            ]
                        },
                        "subject": {
                            "reference": "urn:uuid:139cc2c4-d4af-436c-91d7-68434ad215d7",
                            "display": "КОНСТАНТИН ТРАЙЧЕВ ТРАЕВ"
                        },
                        "encounter": {
                            "reference": "urn:uuid:07abb08c-bc31-466b-8042-bf4de71aa1b1"
                        },
                        "authoredOn": "2020-09-01T00:00:00.000Z",
                        "requester": {
                            "reference": "urn:uuid:c7066a7c-064f-443d-8211-b763d8c47312",
                            "display": "д-р Иван Акимов Поляков"
                        },
                        "reasonCode": [
                            {
                                "coding": [
                                    {
                                        "system": "http://hl7.org/fhir/sid/icd-10",
                                        "code": "J44.8"
                                    }
                                ]
                            }
                        ],
                        "groupIdentifier": {
                            "system": "http://erx.e-health.bg/ns/barcode-id",
                            "value": "ERXE-20200917-124-3014-C"
                        },
                        "dosageInstruction": [
                            {
                                "text": "SIG",
                                "doseAndRate": [
                                    {
                                        "doseQuantity": {
                                            "value": 3,
                                            "unit": "units"
                                        },
                                        "rateQuantity": {
                                            "value": 2,
                                            "unit": "daily"
                                        }
                                    }
                                ]
                            }
                        ],
                        "dispenseRequest": {
                            "dispenseInterval": {
                                "value": 30,
                                "unit": "days"
                            },
                            "quantity": {
                                "value": 2,
                                "unit": "pack"
                            }
                        },
                        "substitution": {
                            "allowedBoolean": false
                        }
                    },
                    "request": {
                        "method": "PUT",
                        "url": "MedicationRequest?identifier=http://erx.e-health.bg/ns/mr-id|ERXE-20200917-124-3014-C0&identifier=http://erx.e-health.bg/ns/erx-users|medic5"
                    }
                },
                {
                    "resource": {
                        "resourceType": "Practitioner",
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/uin",
                                "value": "2300013314"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/dh-id",
                                "value": "1234567897"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/erx-users",
                                "value": "medic5"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/software-clients",
                                "value": "medic"
                            }
                        ],
                        "name": [
                            {
                                "use": "official",
                                "family": "Поляков",
                                "given": [
                                    "Иван",
                                    "Акимов"
                                ],
                                "prefix": [
                                    "д-р"
                                ]
                            }
                        ],
                        "telecom": [
                            {
                                "system": "phone",
                                "value": "088888880",
                                "use": "mobile"
                            },
                            {
                                "system": "email",
                                "value": "ipolyakov@e-health.bg",
                                "use": "work"
                            }
                        ],
                        "gender": "male"
                    },
                    "request": {
                        "method": "POST",
                        "url": "Practitioner"
                    }
                },
                {
                    "resource": {
                        "resourceType": "PractitionerRole",
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/uin",
                                "value": "2300013314"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/dh-id",
                                "value": "1234567897"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/erx-users",
                                "value": "medic5"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/software-clients",
                                "value": "medic"
                            }
                        ],
                        "practitioner": {
                            "reference": "urn:uuid:752de6ea-79b6-43c3-bb21-88639f8992fd"
                        },
                        "organization": {
                            "reference": "Organization/285",
                            "display": "Test Clinic"
                        },
                        "specialty": [
                            {
                                "coding": [
                                    {
                                        "system": "http://terminology.e-health.bg/CodeSystem/doctor-speciality-nhif",
                                        "code": "00",
                                        "display": "Общопрактикуващ лекар"
                                    }
                                ]
                            }
                        ],
                        "location": [
                            {
                                "reference": "Location/286",
                                "display": "Test Clinic"
                            }
                        ],
                        "telecom": [
                            {
                                "system": "phone",
                                "value": "088888880",
                                "use": "mobile"
                            },
                            {
                                "system": "email",
                                "value": "ipolyakov@e-health.bg",
                                "use": "work"
                            }
                        ]
                    },
                    "request": {
                        "method": "PUT",
                        "url": "PractitionerRole?identifier=http://erx.e-health.bg/ns/uin|2300013314&identifier=http://erx.e-health.bg/ns/erx-users|medic5"
                    }
                },
                {
                    "resource": {
                        "resourceType": "Encounter",
                        "identifier": [
                            {
                                "system": "http://erx.e-health.bg/ns/enc-id",
                                "value": "006665"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/dh-id",
                                "value": "1234567897"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/erx-users",
                                "value": "medic5"
                            },
                            {
                                "system": "http://erx.e-health.bg/ns/software-clients",
                                "value": "medic"
                            }
                        ],
                        "participant": [
                            {
                                "individual": {
                                    "reference": "urn:uuid:c7066a7c-064f-443d-8211-b763d8c47312"
                                }
                            }
                        ],
                        "period": {
                            "start": "2020-09-01T00:00:00.000Z",
                            "end": "2020-09-01T00:10:00.000Z"
                        },
                        "location": [
                            {
                                "location": {
                                    "reference": "Location/286",
                                    "display": "Test Clinic"
                                }
                            }
                        ]
                    },
                    "request": {
                        "method": "PUT",
                        "url": "Encounter?identifier=http://erx.e-health.bg/ns/enc-id|006665&identifier=http://erx.e-health.bg/ns/erx-users|medic5&identifier=http://erx.e-health.bg/ns/software-clients|medic"
                    }
                }
            ]
        },
        "t3": [
            "ERXE-20200917-124-3014-A",
            "ERXE-20200917-124-3014-B",
            "ERXE-20200917-124-3014-C"
        ]
    },
    "success": true,
    "errors": []
}
```
Отново структурата е обвита в т.нар ResultWrapper чийто флаг success носи информация за успеха на заяквата. 
При успешна заяква в резултата присъстват три характеристики (Т1, Т2 и Т3). 
* Т1 е резултата от записа 
* Т2 е реалната FHIR заявка изпълнена към сървъра. Олекотената FHIR заявка се трансформира и записва във FHIR сървъра.
* Т3 е масив от баркодове на записаната рецепта. 

Структурата на Т1 атрибура носи в себе си информация за изпълнената операция върху конкретния ресурс:
```
...
"Id": "aecad7a6-1e36-41e7-a757-71a99ef47dd5",
            "ResourceType": "Bundle",
            "Type": "transaction-response",
            "Entry": [
                {
                    "Response": {
                        "Status": "201 Created",
                        "Location": "MedicationRequest/68/_history/1",
                        "Etag": "2"
                    }
                },
...

```
При статус *201 Created* системата индикира че исканият запис е създаден успешно. Конкретната заявка за създадения 
ресурс се намира в Т2 като позицията на ресурса в състава на бъндъла остава константна. 
При промяна на ресурса (например промяна на дозировката) се изпраща същата заявка с идентични идентификатори на рецепта.
В този случаи данните за рецептата генерират нова версия и врезултатния масив се връща Status: 200 ОК , като версията 
на ресурсите се инкрементира:

```
...
"Id": "aecad7a6-1e36-41e7-a757-71a99ef47dd5",
            "ResourceType": "Bundle",
            "Type": "transaction-response",
            "Entry": [
                {
                    "Response": {
                        "Status": "200 ОК",
                        "Location": "MedicationRequest/68/_history/2",
                        "Etag": "2"
                    }
                },
...

``` 

*Важно: query параметъра seed определя уникалността на рецептата. Той е вътрешния идентификатор на рецептата в системата на издателя. Рецептата може да бъде променена 
ако се изпълни същата заявка с новите данни и същия seed, но само в рамките на деня в който е изписана*

#### Appendix #1 Услуга за генериране на баркод

Примерна заявка за генериране на баркод по идентификатор на рецептата.
https://erx2.e-health.bg/print/barcode/code128?code=ERXE-20200917-124-3015-A&scale=3

<img src="https://erx2.e-health.bg/print/barcode/code128?code=ERXE-20200917-124-3015-A&scale=3"></img>

##### Генериране на баркод
При изпълнението на заявката за запис на рецепта трябва да бъде подаден и т.нар. seed, който е стрингове поле без интервали определящо идентификатора на рецептата в системата на издателя и.
Примерна заявка представлява POST https://stgerx2.e-health.bg/fhirlite/prescription?seed=1100102

Примерен баркод генериран по този подход е: ERXE-20200627-60-1100102

Полето seed трябва да бъде уникално за всяка издадена рецепта, това е опция която се съблюдава от издателя на рецептата. 

Регистъра от своя страна генерира други уникални баркодове, които се връщат в отговора на в полето T3. Те са генерирани на базата на
вътрешни алгоритми съблюдаващи уникалността на всяка заявка. Имайте в предвид че всяка промяна на вече изписана рецепта генерира нов уникален баркод
в полето Т3. Баркода генериран по метода използващ seed е статичен и уникалността му зависи от уникалността на полето seed.
Баркода генериран чрез seed се записва в MedicationRequest.identifier и може да бъде открит в отговора на заявката.  
Структура на баркода генериран по този принцип:

 * "ERXE"
 * текущата дата във формат yyyyMMdd
 * полето sub на автентикационния токен, което преставлява Result.Id. Тази стойност е постоянна и се генерира по време на регистрация. Може да се вземе и от резултата на заявката за регистрация, която изглежда по следния начин:{
  "Result" : {
    "Id" : 145,
    "Enabled" : true,
    "Username" : "mediksoft_23",
    "EmailVerified" : false, .... в случая това ид е 145
 * Идентификатора на рецептата който се намира в базата на издателя. Ако рецептата записана в таблица на базата има идентификатор = 1000 то това е id-то което се изпраща в seed. 
 * Примерен баркод: ERXE-20200627-145-1000.
 
При генерирането на баркод за рецепта по НЗОК бланка 5А(Тройна рецепта) се издава отделен баркод за всеки отрязък нарецептата, тъй като
при изписването на лекарствата в аптечния софтуер се съблюдава конкретния отрязък. По тази причина към баркода
автоматично се добавя и съответния индекс на отрязъка - A, B или C разделен с тире. Това касае само принтирането на рецептата от страна на издателя, т.е. не променя формата на заявката.
Издателя трябва да принтира отделен баркод на всеки отрязък който има следния формат:

ERXE-20200627-145-1000-A - за отрязък А, ERXE-20200627-145-1000-B за отрязък B и ERXE-20200627-145-1000-C за отрязък C.

#### Услуга за принтиране на рецепти
ToDo:

#### Регистриране за нотификация при промяна на рецепта
ToDo:

#### Пример за пълен процес по регистрация, вход и издаване на рецепти
##### Вход с административен потребител съответстващ на издателя на медицинския софтуер.
```
curl --location --request POST 'https://stgerx2.e-health.bg/auth/login' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic Base64(<clientUsername>:<clientPassword>)' \
--data-raw '{
    "User": {
        "Data": [
            {
                "Username": "medic",
                "Credentials": [
                    {
                        "Type": "password",
                        "Value": "<the password>"
                    }
                ]
            }
        ]
    }
}'
```
Атрибутите, чрез които се генерира Basic автентикацията се издават от администратора на регистъра и са различни от тези за вход (User.Data.Username). Те определят клиенсткото приложение.
Атрибутите за потребителско име и парола В Json body се създават от администратора на регистъра и определят съответния административен акаунт който използва въпросния клиетски софтуер. 
Може да има повече от един администраторски акаунт който използва едни и същи Basic-Auth атрибути.
 
Очакван резултат:
```
{
    "Result": {
        "User": {
            "Token": "<Bearer token>"
        }
    },
    "Success": true,
    "Errors": []
}
```
 
##### Регистрация на практика.
Регистрация на практика с данните за нея, като се използва <Bearer Token> получен от предходната заявка;

```
curl --location --request POST 'https://stgerx2.e-health.bg/auth/register' \
--header 'Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOjU3LCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInJvbGVzIjpbeyJyb2xlIjoiUk9MRV9BRE1JTiIsImNsaWVudCI6Im1lZGljIn1dLCJpc3MiOiJodHRwczovL2VsbWVkaWtvLmNvbSIsImdpdmVuX25hbWUiOiJBZG1pbmlzdHJhdG9yIiwiY2xpZW50X2lkIjoibWVkaWMiLCJwaWN0dXJlIjpudWxsLCJzY29wZSI6IiovKi4qIiwicGhvbmVfbnVtYmVyIjpudWxsLCJleHAiOjE1OTk3MDExMDksImZhbWlseV9uYW1lIjoiTWVkaWMiLCJlbWFpbCI6ImFkbWluQG1lZGljYmcuY29tIiwidXNlcm5hbWUiOiJtZWRpYyJ9.J5Ic2VDtFw3MvuE-gyR6eUzTzaVkGgWgrt-Sf6O3I8h4QYO3CGz-pxdwpUx6xWYCvgpE_fEyV32Fv-3WS8Z-YQ' \
--header 'Content-Type: application/json' \
--data-raw '{
    "Auth": {
        "Data": [
            {
                "Username": "medic_practice1",
                "Credentials": [
                    {
                        "Type": "password",
                        "Value": "<password string>"
                    }
                ]
            }
        ],
        "MedicRegRq": {
            "OrganizationName": "Test Clinic",
            "DhId": "1234567898",
            "NhifBranch": "03"
        }
    }
}'
```
Полетата в MedicRegReq съответстват на 
    ** "OrganizationName": име на организацията
    ** "DhId": номер на лечебно заведение
    ** "NhifBranch": номер на здравен рейон

Очакван резултат:
```
{
    "Result": {
        "Id": 118,
        "Enabled": true,
        "Username": "medic_practice1",
        "EmailVerified": false,
        "Acknowledged": false,
        "Authorities": [
            {
                "Id": 119,
                "Authority": "ROLE_MEDIC",
                "Client": "56"
            }
        ],
        "Properties": [
            {
                "Id": 120,
                "Key": "roleId",
                "Value": "PractitionerRole/3205/_history/1"
            }
        ],
        "Provider": "local"
    },
    "Success": true,
    "Errors": []
}

```
Запазете полето "Id" във вътрешна база данни, защото то е необходимо в последствие при генерирането на баркод на рецептата. 

##### Вход с креденции на практиката.

```
curl --location --request POST 'https://stgerx2.e-health.bg/auth/login' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic Base64(<clientUsername>:<clientPassword>)' \
--data-raw '{
    "User": {
        "Data": [
            {
                "Username": "medic_practice1",
                "Credentials": [
                    {
                        "Type": "password",
                        "Value": "<password string>"
                    }
                ]
            }
        ]
    }
}'
```

##### Издаване на рецепта по НЗОК, бланка 5.
```
curl --location --request POST 'https://erx2.e-health.bg/fhirlite/prescription?seed=3002&category=N' \
--header 'Authorization: Bearer <token>' \
--header 'Content-Type: application/json' \
--data-raw '{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "A0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 1,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "Practitioner",
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/uin",
            "value": "2300013314"
          }
        ],
        "name": [
          {
            "use": "official",
            "family": "Поляков",
            "given": [
              "Иван",
              "Акимов"
            ],
            "prefix": [
              "д-р"
            ]
          }
        ],
        "telecom": [
          {
            "system": "phone",
            "value": "088888880",
            "use": "mobile"
          },
          {
            "system": "email",
            "value": "ipolyakov@e-health.bg",
            "use": "work"
          }
        ],
        "gender": "male"
      }
    },
    {
      "resource": {
        "resourceType": "PractitionerRole",
        "specialty": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/doctor-speciality-nhif",
                "code": "00",
                "display": "Общопрактикуващ лекар"
              }
            ]
          }
        ]
      }
    },
    {
      "resource": {
        "resourceType": "Patient",
        "extension": [
          {
            "url": "http://terminology.e-health.bg.com/Extension/patient-maternity-flag",
            "valueBoolean": false
          },
          {
            "url": "http://terminology.e-health.bg.com/Extension/patient-pregnancy-flag",
            "valueBoolean": false
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/nnbgr",
            "value": "<EGN>"
          }
        ],
        "birthDate": "1000-05-27T00:00:00.000Z",
        "gender": "male",
        "name": [
          {
            "family": "ПЕТКОВ",
            "given": [
              "СЕВДАЛИН",
              "ПЕТКОВ"
            ]
          }
        ]
      }
    },
    {
      "resource": {
        "resourceType": "Encounter",
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/enc-id",
            "value": "006655"
          }
        ],
        "period": {
          "start": "2020-09-01T00:00:00.000Z",
          "end": "2020-09-01T00:10:00.000Z"
        }
      }
    }
  ]
}
```
##### Издаване на рецепта по НЗОК, бланка 5А.
```
curl --location --request POST 'https://erx2.e-health.bg/fhirlite/prescription?seed=3002&category=A' \
--header 'Authorization: Bearer <token>' \
--header 'Content-Type: application/json' \
--data-raw '{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "A0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 1,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "B0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 2,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "MedicationRequest",
        "extension": [
          {
            "url": "http://terminology.e-health.bg/Extension/bgnhif-booklet-part",
            "valueString": "C0"
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/booklet-id",
            "value": "1520002"
          }
        ],
        "medicationCodeableConcept": {
          "coding": [
            {
              "system": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
              "code": "RF127",
              "display": "Foster"
            }
          ]
        },
        "authoredOn": "2020-09-01T00:00:00.000Z",
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J44.8"
              }
            ]
          }
        ],
        "dosageInstruction": [
          {
            "text": "SIG",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 3,
                  "unit": "units"
                },
                "rateQuantity": {
                  "value": 2,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "dispenseInterval": {
            "value": 30,
            "unit": "days"
          },
          "quantity": {
            "value": 2,
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": false
        }
      }
    },
    {
      "resource": {
        "resourceType": "Practitioner",
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/uin",
            "value": "2300013314"
          }
        ],
        "name": [
          {
            "use": "official",
            "family": "Поляков",
            "given": [
              "Иван",
              "Акимов"
            ],
            "prefix": [
              "д-р"
            ]
          }
        ],
        "telecom": [
          {
            "system": "phone",
            "value": "088888880",
            "use": "mobile"
          },
          {
            "system": "email",
            "value": "ipolyakov@e-health.bg",
            "use": "work"
          }
        ],
        "gender": "male"
      }
    },
    {
      "resource": {
        "resourceType": "PractitionerRole",
        "specialty": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/doctor-speciality-nhif",
                "code": "00",
                "display": "Общопрактикуващ лекар"
              }
            ]
          }
        ]
      }
    },
    {
      "resource": {
        "resourceType": "Patient",
        "extension": [
          {
            "url": "http://terminology.e-health.bg.com/Extension/patient-maternity-flag",
            "valueBoolean": false
          },
          {
            "url": "http://terminology.e-health.bg.com/Extension/patient-pregnancy-flag",
            "valueBoolean": false
          }
        ],
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/nnbgr",
            "value": "<EGN>"
          }
        ],
        "birthDate": "1000-05-27T00:00:00.000Z",
        "gender": "male",
        "name": [
          {
            "family": "ПЕТКОВ",
            "given": [
              "СЕВДАЛИН",
              "ПЕТКОВ"
            ]
          }
        ]
      }
    },
    {
      "resource": {
        "resourceType": "Encounter",
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/enc-id",
            "value": "006655"
          }
        ],
        "period": {
          "start": "2020-09-01T00:00:00.000Z",
          "end": "2020-09-01T00:10:00.000Z"
        }
      }
    }
  ]
}
```
##### Издаване на бяла (обикновенна) рецепта.
ToDo:
#### Получаване на информация за изписани рецепти и лекарства от аптеките работещи в регистъра
ToDo:

#### Описание на staging среда
##### Staging #1
 * host: stgerx2.e-health.bg
 * protocol: HTTPS
 * credentials: съвпадат с тези на продукционната среда
 
#### Changelog
* (2020-06-27 08:54)
    * Генериране на офлайн баркод
* (2020-07-15 19:43)
    * Описание на Staging среда
* (2020-08-06 08:49)
    * Промяна на категорията на рецептата от 'hhif' на 'nhif-5a'. Добавяне на списък с категории на рецептите.
* (2020-08-20 13:34)
    * Промяна на категорията на рецептата от 'hhif' на 'A'. Промяна в номенклатурата за категории рецепти.
    Категорията вече се отбелязва с един от следните символи - W, G, Y, N, A, B, C. 
    Отбелязването с един симовл е по-подходящо поради причината, че тази символика на отбелязване, съответства на символът използван
    при генерирането на баркода.  
* (2020-08-23 07:38)
    * Добавяне на функционалност за кодиране на лекарство по две кодови системи (NHIF, SatHealth)
    * Промяна на подхода за генериране на offline barcode 
* (2020-09-16 14:11)
    * Пример за пълен процес по регистрация, вход и издаване на рецепти
    * Получаване на информация за изписани рецепти и лекарства от аптеките работещи с регистъра
    * Примяна на формата за генериране на Баркод. В момента остава само вариант за Offline генериране чрез опоменатия алгоритъм.
