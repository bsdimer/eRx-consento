# eRx <-> PharmaStar
eRx Consento - интеграция с ФармаСтар

Този документ описва интеграцията на модела на данните на eRx Consento с модела на данните на приложението ФармаСтар

#### Описание на процеса на вземане на данни

Четенето на данни от регистъра изисква вход в системата посредством потребителско име и парола. 
Описанието за това как да се извърши регистрацията и взимане на токен може да бъде намеререно в предходния документ.
// ТоДо: линк към предходния документ

След вземането на токен стартирайте търсене за обект от тип MedicationRequest чрез следната примертна заявка.

 GET https://consento-erx.kubocloud.io/fhir/MedicationRequest?identifier=0000000308&_include=*
 
като параметърът identifier е номера(идентификатор) на рецептата, опицията _include=* указва искането в резултата 
да бъдат добавени всички ресурси релативни към текущата рецепта. 
В полученият резултат се съдържат всички необходими данни за попълването на модела на ФармаСтар. 
В него може има обекти от няколко различни типа: **MedicationRequest, Medication, PractitionerRole, Patient, Encounter.**

- MedicationRequest е ресурсът, който указва един ред от предписанието в рецептата. Ако изписаната рецепта има повече от едно лекарство, то в резултата ще присъстват повече от един ресурс от тип MedicationRequest.
Всички видове рецепти, както и протокола по НЗОК представляват ресурси от тип MedicationRequest. Атрибута който ги отличава е MedicationRequest.category.
Категориите MedicationRequest са представени под формата на CodeableConcept атрибут (https://www.hl7.org/fhir/datatypes.html#CodeableConcept).
Кодовете на на съответните рецепти могат да бъдат разгледани в кодовата система "Категории рецепти" ( GET https://consento-erx.kubocloud.io/fhir/CodeSystem/medication-request-category-bg). Те биват: 
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
    "resourceType": "MedicationRequest"
    "identifier": [
        {
            "system": "http://erx.e-health.bg/ns/prescription-id"
            "value": "0000000308"
        },
        {
            "system": "http://erx.e-health.bg/ns/nhif-protocol-id"
            "value": "0004401232"
        }
    ]
    ...
} 
```
В горния пример ресурсът има няколко идентификатора, първият е идентификатора на рецептата а втория е идентификатора на 
протокола по НЗОК към който е издадена конкретната рецепта.
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
            "system": "http://erx.e-health.bg/ns/rhif-codes",
            "value": "2200000010" <-- Това е код по РЗОК на лечебното заведение
        },
        {
            "system": "http://erx.e-health.bg/ns/practice-no",
            "value": "1202340392" <-- Номер на практика
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
                    "system": "http://terminology.e-health.bg/CodeSystem/nhif-doctor-specialities",
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
            "display": "клон Плевен" <-- Локация в която е издадена рецептата
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
        }
    ],
    "identifier": [
        {
            "use": "official",
            "system": "http://erx.e-health.bg/ns/nnbgr",
            "value": "7703022402" <-- ЕГН на пациента
        },
        {
            "system": "http://erx.e-health.bg/ns/ss",
            "value": "100000000" <-- Номер на социална осигуровка
        },
        {
            "system": "http://erx.e-health.bg/ns/tpr",
            "value": "100000002" <-- Личен номер на чужденец ЛНЧ
        },
        {
            "system": "http://erx.e-health.bg/ns/prc",
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
- **PartType** - Тази характеристика на рецептата се отнася за рецепти издадени по НЗОК.
Тъй като в модела на FHIR липсва аналогия на отрязък на рецепта, в модела на eRx в въведено
разширение Extension 


```
public class Prescription : IData
    {
        public int Id { get; set; }
        public string Barcode { get; set; } = string.Empty;
        public int PrescriptionNo { get; set; } <- MedicationRequest.identifier[system==http://erx.e-health.bg/ns/prscr-id].value
        public DateTime PrescriptionDate { get; set; } <- MedicationRequest.authoredOn
        public int ProtocolNo { get; set; } <- MedicationRequest.basedOn(identifier[system==http://erx.e-health.bg/ns/prscr-id].value)*
        public DateTime ProtocolDate { get; set; } <- MedicationRequest.basedOn(MedicationRequest.authoredOn)**
        public int PrescriptionBookletNo { get; set; }
        public int AmbSheetNo { get; set; }
        public Doc Doctor { get; set; } = new Doc();
        public Patient Patient { get; set; } = new Patient();

        public PartType Part { get; set; } = PartType.PartNone;

        #region ... VETERAN ...

        public string VeteranBookledNo { get; set; } = string.Empty;
        public DateTime VeteranBookledDate { get; set; }
        public string TerritorialExpertMedicalCommissionBookledNo { get; set; } = string.Empty;
        public DateTime TerritorialExpertMedicalCommissionBookledDate { get; set; }

        #endregion

        #region ... DISPENSION ...

        public DispensionState State { get; set; }
        public bool IsDeleted { get; set; }

        public DateTime DispensingDate { get; set; }
        public PaymentWay WayOfPayment { get; set; }
        public Pharmacist Pharmacist { get; set; }
        public Pharmacy Pharmacy { get; set; }

        #endregion

        public List<PrescriptionRow> PrescriptionRows { get; set; } = new List<PrescriptionRow>();
       
	   //...
    }

public class Doc
    {
        public string SoftwareCode { get; set; } = string.Empty;
        public string RhifCode { get; set; } = string.Empty;
        public string PracticeNo { get; set; } = string.Empty;
        public string Uin { get; set; } = string.Empty;
        public string Specialty { get; set; } = string.Empty;
        public string Name { get; set; } = string.Empty;
        public string Phone { get; set; } = string.Empty;
    }
	
public class Patient
    {
        public string FirstName { get; set; } = string.Empty;
        public string MiddleName { get; set; } = string.Empty;
        public string LastName { get; set; } = string.Empty;
        public int Age { get; set; }
        public string Egn { get; set; } = string.Empty;
        public string Lnch { get; set; } = string.Empty;
        public string Ssn { get; set; } = string.Empty;
        public string PersId { get; set; } = string.Empty;
        public string Address { get; set; } = string.Empty;
        public string CertificateCountryCode { get; set; } = string.Empty;
        public DateTime CertificateIssueDate { get; set; }
        public DateTime CertificateValidFrom { get; set; }
        public DateTime CertificateValidTo { get; set; }
        public string CertificateNumber { get; set; } = string.Empty;
        public string CertificateType { get; set; } = string.Empty;
        public DateTime BirthDate { get; set; }
        public string Sex { get; set; } = string.Empty;
        public bool MaternityFlag { get; set; } = false;
        public bool PregnancyFlag { get; set; } = false;
    }
	
public class Pharmacist
    {
        public string Uin { get; set; } = string.Empty;
        public string Name { get; set; } = string.Empty;        
    }
	
public class Pharmacy
    {
        public string Name { get; set; } = string.Empty;
        public string Number { get; set; } = string.Empty;
        public string SoftwareCode { get; set; } = string.Empty;        
    }
	
public enum PartType
    {
        PartA,
        PartB,
        PartC,
        PartNone = 2147483647
    }

public enum PaymentWay
    {
        Pharmacy = 0,
        Nhif = 1
    }

public enum DispensionState
    {
        Downloaded,
        Finished
    }

```
