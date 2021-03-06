# eRx <-> PharmaStar API v1.0

Този документ описва интеграционния подход и начин на работа на eRx-Pharmastar REST API v1.0

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

Административни потребител за системата на PharmaStar се нарича PharmaStarPowerUser.

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
                "Username": "pharmastar",
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
                ],
                "ClientRoles": {
                    "erx-resource-server": [
                        "pharmastar_user"
                    ]
                }
            }
        ],
        "Action": "CREATE",
        "PharmacyRegRq": {
            "Address": "Test street 9",
            "State": null,
            "District": "Sofia",
            "City": "Sofia",
            "VatNumber": "123456789",
            "Phone": "+359 222222",
            "PharmacyNo": "NO12345678",
            "PharmacyName": "TEST PHARMACY",
            "PharmacistFirstName": "John",
            "PharmacistFamilyName": "Doe"
        }
    }
}
```
**Обърнете внимание на това че `erx-resource-server` е с малка буква**
```
...
"ClientRoles": {
   "erx-resource-server": [
        "pharmastar_user"
    ]
}
...
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
        ],
        "Action": "LOGIN"
    }
}
```
*Отново при изпълнениет на тази заявка се изисква добавяне на Basic автентикация.*

При изпълнението заявката ще получите отговор 200 ОК, като в резултата ще получите JSON с токен нужен за изпълнението 
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
  "Result": {
    "User": {
      "Token": "<token content>"
    }
  },
  "Success": true,
  "Errors" []
}
```

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
      "Key" "{user.login.failed}"
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

### Описание на процеса на вземане на данни

Четенето на данни от регистъра изисква вход в системата посредством потребителско име и парола. 
Описанието за това как да се извърши регистрацията и взимане на токен може да бъде намеререно в предходния документ.

#### Вземане на рецепта по идентифиактор

```
curl --location 
--request GET 'https://erx2.e-health.bg/phs/prescription?identifier=ERXN20200423000002%D0%90' \
--header 'Content-Type: application/json' \
--header 'Accept-Language: bg-BG' \
--header 'Authorization: Bearer <token>'
```

Резултата от търсенето е масив от рецепти. Стойността е масив тъй като идентификатора сам по себе си в сложна стойност 
интегрираща в себе си множество данни. Така например рецептата по НЗОК по бланка 5А притежава собствен групов идентификатор
който позволява да бъдат намерени всички рецепти по тази бланка. В най-общия случай търсенето ще бъде по идентификатор на конректната
рецепта и резултата ще бъде масив от една рецепта. 

```
{
  "Result": [
    {
      "Patient": {
        "FirstName": "ГЕОРГИ",
        "MiddleName": "ТОДОРОВ",
        "LastName": "ДИМИТРОВ",
        "Age": 43,
        "Egn": "<omitted>",
        "Lnch": "<omitted>",
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
      "PrescriptionType": "A",
      "AmbSheetNo": 12580,
      "Doctor": {
        "SoftwareCode": null,
        "RhifCode": null,
        "PracticeNo": "<omitted>",
        "Uin": "<omitted>",
        "Specialty": "00",
        "Name": "д-р John The Docor",
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
      "PreviousPartDate":  null,
      "VeteranBookledNo": null,
      "Part": "PartA"
    }
  ],
  "Success": true,
  "Errors" [
  ]
}
```

При настъпване на грешка, резултата ще бъде 200 ОК, но в структурата ще присъства обект с атрибут "Errors"

```
Request method: GET
Request URI:    https://erx2.e-health.bg/phs/prescription
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
    "Result": null,
    "Success": false,
    "Errors" [
        {
            "Path": null,
            "Message" "Атрибута identifier е задължителен",
            "Key" "{missing.queryAttribute.identifier}"
        }
    ]
}

```

При липсваща рецепта по зададен идентификатор се връща следния отговор:

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
  "Result": [],
  "Success": false,
  "Errors" [
          {
              "Message" "Липсва рецепта по зададените критерии",
              "Id" "<UUID>"
          }
      ]
}

```

#### Изписване на лекарство по рецепта

При изписване на лекарствата в аптеката се изпълнява заявка от вида:

```
Request method:	POST
Request URI:	https://erx2.e-health.bg/phs/prescription
Proxy:			<none>
Request params:	<none>
Query params:	<none>
Form params:	<none>
Path params:	<none>
Headers:		Cache-Control=no-cache
                Accept-Language: bg-BG
                Authorization: Bearer <token>
				Accept=*/*
				Content-Type=application/json; charset=UTF-8
Cookies:		<none>
Multiparts:		<none>
Body:
{
    "State": null,
    "Id": null,
    "PrescriptionNo": "ERXN20200423000001А",
    "PrescriptionType": "A",
    "Barcode": "ERXN20200423000001А",
    "ProtocolNo": null,
    "PrescriptionDate": "2020-01-01T00:00:00",
    "VeteranBookledNo": null,
    "Part": "PartNone",
    "Patient": {
        "FirstName": "ГЕОРГИ",
        "MiddleName": "ТОДОРОВ",
        "LastName": "ДИМИТРОВ",
        "Age": 43,
        "Egn": "<omitted>",
        "Lnch": "<omitted>",
        "Ssn": null,
        "PersId": null,
        "Address": null,
        "CertificateCountryCode": null,
        "CertificateIssueDate": null,
        "CertificateValidFrom": null,
        "CertificateValidTo": null,
        "CertificateNumber": null,
        "CertificateType": null,
        "BirthDate": "2020-01-01T00:00:00",
        "Sex": "male",
        "MaternityFlag": false,
        "PregnancyFlag": false
    },
    "Doctor": {
        "SoftwareCode": null,
        "RhifCode": null,
        "PracticeNo": "<omitted>",
        "Uin": "<omitted>",
        "Specialty": "00",
        "Name": "д-р John The Doctor",
        "Phone": "088888880"
    },
    "VeteranBookledDate": null,
    "ProtocolDate": null,
    "PrescriptionBookletNo": 1080005,
    "AmbSheetNo": 12580,
    "PrescriptionRows": [
        {
            "Line": 0,
            "NhifCode": "RF034",
            "IcdCode": "J30.1",
            "Description": "ZYRTEC FILM COATED TABLETS 10MG 10",
            "IsGeneric": false,
            "PrescribedQuantity": 2,
            "DispensionQuantity": 1,
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
            "DispensionQuantity": 2,
            "NumberOfDays": 10,
            "Price": null
        }
    ],
    "TerritorialExpertMedicalCommissionBookledNo": null,
    "TerritorialExpertMedicalCommissionBookledDate": null,
    "IsDeleted": null,
    "DispensingDate": "2020-06-02T00:00:00+03:00",
    "WayOfPayment": null,
    "Pharmacist": null,
    "Pharmacy": null
}
```
Като валидациите за обекта са:
* Barcode @NotBlank
* size(PrescriptionRows) > 0
* PrescriptionRows[*][DispensionQuantity] > 0

Отговора при успешно изпълнение на операцията е: 
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
    "State": null,
    "Id": null,
    "PrescriptionNo": "ERXN20200423000001А",
    "PrescriptionType": "A",
    "Barcode": "ERXN20200423000001А",
    "ProtocolNo": null,
    "PrescriptionDate": "2020-01-01T00:00:00",
    "VeteranBookledNo": null,
    "Part": "PartNone",
    "Patient": {
      "FirstName": "ГЕОРГИ",
      "MiddleName": "ТОДОРОВ",
      "LastName": "ДИМИТРОВ",
      "Age": 43,
      "Egn": "<omitted>",
      "Lnch": "<omitted>",
      "Ssn": null,
      "PersId": null,
      "Address": null,
      "CertificateCountryCode": null,
      "CertificateIssueDate": null,
      "CertificateValidFrom": null,
      "CertificateValidTo": null,
      "CertificateNumber": null,
      "CertificateType": null,
      "BirthDate": "2020-01-01T00:00:00",
      "Sex": "male",
      "MaternityFlag": false,
      "PregnancyFlag": false
    },
    "Doctor": {
      "SoftwareCode": null,
      "RhifCode": null,
      "PracticeNo": "<omitted>",
      "Uin": "<omitted>",
      "Specialty": "00",
      "Name": "д-р John The Doctor",
      "Phone": "088888880"
    },
    "VeteranBookledDate": null,
    "ProtocolDate": null,
    "PrescriptionBookletNo": 1080005,
    "AmbSheetNo": 12580,
    "PrescriptionRows": [
      {
        "Line": 0,
        "NhifCode": "RF034",
        "IcdCode": "J30.1",
        "Description": "ZYRTEC FILM COATED TABLETS 10MG 10",
        "IsGeneric": false,
        "PrescribedQuantity": 2,
        "DispensionQuantity": 1,
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
        "DispensionQuantity": 2,
        "NumberOfDays": 10,
        "Price": null
      }
    ],
    "TerritorialExpertMedicalCommissionBookledNo": null,
    "TerritorialExpertMedicalCommissionBookledDate": null,
    "IsDeleted": null,
    "DispensingDate": "2020-06-02T00:00:00+03:00",
    "WayOfPayment": null,
    "Pharmacist": null,
    "Pharmacy": null
  },
  "Success": true,
  "Errors" []
}
```

При липсваща рецепта по зададен идентификатор се връща следния отговор:

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
  "Result": [],
  "Success": false,
  "Errors" [
          {
              "Message" "Липсва рецепта по зададените критерии",
              "Id" "<UUID>"
          }
      ]
}

```

#### Appendix #1 - Формат на идентификатор на рецепта.
Идентификаторът на рецептата е сложна структура от тип символен низ. 
Примерен идентификатр за рецепта по НЗОК е: *ERX1592863105N00246А*
От ляво на дясно данните в идентификатора са следните:
* ERX - три симовелен низ определящ регистъра
* 1592863105 - десет символа определящо времето на издаване на рецетата в Unix timestamp. (Секундите от 1 Януари 1970 година до сега) 
* N - един символ определящ типа на рецетата. Може да бъде W(за обикновенна "бяла" рецепта), G(заелена рецепта), Y(жълта рецепта), N(за рецепта по НЗОК), A,B,C съответно за рецепта по НЗОК бланки 5а, 5б и 5в.
* 002 - трицифрено число - counter с допълващ символ '0'. Използва се за създаване на пореден уникален номер, тъй като е възможно издаването на две рецепти в една и съща секунда. Стойностите му са от 0 до 999 след което започва отново от 0.
* 46 - уникален идентификатор на потребителския профил който е създал рецептата в регистъра. Размера на това поле е произовлен и варира в зависимост от общата брока потребителски профили. 
* А - използва се само при рецепти по НЗОК бланка 5А. Отговаря на отрязъка който е част от рецептата. При рецептите издадени в една бланка този символ липсва.   

#### Appendix #2 - Генератор на баркодове
 
Примерна заявка за генериране на баркод по идентификатор на рецептата.
https://erx2.e-health.bg/print/barcode/code128?code=ERX1592920883N00360A&scale=3

<img src="https://erx2.e-health.bg/print/barcode/code128?code=ERX1592920883N00360A&scale=3"/>

Формат на идентификатор на рецепта.
Идентификаторът на рецептата е сложна структура от тип символен низ. 
Примерен идентификатр за рецепта по НЗОК е: *ERX1592863105N00246А*
От ляво на дясно данните в идентификатора са следните:
* ERX - три симовелен низ определящ регистъра
* 1592863105 - десет символа определящо времето на издаване на рецетата в Unix timestamp. (Секундите от 1 Януари 1970 година до сега) 
* N - един символ определящ типа на рецетата. Може да бъде W(за обикновенна "бяла" рецепта), G(заелена рецепта), Y(жълта рецепта), N(за рецепта по НЗОК).
* 002 - трицифрено число - counter с допълващ символ '0'. Използва се за създаване на пореден уникален номер, тъй като е възможно издаването на две рецепти в една и съща секунда. Стойностите му са от 0 до 999 след което започва отново от 0.
* 46 - уникален идентификатор на потребителския профил който е създал рецептата в регистъра. Размера на това поле е произовлен и варира в зависимост от общата брока потребителски профили. 
* А - използва се само при рецепти по НЗОК бланка 5А. Отговаря на отрязъка който е част от рецептата. При рецептите издадени в една бланка този символ липсва.

#### Appendix #3 Категории рецепти
Категорията на рецептат се специфицира в атрибута `PrescriptionType` и може да притежава една от следните стойности:
* W - бяла рецептурна бланка
* G - зелена рецептурна бланка
* Y - жълта рецептурна бланка
* N - рецепта по НЗОК - бланка №5
* A - рецепта по НЗОК - бланка №5а
* B - рецепта по НЗОК - бланка №5б
* C - рецепта по НЗОК - бланка №5в

Ако рецептата е с категории започваща с А, B, C или N, то кодът на лекарството, трябва да бъде задължително от позитивния лекарствен списък по НЗОК.

#### Appendix #4 Код на лекарствен продукт
Атрибута PrescriptionRows представлява описание на един ред от рецептурната бланка:

```
...
"PrescriptionRows": [
        {
          "Line": 0,
          "NhifCode": "RF034",
          "IcdCode": "J30.1",
          "Description": "ZYRTEC FILM COATED TABLETS 10MG 10",
          "IsGeneric": false,
          "PrescribedQuantity": 2,
          "DispensionQuantity": 0,
          "NumberOfDays": 7,
          "Price": 0.0,
          "MedicationCodeSystemUrl": "http://terminology.e-health.bg/CodeSystem/mc-nhif",
          "MedicationCode": "RF034",
          "DosageInstruction": {
            "Text": "сутрин на гладно по едно впръскване"
          }
        },
...
```  
В последната версия на API в отговора на заявката се включват и атрибутите `MedicationCodeSystemUrl` и `MedicationCode`.
Атрибута `MedicationCodeSystemUrl` обозначава системата по която се кодира,a `MedicationCode` предствлява кода на лекарството.
Системата е във формат на URL, който може да носи в себе си информация за издателя на кодовата система, версия, както и други атрибути. 

Понастоящем са позволени две системи за кодиране:
 - http://terminology.e-health.bg/CodeSystem/mc-nhif - обозначава лекарствените продукти от позитивния лекарствен списък
 - http://sathealth.com/cs/pt_code - обозначава лекарствените продукти от списъкут изготвен от SatHealth. Пълна спецификация на този списък можете да намерите тук: http://pharmacy.e-health.bg/ml
 
Кодовете от системата на SatHealth започват с префикса PT. В тяхната номенклатура са кодирани, лекарствата от позитивния лекарствен списък, OTC и хранителни добавки.
Рецептите по НЗОК са винаги по http://terminology.e-health.bg/CodeSystem/mc-nhif, без значение че кодовата система на SatHealth притежава същите лекарствени продукти но с друг код. 
При рецептите с лекарства кодирани по системата на SatHealth липсва атрибута `NhifCode`:

```
...
"PrescriptionRows": [
        {
          "Line": 0,
          "IcdCode": "J30.1",
          "Description": "ZYRTEC D TABLETS LA 14",
          "IsGeneric": false,
          "PrescribedQuantity": 2,
          "DispensionQuantity": 0,
          "NumberOfDays": 7,
          "Price": 0.0,
          "MedicationCodeSystemUrl": "http://sathealth.com/cs/pt_code",
          "MedicationCode": "PT004139",
          "DosageInstruction": {
            "Text": "сутрин на гладно по едно впръскване"
          }
        },
...
``` 

#### Appendix #5 - Описание на дозировката
При всички рецепти в структурата на лекарствения продукт присъства атрибутът `DosageInstruction`. Това е мястото в което се описват
предписаниета за ползване на медикамента. Формата на това поле може да бъде представен по следния начин:

```
{
...
        "DosageInstruction": [
                         {
                             "Sequence": 0,
                             "Text": "вечер преди лягане",
                             "DoseAndRate": [
                                 {
                                     "DoseQuantity": {
                                         "Value": 2.0,
                                         "Unit": "units"
                                     },
                                     "RateQuantity": {
                                         "Value": 1.0,
                                         "Unit": "daily"
                                     }
                                 }
                             ],
                             "MaxDosePerPeriod": {
                                 "Numerator": {},
                                 "Denominator": {}
                             }
                         }
                     ]
                 },
...
```
Атрибутът DosageInstruction представлява масив от елементи със следната структура:
 * Sequence - Пореден номер на предписанието. Това се ползва в случаите, когато лекарството се изписва в различни дозировки в различен период от лечението. Sequence отговаря на поредния номер на периода на приложение. 
 * Text - текстова репрезентация на дозировката, в която е описано всичко необходимо за приема на лекарството. Тази характеристика е т. нар SIG. Инструкциите за дозиране на свободен текст могат да се използват за случаите, когато инструкциите са твърде сложни за кодиране. Съдържанието на този атрибут не включва името или описанието на лекарството. Когато има кодирани инструкции, инструкциите за свободен текст все още могат да бъдат представени за показване на хора, които приемат или прилагат лекарството. Очаква се текстовите инструкции винаги да се попълват. Допълнителна информация относно прилагането или приготвянето на лекарството трябва да бъде включена като текст.
 * DoseAndRate.Dose[x] - Използва се в случаите, когато трябва да се упомене количеството лекарство в единица доза. Този атрибут е с променливо съдържание, т.е. може да има една от следните стойности: 
    - DoseAndRate.DoseQuantity - Количество на лекарствения продукт в една доза, представено като брой и единица. Например "2 таблетки", "2 впръсквания" и др.
    - DoseAndRate.DoseRange - Количество на лекарствения продукт представен, като минимална и максимална стойност на приема в една доза. Например "2 до 3 таблетки".
 * DoseAndRate.Rate[x] - Използва се в случаите, когато трябва да се опише количество лекарство за единица време. Идентифицира скоростта, с която лекарството ще бъде въведено в пациента. Този атрибут е с променливо съдържание, т.е. може да има една от следните стойности:
    - DoseAndRate.RateQuantity - Точен период от време, в който се прилага единична доза. Например "6 часа". Чете се "за 6 часа"
    - DoseAndRate.RateRatio - Използва Ratio ресурс. Използването на този атрибут е с цел описание на скоростта, с която лекарството се приема. Прилага се, за да се упомене брой приеми за брой часове. Например: "2 приема на всеки 8 часа" (2 repetitions/8 hr), "5 таблетки на всеки час", "2 впръсквания на всеки 6 часа". В повечето случаи този ресурс може да изрази и максималното време на приложение на лекарството. Например:  "500 ml / 2 часа" предполага продължителност от 2 часа. 
 * MaxDosePerPeriod - Това е предназначено за използване като допълнение към дозировката, когато има горна граница на приложение. Например „2 таблетки на всеки 4 часа до максимум 8 таблетки на ден“. В този случаи ресурса описва 8 tablets/1 days:
 ```
                            "MaxDosePerPeriod": {
                                 "Numerator": {
                                    "Value": 8
                                    "Unit": "units"
                                 },
                                 "Denominator": {
                                    "Value": 1
                                    "Unit": "days"
                                 }
                             }
```
Атрибутът `Unit` в обекта от тип `Quantity` може да притежава различни стойности съобразени с графата PHRM_FORM на номенклатурата на SATHealth.

При някои рецепти присъства само текстова репрезентация на дозировката или т.нар SIG, в която е описано всичко необходимо за приема на лекарството:
```
          "DosageInstruction": {
            "Text": "3 таблетки на всеки 6 часа"
          }
```

_Повече информация за ресурса Dosage можете да намерите тук: https://www.hl7.org/fhir/dosage.html_

### Staging среда
* Staging #2
    * host: stg2.e-health.bg
    * port: TCP/21003
    * protocol: HTTP
    * credentials: същите както на продукционната среда
    * текуща API версия: 0.0.2-SNAPSHOT
    * URL с прокси https://stgerx2.e-health.bg/
    * Swagger: http://stgerx2.e-health.bg:21003/swagger-ui/index.html

### Последни промени (Changelog)
* (16/06/2020)
    * Логин: Промяна на URL. Актуално: https://erx2.e-health.bg/auth/login. При всяка логин заявка се добавя Authentication: Basic <> header.
    * Pharmastar Admin login. Промяна на потребителското име от pharmacist001 на pharmastar
    * Регистрация: Промяна на URL. Актуално: https://erx2.e-health.bg/auth/register. Резултатната структура е различно от предходната. 
    * Whoami: Промяна на URL. Актуално: https://erx2.e-health.bg/user/me Промяна на типа заявка. Преди беше POST сега е GET.
    * Pharmastar Service: Промяна на URL. Актуално: https://erx2.e-health.bg/phs/prescription

* (20/08/2020)
    * Appendix #3 Категории рецепти
    * Добавяне на информация за staging среда.

* (23/08/2020)
    * Appendix #4 Код на лекарствен продукт

* (31/08/2020)
    * Appendix #5 - Описание на дозировката
    * Добавяне на Swagger в описанието на стейджинг средата
