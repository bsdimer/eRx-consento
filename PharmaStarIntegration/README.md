# eRx <-> PharmaStar
eRx Consento - интеграция с ФармаСтар

Този документ описва интеграцията на модела на данните на eRx Consento с модела на данните на приложението ФармаСтар

### Описание на процеса на вземане на данни

Четенето на данни от регистъра изисква вход в системата посредством потребителско име и парола. 
Описанието за това как да се извърши регистрацията и взимане на токен може да бъде намеререно в предходния документ.
// ТоДо: линк към предходния документ

#### Вземане на рецепта по идентифиактор

```
curl --location --request GET 'https://stagingerx.e-health.bg/fhir/erx-phs/api/prescription?identifier=ERXN20200423000002%D0%90' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjZklBRE5feHhDSm1Wa1d5Ti1QTlhFRXZNVVdzMnI2OEN4dG1oRUROelhVIn0.eyJleHAiOjE1OTA1NzE3MTksImlhdCI6MTU5MDQ4NTMxOSwianRpIjoiNmJlNjI4NzMtMmNiMC00N2VjLTk0ZTktMjk1NWYyODZkZjBhIiwiaXNzIjoiaHR0cDovL2NvbnNlbnRvLWVyeC1rZXljbG9hay5rdWJvY2xvdWQuaW8vYXV0aC9yZWFsbXMvZS1oZWFsdGgtYmciLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiZjBjMDQ4MWUtM2UwZC00YTljLWI1YjItMzVmOTczMzEyNTE3IiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiZXJ4LXJlc291cmNlLXNlcnZlciIsInNlc3Npb25fc3RhdGUiOiI4ZmIxYTgyOC1kNWUwLTRkYmUtYTg5OS0zNmFjNDZiNDcyMTMiLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwicGhhcm1hY2lzdCIsInVtYV9hdXRob3JpemF0aW9uIiwidXNlciJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfSwiZXJ4LXJlc291cmNlLXNlcnZlciI6eyJyb2xlcyI6WyJwaGFybWFzdGFyX3VzZXIiLCJlcnhfdXNlciJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFtZSI6ImFzdGVzdDEgYXN0ZXN0MSIsInByZWZlcnJlZF91c2VybmFtZSI6ImFzdGVzdDEwMCIsImdpdmVuX25hbWUiOiJhc3Rlc3QxIiwiZmFtaWx5X25hbWUiOiJhc3Rlc3QxIiwiZW1haWwiOiJ1c2VyMTAwQGFwcGxzcy5jb20ifQ.kmFVKJk8E7051fve594e2jqLwVi8IwvkecZDwLolhg9faAqlx_kDmYYehqddySdy1ooH2139Pj34SA6OZK-rVwZygyA4ZwfY5kckWcc4z7UgJyuwkR9R6s4HZhz4ZnSDRhBFOhkV9zyxJy-f96kdkMrVg_Ajn9dRdpu5DbX5-acYKJ2dR_Ykyt7TpoktTBUGlzjh7i0mb29UUn1K5Q0Sx0KV7ILXnOe7wKHnmbjwa1CE4zpA70aOpDc-W-6Io5m4J-VsIK008fxcG63PzctCl6OIOQAMEssfsw8uAoaVClEdg948ygBxhLJuPKwBvq2sEO2HT9q8Vryc1_1FzPmxNA'
```

Резултата от търсенето е масив от рецепти. Стойността е масив тъй като идентификатора сам по себе си в сложна стойност 
интегрираща в себе си множество данни. Така например рецептата по НЗОК по бланка 5А притежава собствен групов идентификатор
който позволява да бъдат намерени всички рецепти по тази бланка. В най-общия случай търсенете ще бъде по идентификатор на конректната
рецепта и резултата ще бъде масив от една рецепта. 

```
[
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
]
```

#### Изписване на лекарсто по рецепта

