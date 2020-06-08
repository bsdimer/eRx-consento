# eRx <-> PharmaStar
eRx Consento - интеграция с ФармаСтар

Този документ описва интеграцията на модела на данните на eRx Consento с модела на данните на приложението ФармаСтар

### Общо описание за работата с REST API адаптера

Всички отговори на заявки към REST API са пакетирани в обект носещ основна информация за резултата от изпълнението:

```
{
  "result": any,
  "success": boolean,
  "errors": {path: string, message: string, key: string}[]
}
``` 
* в атрибута `success` е закодиран крайния резултат на операцията под формата на булева стойност, т.е. дали операцията е успешна или не
* атрибута `result` е контейнер за данните върнати от изпълнението на заявката
* при наличие на грешки в атрибута errors се попълва масив от грешки касаещи изпълнението на заявката. Всяка грешка е под формата на обект
притежаващ атрибутите `path`, `message` и `key`. `path` е незадължителен атрибут указващ пътя на променливата, която е с невалидна стойност. 
Всички валидационни грешки притежават такъв атрибут. Понякога обаче този атрибут може да бъде празен, тъй като грешката на касае и не реферира
конкретни данни от заявката. Атрибута `message` е информативно поле описващо причината за настъпване на грешката. Езикът на който се връща
отговора зависи от стойността на хидъра `Accept-Language`, който се изпраща в заявката. Примерни стойности могат да бъдат `Accept-Language: bg-BG` 
или `Accept-Language: en-EN`. За сега API поддържа само тези два езика. 

##### Пример за успешна заявката:

##### Пример за неуспешна заявка:

##### Други грешки - некасаещи бизнес логиката на системата

* 404 NotFount - грешен URL на заявката
* 401 Unauthorized - Среща се при автентикираните заявки - означава, че липсват креденции в заяквата (??? Можем ли да ги обвием в горната структура)
* 403 Forbidden - изтекъл автентикационен токен. Трябва да се направи заявка за вземане на нов. (??? Можем ли да ги обвием в горната структура)
* 500 Internal Server Error - грешки на софтуера, които излизат извън видимостта на софтуера - например недостатъчни количесто памет и др. 

### Описание на процеса за регистрация на потребител

Регистрацията на краен потребител (Потребител на аптечния софтуер) преминава през две основни стъпки.

1. Първа стъпка - вземане на автентикационен токен за администратор. 
2. Втора стъпка - Създаване на краен потребител с помощта на автентикационния токен на администратора. 

Административни потребител за системата на PharmaStar се нарича PharmaStarPowerUser.

Процесът за това е следният:
1. Вземане на автентикационен токен за администратор. Изпълнение на POST заявка със следните данни:
```
Request method:	POST
Request URI:	https://stagingerx.e-health.bg/erx-ui/api/entry
Proxy:			<none>
Request params:	<none>
Query params:	<none>
Form params:	<none>
Path params:	<none>
Headers:		Accept=*/*
                Accept-Language=bg-BG
				Content-Type=application/json; charset=UTF-8
Cookies:		<none>
Multiparts:		<none>
Body:
{
    "user": {
        "data": [
            {
                "username": "pharmacist001",
                "credentials": [
                    {
                        "type": "password",
                        "value": "<omitted>"
                    }
                ]
            }
        ],
        "action": "LOGIN"
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
  "result": {
    "user": {
      "token": "<admin user token>"
    }
  },
  "success": true,
  "errors": []
}
```

Възможни грешки при обстоятелство - не-касаещи бизнес логиката:
* 404 NotFound - сбъркан URL на REST API
* 500 Internal Server Error - случайни грешки ???

3. Стъпка в - изполазвайте полученият токен за да изпълните нова заявка за създаване на краен потреботел:
```
Request method: POST
Request URI:    https://stagingerx.e-health.bg/erx-ui/api/auth-entry
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
    "auth": {
        "data": [
            {
                "username": "test2",
                "enabled": true,
                "firstName": "John",
                "lastName": "Doe",
                "email": "user@e-health.bg",
                "credentials": [
                    {
                        "type": "password",
                        "value": "passworString"
                    }
                ],
                "clientRoles": {
                    "erx-resource-server": [
                        "pharmastar_user"
                    ]
                }
            }
        ],
        "action": "CREATE",
        "pharmacyRegRq": {
            "address": "Test street 9",
            "state": null,
            "district": "Sofia",
            "city": "Sofia",
            "vatNumber": "123456789",
            "phone": "+359 222222",
            "pharmacyNo": "NO12345678",
            "pharmacyName": "TEST PHARMACY",
            "pharmacistFirstName": "John",
            "pharmacistFamilyName": "Doe"
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
  "result": {
    "user": {
      "result": [
        {
          "id": "4f6dffa7-a0d5-4119-a0c5-07b0ae05ac29",
          "createdTimestamp": 1590039593854,
          "username": "test2",
          "enabled": true,
          "totp": false,
          "emailVerified": false,
          "firstName": "John",
          "lastName": "Doe",
          "email": "user@e-health.bg",
          "attributes": {
            "PHARMACY_NS_CITY": [
              "Sofia"
            ],
            "PHARMACY_NS_VAT_NUMBER": [
              "123456789"
            ],
            "PHARMACY_NS_DISTRICT": [
              "Sofia"
            ],
            "PHARMACY_NS_PHARMACIST_FIRST_NAME": [
              "John"
            ],
            "PHARMACY_NS_PHARMACIST_FAMILY_NAME": [
              "Doe"
            ],
            "PHARMACY_NS_PHARMACY_NAME": [
              "TEST PHARMACY"
            ],
            "PHARMACY_NS_PHARMACY_NO": [
              "NO12345678"
            ],
            "PHARMACY_NS_ADDRESS": [
              "Test street 9"
            ],
            "PHARMACY_NS_PHONE": [
              "+359 222222"
            ]
          },
          "disableableCredentialTypes": [],
          "requiredActions": [],
          "realmRoles": [
            "pharmacist"
          ],
          "clientRoles": {
            "erx-resource-server": [
              "pharmastar_user"
            ]
          },
          "notBefore": 0,
          "access": {
            "manageGroupMembership": true,
            "view": true,
            "mapRoles": true,
            "impersonate": true,
            "manage": true
          },
          "auth": {}
        }
      ]
    }
  },
  "success": true,
  "errors": []
}

```

#### Вход в системата за краен потребител

Входа в системата става чрез: 
```
Request method: POST
Request URI:    https://stagingerx.e-health.bg/erx-ui/api/entry
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Headers:                        Accept-Language=bg-BG
                                Accept=*/*
                                Content-Type=application/json; charset=UTF-8
Cookies:                <none>
Multiparts:             <none>
Body:

{
    "user": {
        "data": [
            {
                "username": "test2",
                "credentials": [
                    {
                        "type": "password",
                        "value": "<specified password>"
                    }
                ]
            }
        ],
        "action": "LOGIN"
    }
}
```

При изпълнението на тази заявка ще получите отговор 200 ОК, като в резултата ще получите JSON с токен нужен за изпълнението 
на последващите заявки към микроуслугата за работа с рецепта на PharmaStar:
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
  "result": {
    "user": {
      "token": "<token цонтент>"
    }
  },
  "success": true,
  "errors": []
}
```

#### Заявка "Whoami" - "Кой съм аз"

За да валидирате текущо издадения токен можете да използвате следната заявка:

```
curl --location --request POST 'https://stagingerx.e-health.bg/erx-ui/api/auth-entry' \
--header 'Authorization: Bearer <access token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "auth": {
        "whoAmI": true
    }
}'
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
  "result": {
    "auth": {
      "whoAmI": {
        "id": "9cdf4017-13df-45db-a1e9-79985a9b341d",
        "createdTimestamp": 1586968251423,
        "username": "test2",
        "enabled": true,
        "totp": false,
        "emailVerified": false,
        "firstName": "User",
        "lastName": "Test",
        "email": "test2@gbg.bg",
        "attributes": {
          "PHARMACY_NS_CITY": [
            "Sofia"
          ],
          "PHARMACY_NS_VAT_NUMBER": [
            "324234234"
          ],
          "PHARMACY_NS_DISTRICT": [
            "Sofia"
          ],
          "PHARMACY_NS_PHARMACY_NAME": [
            "PHARMACY1"
          ],
          "PHARMACY_NS_PHARMACY_NO": [
            "NO11111111"
          ],
          "PHARMACY_NS_ADDRESS": [
            "Address 1"
          ],
          "PHARMACY_NS_STATE": [
            "Sofia"
          ],
          "PHARMACY_NS_PHONE": [
            "08888888"
          ]
        },
        "disableableCredentialTypes": [],
        "requiredActions": [],
        "notBefore": 0,
        "access": {
          "manageGroupMembership": true,
          "view": true,
          "mapRoles": true,
          "impersonate": true,
          "manage": true
        }
      }
    }
  },
  "success": true,
  "errors": []
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
  "result": null,
  "success": false,
  "errors": [
    {
      "message": "Грешно потребителко име или парола",
      "key": "{user.login.failed}"
    }
  ]
}

```

Грешките при регистрация на потребител могат да бъдат:

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
  "result": null,
  "success": false,
  "errors": [
    {
      "path": "user",
      "message": "Некоректен обект за регистрация",
      "key": "Registration request structure is wrong"
    },
    {
      "path": "user.pharmacyRegRq.pharmacistFirstName",
      "message": "Името на фармацевта е задължително",
      "key": "{jakarta.validation.constraints.NotBlank.message}"
    },
    {
      "path": "user.pharmacyRegRq.pharmacyNo",
      "message": "Номерът на аптека е задължителен",
      "key": "{jakarta.validation.constraints.NotBlank.message}"
    },
    {
      "path": "user.pharmacyRegRq.pharmacistFamilyName",
      "message": "Фамилията на фармацевта е задължителна",
      "key": "{jakarta.validation.constraints.NotBlank.message}"
    }
  ]
}

```

Грешка 404 получава се само в случаите че URL-то към услугите е грешно. В тялото на заявката при този случай се получава HTML съдържанние
Грешка 500 се получава в случаите на срив в системата,

### Описание на процеса на вземане на данни

Четенето на данни от регистъра изисква вход в системата посредством потребителско име и парола. 
Описанието за това как да се извърши регистрацията и взимане на токен може да бъде намеререно в предходния документ.
// ТоДо: линк към предходния документ

#### Вземане на рецепта по идентифиактор

```
curl --location --request GET 'https://stagingerx.e-health.bg/fhir/erx-phs/api/prescription?identifier=ERXN20200423000002%D0%90' \
--header 'Content-Type: application/json' \
--header 'Accept-Language: bg-BG' \
--header 'Authorization: Bearer <token>'
```

Резултата от търсенето е масив от рецепти. Стойността е масив тъй като идентификатора сам по себе си в сложна стойност 
интегрираща в себе си множество данни. Така например рецептата по НЗОК по бланка 5А притежава собствен групов идентификатор
който позволява да бъдат намерени всички рецепти по тази бланка. В най-общия случай търсенете ще бъде по идентификатор на конректната
рецепта и резултата ще бъде масив от една рецепта. 

```
{
  "result": [
    {
      "Patient": {
        "FirstName": "ГЕОРГИ",
        "MiddleName": "ТОДОРОВ",
        "LastName": "ДИМИТРОВ",
        "Age": 43,
        "Egn": "7703022402",
        "Lnch": "100000002",
        "Ssn": null,
        "PersId": null,
        "Address": null,
        "CertificateCountryCode": null,
        "CertificateIssueDate": null,
        "CertificateValidFrom": null,
        "CertificateValidTo": null,
        "CertificateNumber": null,
        "CertificateType": null,
        "BirthDate": "1977-03-02",
        "Sex": "male",
        "MaternityFlag": false,
        "PregnancyFlag": false
      },
      "ProtocolDate": null,
      "PrescriptionBookletNo": 1080005,
      "AmbSheetNo": 12580,
      "Doctor": {
        "SoftwareCode": null,
        "RhifCode": null,
        "PracticeNo": "2210113508",
        "Uin": "2300013314",
        "Specialty": "00",
        "Name": "д-р Иван Поляков",
        "Phone": "088888880"
      },
      "VeteranBookledDate": null,
      "TerritorialExpertMedicalCommissionBookledNo": null,
      "TerritorialExpertMedicalCommissionBookledDate": null,
      "IsDeleted": null,
      "DispensingDate": null,
      "WayOfPayment": null,
      "Pharmacist": null,
      "Pharmacy": null,
      "PrescriptionRows": [
        {
          "Line": 0,
          "NhifCode": "RF034",
          "IcdCode": "J30.1",
          "Description": "ZYRTEC FILM COATED TABLETS 10MG 10",
          "IsGeneric": false,
          "PrescribedQuantity": 2,
          "DispensionQuantity": null,
          "NumberOfDays": 7,
          "Price": null
        },
        {
          "Line": 1,
          "NhifCode": "SF083",
          "IcdCode": "H16.0",
          "Description": "FLAREX EYE DROPS 0.1% 5ML 1",
          "IsGeneric": true,
          "PrescribedQuantity": 1,
          "DispensionQuantity": null,
          "NumberOfDays": 10,
          "Price": null
        }
      ],
      "State": "Finished",
      "Id": null,
      "PrescriptionNo": "ERXN20200423000002А",
      "Barcode": "ERXN20200423000002А",
      "ProtocolNo": null,
      "PrescriptionDate": "2020-01-01T00:00:00",
      "VeteranBookledNo": null,
      "Part": "PartA"
    }
  ],
  "success": true,
  "errors": [
  ]
}
```

При настъпване на грешка, резултата ще бъде 200 ОК, но в структурата ще присъства обект с атрибут "errors":

```
Request method: GET
Request URI:    https://stagingerx.e-health.bg/fhir/erx-phs/api/prescription
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Headers:                        Authorization=Bearer <token>
                                Accept-Language=bg-BG
                                Accept=*/*
                                Content-Type=application/json; charset=UTF-8
Cookies:                <none>
Multiparts:             <none>

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Thu, 04 Jun 2020 16:32:18 GMT
Content-Type: application/json
Content-Length: 1032
Connection: keep-alive
access-control-allow-origin: *
x-envoy-upstream-service-time: 8

{
    "user": null,
    "auth": null,
    "errors": [
        {
            "path": null,
            "message": "Атрибута identifier е задължителен",
            "key": "{missing.queryAttribute.identifier}"
        }
    ]
}

```

__При липсваща рецепта по зададен идентификатор се връща празен масив.__

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
  "result": [
  ],
  "success": true,
  "errors": [
  ]
}

```

#### Изписване на лекарсто по рецепта

