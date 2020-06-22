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
                        "niset_user"
                    ]
                }
            }
        ],
        "Action": "CREATE",
        "MedicRegRq": {
            "dhId": "2201928818",
            "organizationName": "ДКЦ ТЕСТ",
            "nhifRegion": "01",
            "nhifBranch": "22"
        }
    }
}
```
**Обърнете внимание на това че `erx-resource-server` е с малка буква**
```
...
"ClientRoles": {
   "erx-resource-server": [
        "niset_user"
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
на последващите заявки към микроуслугите обслужващи осноните бизнес функции на регистъра:
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
