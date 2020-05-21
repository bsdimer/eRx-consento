# eRx <-> PharmaStar
eRx Consento - интеграция с ФармаСтар

Този документ описва интеграцията на модела на данните на eRx Consento с модела на данните на приложението ФармаСтар

#### Описание на процеса на вземане на данни

Четенето на данни от регистъра изисква вход в системата посредством потребителско име и парола. 
Описанието за това как да се извърши регистрацията и взимане на токен може да бъде намеререно в предходния документ.
// ТоДо: линк към предходния документ

След вземането на токен стартирайте търсене за обект от тип MedicationRequest чрез следната примертна заявка.

 GET https://stagingerx.e-health.bg/fhir/MedicationRequest?identifier=ERXW20200423000001&_include=*
 
като параметърът identifier е номера(идентификатор) на рецептата, опицията _include=* указва искането в резултата 
да бъдат добавени всички ресурси релативни към текущата рецепта. 
В полученият резултат се съдържат всички необходими данни за попълването на модела на ФармаСтар. 
В него може има обекти от няколко различни типа: **MedicationRequest, Medication, PractitionerRole, Patient, Encounter.**

- MedicationRequest е ресурсът, който указва един ред от предписанието в рецептата. Ако изписаната рецепта има повече от едно лекарство, то в резултата ще присъстват повече от един ресурс от тип MedicationRequest.
Всички видове рецепти, както и протокола по НЗОК представляват ресурси от тип MedicationRequest. Атрибута който ги отличава е MedicationRequest.category.
Категориите MedicationRequest са представени под формата на CodeableConcept атрибут (https://www.hl7.org/fhir/datatypes.html#CodeableConcept).
Кодовете на на съответните рецепти могат да бъдат разгледани в кодовата система "Категории рецепти" ( GET https://stagingerx.e-health.bg/fhir/CodeSystem/medication-request-category-bg). Те биват: 
**white**(обикновенна бяла рецепта), **yellow**(жълта рецепта), **green**(зелена рецепта), **nhif**(рецепта по НЗОК), **nhif-military-veterans**(рецепта за ветерани), 
**nhif-military-invalids-below50**(Военно-инвалиди с до 50% инвалидизиране), **nhif-military-invalids-over50**(Военно-инвалиди с над 50% инвалидизиране), **nhif-protocol**(протокол за лекарства по НЗОК).  
- Medication е лекарственият продукт, който е изписан в рецептата. Този ресурс може да отстъства от резултата
ако MedicationRequest ресурса реферира лекарството само по код (MedicationRequest.medicationCodeableConcept - https://www.hl7.org/fhir/medicationrequest-definitions.html#MedicationRequest.medication_x_).
 Ако обаче лекарството е реферирано с обекта в eRx FHIR базата данни, то в резулата ще присъства и обект от тип Medication
 Отново при наличие на повече от едно предписание в рецептата, в резултата ще има повече от един Medication обект. 
- PractitionerRole това е ресурса, който реферира доктора, издал рецептата. Това всъщност е ресурс обединяващ в себе си информация за лекаря, лечебното заведение и локацията, в която е издадена рецептата. 
- Encounter представлява ресурса съответстващ на медицинския преглед. 
- Patient е ресурсния обект описващ пациента.

##### Съответствие на моделите

Съответствие с модела на клас _Prescription_.

- **PrescriptionNo** - тази стойност може да бъде взета от атрибута identifier.value на някой от ресурсите MedicationRequest (ако рецептата притежава
повече от един MedicationRequest всички те имат един и същ идентификатор). Тъй като е възможно ресурса MedicationRequest да има повече
от един identifier атрибут PrescriptionNo номера на самата рецепта се съдържа в атрибута, който има "system": "http://erx.e-health.bg/ns/prescription-id"
, т.е.

```...
{
    "resourceType": "MedicationRequest",
    "identifier": [
              {
                "system": "http://erx.e-health.bg/ns/prescription-id",
                "value": "ERXN20200423000001"
              },
              {
                "system": "http://erx.e-health.bg/ns/mr-id",
                "value": "ERXN20200423000001-А1"
              },
              {
                "system": "http://erx.e-health.bg/ns/booklet-id",
                "value": "1080005"
              }
    ],
    ...
} 
```

Идентификатора е символен низ със следната структура:

```ERXW20200423000001```

където ERX са 3 символа за издател на идентификатора
W - означава бяла рецепта, може да има N(Рецепта по НЗОК), G,Y съответно за зелена и жълта, P за протокол, М(Ветерани), 
I(Военно-инвалиди с до 50% инвалидизиране), J(Военно-инвалиди с над 50% инвалидизиране)
20200423 - е датата на издаване на рецептата във формат YYYYMMDD
000001 - е поредия номер в един ден (тази последна част може да бъде с променлива дължина, но винаги padding character e 0-нула)

Идентификаторът с именовано пространство **http://erx.e-health.bg/ns/mr-id** реферира конкретния изписан медикамент в състава на рецептата.
Тъй като номенклатурата позволява изписването на лекарства в бланка 5А (три разпечатки), то в случая лекарството изписано
с този идентификатор ERXN20200423000001-**А1** се намира в първа позиция на отрязък А. 

Идентифиакторът с именовано пространство **http://erx.e-health.bg/ns/booklet-id** определя номера на рецептурната книжка. 
Така например за да се намерят всички лекарства (рецепти) изписани по рецептурна книжка с номер 1080005 се изпълнява следната
заявка:

```
GET https://stagingerx.e-health.bg/fhir/MedicationRequest?identifier=http://erx.e-health.bg/ns/booklet-id|1080005
```

В горния пример ресурсът има няколко идентификатора, първият е идентификатора на рецептата а втория е идентификатора на 
протокола по НЗОК, към който е издадена конкретната рецепта.
- **PrescriptionDate** - стойността на този атрибут може да бъде взета от MedicationRequest.authoredOn
- **ProtocolNo** - в случай че рецептата е издадена в контекста на протокол за лекарства по НЗОК, то тя притежава два 
идентификатора както е показано в горния пример. Разликата е в това, че именованото пространство (namespace) на 
идентификатора на протокола се характеризира със "system": "http://erx.e-health.bg/ns/nhif-protocol-id" 
- **ProtocolDate** - протокола в eRx FHIR представлява MedicationRequest с категория nhif-protocol. Датата на протокола се намира в 
атрибута MedicationRequest.authoredOn.
- **PrescriptionBookletNo** - ???
- **AmbSheetNo** - номера на амбулаторен лист се взима от Encounter.identifier.value. Ресурса Encounter представлява запис 
за самия преглед.
- **Doctor** - Класът Doc се попълва от ресурса PractitionerRole. Съответствието на стойностите е както следва:

```
...
{
    "resourceType": "PractitionerRole",
    "id": "5",
    "meta": {
        "versionId": "2",
        "lastUpdated": "2020-04-13T14:26:44.150+00:00"
    },
    "identifier": [
        {
            "system": "http://erx.e-health.bg/ns/uin",
            "value": "2300013314" <-- Тов е УИН номера на доктора
        },
        {
            "system": "http://erx.e-health.bg/ns/rhif-id",
            "value": "2200000010" <-- Това е код по РЗОК на лечебното заведение
        },
        {
            "system": "http://erx.e-health.bg/ns/bulstat",
            "value": "1202340392" <-- ЕИК на лечебното заведение
        }
    ],
    "telecom": [
        {
            "system": "phone",
            "use": "mobile",
            "value": "088888880"
        },
        {
            "system": "email",
            "use": "work",
            "value": "doc1@e-health.bg"
        }
    ],
    "practitioner": {
        "reference": "Practitioner/4",
        "type": "Practitioner",
        "display": "д-р Иван Поляков" <-- Имена и префикс на доктора
    },
    "organization": {
        "reference": "Organization/2",
        "type": "Organization",
        "display": "ГППМ - МКЦ Моят лекар" <-- Име на лечебното заведение в което е издадена рецептата
    },
    "code": [
        {
            "coding": [
                {
                    "system": "http://terminology.hl7.org/CodeSystem/practitioner-role",
                    "code": "doctor",
                    "display": "Доктор" <-- Код на длъжността на доктора в лечебното заведение
                }
            ]
        }
    ],
    "specialty": [
        {
            "coding": [
                {
                    "system": "http://terminology.e-health.bg/CodeSystem/doctor-speciality-nhif",
                    "code": "00", <-- Код на специалност по НЗОК
                    "display": "Общопрактикуващ лекар" 
                }
            ]
        }
    ],
    "location": [
        {
            "reference": "Location/3",
            "type": "Location",
            "display": "клон Плевен" <-- Име на локация в която е издадена рецептата
        }
    ]
}
...
```
- **Patient** - Информацяита за пациента се взима от обекта с _"resourceType": Patient_. 
Съответствието на моделите е както следва:

```
{
    "resourceType": "Patient",
    "id": "13",
    "meta": {
        "versionId": "2",
        "lastUpdated": "2020-04-15T17:07:29.299+00:00"
    },
    "text": {
        "status": "generated",
        "div": "<div xmlns=\"http://www.w3.org/1999/xhtml\"><div class=\"hapiHeaderText\">ГЕОРГИ ТОДОРОВ <b>ДИМИТРОВ </b></div><table class=\"hapiPropertyTable\"><tbody><tr><td>Identifier</td><td>7611064085</td></tr></tbody></table></div>"
    },
    "extension": [
        {
            "url": "http://terminology.elmediko.com/Extension/nhif-branch",
            "valueString": "08" <-- РЗИ код
        },
        {
            "url": "http://terminology.elmediko.com/Extension/nhif-region",
            "valueString": "15" <-- код на здравен район по НЗОК
        },
        {
            "url": "http://terminology.elmediko.com/Extension/patient-maternity-flag",
            "valueBoolean": false <-- майчинство
        },
        {
            "url": "http://terminology.elmediko.com/Extension/patient-pregnancy-flag",
            "valueBoolean": false <-- бременност
        }
    ],
    "identifier": [
        {
            "use": "official",
            "system": "http://erx.e-health.bg/ns/nnbgr",
            "value": "7703022402" <-- ЕГН на пациента
        },
        {
            "system": "http://erx.e-health.bg/ns/nnfra",
            "value": "100000004594894850941" <-- Национален номер за страна Франция
        },
        {
            "system": "http://erx.e-health.bg/ns/bg-tpr",
            "value": "0703022411" <-- Личен номер на чужденец ЛНЧ
        },
        {
            "system": "http://erx.e-health.bg/ns/bg-prc",
            "value": "100000003", <-- Номер на лична карта
            "period" { <-- Валидност на документа за самоличност
                "start": "2017-01-01T00:00:00.000Z",
                "end": "2025-01-01T00:00:00.000Z"
            },
            "assigner": { <--  Издател на документ за самоличност
                "display": "МВР София"
            }
        }
    ],
    "active": true,
    "name": [
        {
            "use": "official",
            "family": "ДИМИТРОВ", <-- Фамилия
            "given": [
                "ГЕОРГИ", <-- Собствено име
                "ТОДОРОВ" <-- Презиме 
            ]
        }
    ],
    "gender": "male" <-- Пол на пациента. Може да има следните стойности: male | female | other | unknown
    "birthDate": "1988-01-03T22:00:00.000Z", <-- Рожденна дата в ISO Datetime формат
    "address": [ <-- Адрес на пациента
        {
            "use": "home",
            "type": "physical",
            "line": [
                "Петко Д. Петков"
            ],
            "city": "Пловдив",
            "district": "Пловдив",
            "state": "Пловдив",
            "country": "България"
        }
    ]    "birthDate": 
}
```
- **PartType** - Определя се от идентификатора на рецептата. Идентификаторът с именовано пространство **http://erx.e-health.bg/ns/mr-id** 
реферира конкретния изписан медикамент в състава на рецептата.
Тъй като номенклатурата позволява изписването на лекарства в бланка 5А (три разпечатки), то в случая лекарството изписано
с този идентификатор ERXN20200423000001-**А1** се намира в първа позиция на отрязък А. 

- **VeteranBookledNo** - Не е имплементирано все още, но най-вероятно ще бъде идентификатор в конкретно именовано пространство.
- **VeteranBookledDate** - Не е имплементирано все още, но най-вероятно ще бъде идентификатор в конкретно именовано пространство.
- **TerritorialExpertMedicalCommissionBookledNo** - Не е имплементирано все още, но най-вероятно ще бъде идентификатор в конкретно именовано пространство.
- **TerritorialExpertMedicalCommissionBookledDate** - Не е имплементирано все още, но най-вероятно ще бъде идентификатор в конкретно именовано пространство.

##### Позволение за замяна на медикамента

Ресурса MedicationRequest носи и информация за това дали позволена замяна на медикамента. Това е инидикирано с 
атрибута MedicationRequest.substitution.allowedBoolean:

```
...
    resourceType: "MedicationRequest"
    "substitution": {
        "allowedBoolean": false
    }
...
```

##### Статус на рецепта

Статус на рецептата се определя от атрибута MedicationRequest.status който може да притежава следните стойности:
__active | on-hold | cancelled | completed | entered-in-error | stopped | draft | unknown__.
В повечето случаи създадената и необслужена рецепта е в състояние **active**. След издаването на 
лекарство по нея, тя става в статус **completed**. 

#### Издаване на лекарството по обикновенна - бяла рецепта

Издаването на лекарството в eRx-Consento става чрез създаване на ресурс от тип MedicationDispense (https://www.hl7.org/fhir/medicationdispense.html)
заедно с промяна на статуса на MedicationRequest в completed. Двете операции трябва да бъдат изпълнени в 
рамките на транзакция, като за целта се създава следния ресурс:

```
curl --location --request POST 'http://stagingerx.e-health.bg/fhir/' \
--header 'Authorization: Bearer <token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "resourceType": "Bundle",
    "type": "transaction",
    "entry": [
        {   // Този обект винаги е един и същ. Промяната е само в request.url атрибута, която трябва да сочи ресурса за който се прави издаването на лекарството.
            // В атрибута data се съхранява Base64 закодиран стринг на FHIR PATCH заявка
            // В некодиран вид тя изглежда така: [{ "op": "replace", "path": "/status", "value": "completed" }]
            // което означава промени стройността на атрибута status на "completed"
            "resource": {
                "resourceType": "Binary",
                "contentType": "application/json-patch+json",
                "data": "WwogIHsgIm9wIjogInJlcGxhY2UiLCAicGF0aCI6ICIvc3RhdHVzIiwgInZhbHVlIjogImNvbXBsZXRlZCIgfQpd"
            },
            "request": {
                "method": "PATCH",
                "url": "MedicationRequest/11" <-- Записва се референцията за която се отнася лекарството
            }
        },
        {
            "fullUrl": "urn:uuid:aa4abd42-985b-461a-b15d-c15c04f5e634",
            "resource": {
                "resourceType": "MedicationDispense",
                "status": "completed", <-- Съответства на DispensionState@PharmaStarModel, Може да притежава следните статуси: preparation | in-progress | cancelled | on-hold | completed | entered-in-error | stopped | declined | unknown
                "medicationReference": { <-- Референция към лекарството. Копира се от MedicationRequest.medicationReference
                    "reference": "Medication/12", 
                    "display": "AUGMENTIN FILM COATED TABLETS 875/125MG 14"
                },
                "subject": { <-- Референция към пациента, копира се от MedicationRequest.subject
                    "reference": "Patient/13", 
                    "display": "ГЕОРГИ ДИМИТРОВ"
                },
                "receiver": {
                    "display": "Димитър Димитров" <-- Имена на човека получил лекартствата
                },
                "quantity": { <-- Количество на издаденото лекарство
                    "value": "2",  
                    "unit": "опаковки"
                },
                "authorizingPrescription": { <-- Референция към MedicationRequest обекта
                    "reference": "MedicationRequest/11" 
                },
                "whenHandedOver" "2020-04-22T15:05:22.000Z" <-- Момента на издаване на лекраството,
                "note": "Допълнителна информация - ако има нужда от такава",
                "substitution": {
                    "wasSubstituted": true,
                    "reason": [
                        {
                            "text": "Причина за замяна на лекарството"
                        } 
                    ]
                }
            },
            "request": {
                "method": "POST"
            }
        }
    ]
}'
```

#### Издаване на лекарство със заместване

Издаването на рецепти по здравна каса е подобен на горния процес, с няколко малки допълнения.
// ToDo:


#### Работа с протоколи за лекарства

За да направите търсене за протокол за лекарство по идентификатор изпълнете:
```
https://stagingerx.e-health.bg/fhir/MedicationRequest?identifier=http://erx.e-health.bg/ns/nhif-protocol-id|100002
```
където http://erx.e-health.bg/ns/nhif-protocol-id е именованото пространство за идентификатори на протоколи, 100002 е номера на 
конкретния протокол. При това търсене резултата ще бъде Bundle с 4 ресурса от тип MedicationRequest. Два от резултатите са
с категория nhif-protocol и два с категория nhif.  
Протокола сам по себе си е MedicationRequest с категория "nhif-protocol". Идентификаторите на всеки MedicationRequest могат да бъдат описани по следния начин:

```
...


{
        "resourceType": "MedicationRequest",
        "id": "3995",
        "meta": {
          "versionId": "1",
          "lastUpdated": "2020-04-27T12:00:46.977+00:00",
          "source": "#5e50589f-cc52-46"
        },
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/nhif-protocol-id",
            "value": "100002"  // Номер на протокол
          },
          {
            "system": "http://erx.e-health.bg/ns/mr-id",
            "value": "100002-1"  // Идентификатор на конкретния ред в протокола описващ едно лекарство
          }
        ],
        "status": "active",
        "category": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                "code": "nhif-protocol" // Категория оказваща че това е запис на лекарство в състава на протокола
              }
            ]
          }
        ],
        "medicationReference": {
          "reference": "Medication/695",
          "display": "ZYRTEC FILM COATED TABLETS 10MG 10"
        },
        "subject": {
          "reference": "Patient/314",
          "display": "ГЕОРГИ ДИМИТРОВ"
        },
        "encounter": {
          "reference": "Encounter/3781"
        },
        "authoredOn": "2020-01-01T00:00:00.000Z",
        "requester": {
          "reference": "PractitionerRole/311",
          "display": "ГППМП - МКЦ \"Моят лекар\""
        },
        "recorder": {
          "reference": "Practitioner/310",
          "display": "д-р Иван Поляков"
        },
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "J30.1" // Код на диагноза за протокола
              }
            ]
          }
        ],
        "groupIdentifier": {
          "value": "100002"
        },
        "dosageInstruction": [
          {
            "doseAndRate": [
              { // Дневна доза
                "doseQuantity": {
                  "value": 1,
                  "unit": "tablets"
                },
                "rateQuantity": {
                  "value": 2, // Да се чете - два пъти на ден
                  "unit": "daily"
                }
              },
              { // Месечна доза доза
                "rateRatio": {
                  "numerator": {
                    "value": 61,
                    "unit": "tablets"
                  },
                  "denominator": {
                    "unit": "monthly" // Да се чете месечно
                  }
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "validityPeriod": {
            "start": "2020-01-01T00:00:00.000Z",
            "end": "2020-06-01T00:00:00.000Z" // Период на валидност на протокола
          }
        }
      }
...
```

Когато при това търсене категорията на MedicationRequest-a е с код "nhif" то това е ред от рецепта издаден към кокнретния протокол. 
Идентификатора на протокола към който е издадена може да се вземе от идентификатора със система *"system": "http://erx.e-health.bg/ns/nhif-protocol-id"*,   
Например следния ресурс може да бъде обяснен по следния начин:

```
...
      {
        "resourceType": "MedicationRequest",
        "id": "3998",
        "meta": {
          "versionId": "1",
          "lastUpdated": "2020-04-27T12:15:13.033+00:00",
          "source": "#f26ea636-2e28-4a"
        },
        "identifier": [
          {
            "system": "http://erx.e-health.bg/ns/prescription-id",
            "value": "ERXP20200429000001" // Идентификатор на рецептата. Четвъртия символ в нея (P) указва че това е рецепта издадена по протокол
          },
          {
            "system": "http://erx.e-health.bg/ns/mr-id",
            "value": "ERXP20200429000001-A2" // Идентификатор на реда в рецептата - изписан е на втора позиция 
          },
          {
            "system": "http://erx.e-health.bg/ns/nhif-protocol-id",
            "value": "100002" // Идентифиактор на протокола по който е издадена рецептата
          },
          {
            "system": "http://erx.e-health.bg/ns/mr-id",
            "value": "100002-1" // Идентификатор на реда от протокола по който е издадена рецептата
          }
        ],
        "status": "active",
        "category": [
          {
            "coding": [
              {
                "system": "http://terminology.e-health.bg/CodeSystem/medication-request-category-bg",
                "code": "nhif"  // Рецепта по НЗОК. Комбинацията на това поле с идентификатор със система "http://erx.e-health.bg/ns/nhif-protocol-id" указва че това е ред от рецепта издадена по протокол.
              }
            ]
          }
        ],
        "medicationReference": {
          "reference": "Medication/696",
          "display": "FLAREX EYE DROPS 0.1% 5ML 1"
        },
        "subject": {
          "reference": "Patient/314",
          "display": "ГЕОРГИ ДИМИТРОВ"
        },
        "encounter": {
          "reference": "Encounter/3781"
        },
        "authoredOn": "2020-04-29T00:00:00.000Z",
        "requester": {
          "reference": "PractitionerRole/311",
          "display": "ГППМП - МКЦ \"Моят лекар\""
        },
        "recorder": {
          "reference": "Practitioner/310",
          "display": "д-р Иван Поляков"
        },
        "reasonCode": [
          {
            "coding": [
              {
                "system": "http://hl7.org/fhir/sid/icd-10",
                "code": "H16.0" // Диагноза по която се издава рецептата
              }
            ]
          }
        ],
        "groupIdentifier": {
          "value": "ERXP20200429000001"
        },
        "dosageInstruction": [
          {
            "text": "вечер преди лягане",
            "doseAndRate": [
              {  // Дозировка на лекарството, описана в полето "S." на бланката за рецепта. Най-често това се показва като "S. 1 x 1"  
                "doseQuantity": {
                  "value": 1, 
                  "unit": "drop"
                },
                "rateQuantity": {
                  "value": 1,
                  "unit": "daily"
                }
              }
            ]
          }
        ],
        "dispenseRequest": {
          "quantity": {
            "value": 1, // Брой опаковки на лекарството отпуснати по тази рецепта. Това поле интерпретира стойността "D." в бланката на рецептата
            "unit": "pack"
          }
        },
        "substitution": {
          "allowedBoolean": true
        }
      }
...
```
##### Често използвани заявки за търсене

- Търсене на всички рецепти издадени по протокол с номер 100002, заедно с прилежащите им ресурси 
GET https://stagingerx.e-health.bg/fhir/MedicationRequest?identifier=http://erx.e-health.bg/ns/nhif-protocol-id|100002&_include=*

- Търсене на всички рецепти издадени по рецептурна книжка, заедно с прилжащите им ресурси
GET https://stagingerx.e-health.bg/fhir/MedicationRequest?identifier=http://erx.e-health.bg/ns/booklet-id&_include=*
Както се забелязава по горните два примера търсенето по идентификатор представлява комбинация GET атрибут включващ името на именованото пространство и стойността на идентификатора разделени със символа "|"
Стойността на именованото пространство се записва в ляво, а стойността на идентификатора от дясно. Ако някой от тях се пропусне то търсенето става само по единия атрибут.
Странициранетпо по подразбиране е 20 елемента от колекцията в която се търси. 

// ToDo: 
