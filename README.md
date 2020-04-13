# eRx-consento
eRx Consento - electronic medical prescription platform

This tutorial describes how to deal with the eRx platform of Consento.


In order to read the medication prescription data and to dispense the requested medications you should use the FHIR API which requires authentication. The mechanism uses token based authentication which can be tested in the following way:

##### Signup request
In order to signup with the system execute the following request:

```
curl --location --request POST 'https://consento-erx.kubocloud.io/erx-ui/api/entry' \
--header 'Content-Type: application/json' \
--data-raw '{
  "user": {
    "data": [
      {
        "username": "testuser",
        "firstName": "Test",
        "lastName": "Person",
        "email": "test@test.com",
        "credentials": [
          {
            "type": "password",
            "value": "qweQWE123\\u0021@#"
          }
        ]
      }
    ],
    "action": "CREATE",
    "pharmacyRegRq": {
      "address": "test address",
      "state": "Sofia",
      "district": "Sofia",
      "city": "Sofia",
      "vatNumber": "3947384789",
      "phone": "088888888",
      "pharmacyNo": "NO12345678",
      "pharmacyName": "TEST PHARMACY"
    }
  }
}
'
```

The information related to the pharmacy account must be supplied in the _pharmacyRegRq_ attribute.

##### Signin request

```
curl --location --request POST 'https://consento-erx.kubocloud.io/erx-ui/api/entry' \
--header 'Content-Type: application/json' \
--data-raw '{
    "user": {
        "data": [
            {
                "username": "testuser",
                "credentials": [
                    {
                        "type": "password",
                        "value": "qweQWE123\\u0021@#"
                    }
                ]
            }
        ],
        "action": "LOGIN"
    }
}'
```

You will receive the bearer token which is used against the all authenticated endpoints. 

##### Whoami request

In order to check the validity and the information behind the issued token, execute the following request:

```
curl --location --request POST 'https://consento-erx.kubocloud.io/erx-ui/api/auth-entry' \
--header 'Authorization: Bearer <access token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "auth": {
        "whoAmI": true
    }
}'
```

##### FHIR overview

The data model and the REST API of the eRx platform is based on FHIR. The latest version of the API is R4. 
In order to understand the API you must check the following links:

 FHIR summary - https://www.hl7.org/fhir/summary.html  
 FHIR Restfull API - https://www.hl7.org/fhir/http.html  
 FHIR Search operations - https://www.hl7.org/fhir/search.html  
 All the resource types - https://www.hl7.org/fhir/resourcelist.html  
        _* For the purpose of the eRx we will work only with few ot them_  

##### eRX workflow

When we talk about the eRx there are two distinct business workflows. The first workflow 
involves the creation of the prescription (a MedicationRequest resource in the FHIR nomenclature)
and the second one is the update the status of this prescription along with creation of dispense record.
The first workflow is executed in the software of the doctors and the second one will be performed by the pharmacies.

##### Pharmacies workflow

There various searches that can be performed in order to find the prescription data. 

###### Search for prescription by single atomic unauthenticated request
The main reason for the existence of unauthenticated search mechanism for gathering the 
data of the single prescription is the opportunity to use your mobile device with QRCode scanning functionality.
When you scan a QRCode printed on the prescription blank with your mobile phone it will redirect you directly
to the prescription.

//ToDo: should put a single QRCode for the demo purpose.

The actual execution from the browser will be as follows:  

```
curl --location --request GET 'https://consento-erx.kubocloud.io/erx-ui/prescription?identifier=0000000308' \
--header 'Content-Type: text/html' 
```
This will return the web page in mobile friendly format. 
However when the same request is executed with header "Content-type: application-json+fhir" the 
result will be different:

```
curl --location --request GET 'https://consento-erx.kubocloud.io/erx-ui/prescription?identifier=0000000308' \
--header 'Content-Type: application/json+fhir'
```

This will return the JSON formatted string of the prescription in the format of FHIR Bundle. 

###### What is FHIR Bundle

FHIR Bundle is a container for a collection of resources. In the case of search for a single prescription 
all the resources returned in that Bundle will be related to each other.
```
{
  "resourceType": "Bundle",
  "id": "33ccbe4e-e9b7-41f5-ba0e-a9d163fb960c",
  "meta": {
    "lastUpdated": "2020-04-13T13:04:49.039+00:00"
  },
  "type": "searchset",
  "total": 1,
  "link": [
    {
      "relation": "self",
      "url": "http://consento-erx-keycloak.kubocloud.io/fhir/MedicationRequest?_include=*&identifier=0000000308"
    }
  ],
  "entry": [
    {
      "fullUrl": "http://consento-erx-keycloak.kubocloud.io/fhir/MedicationRequest/11",
      "resource": {
        "resourceType": "MedicationRequest",
        "id": "11",
        "meta": {
          "versionId": "1",
          "lastUpdated": "2020-04-13T12:56:18.231+00:00"
        },
        "identifier": [
          {
            "system": "http://terminology.elmediko.com/CodeSystem/receipt-id",
            "value": "0000000308"
          }
        ],
        "status": "active",
        "intent": "proposal",
        "category": [
          {
            "coding": [
              {
                "system": "http://terminology.elmediko.com/CodeSystem/medication-request-category",
                "code": "white"
              }
            ]
          }
        ],
        "medicationReference": {
          "reference": "Medication/12",
          "display": "AUGMENTIN FILM COATED TABLETS 875/125MG 14 - GSK - FILM COATED TABLETS"
        },
        "subject": {
          "reference": "Patient/13",
          "display": "ГЕОРГИ ДИМИТРОВ"
        },
        "requester": {
          "reference": "PractitionerRole/5",
          "display": "ГППМП - МКЦ \"Моят лекар\""
        },
        "recorder": {
          "reference": "Practitioner/4",
          "display": "д-р Иван Поляков"
        },
        "groupIdentifier": {
          "value": "0000000308"
        },
        "dosageInstruction": [
          {
            "text": "Да се приема по време на хранене",
            "doseAndRate": [
              {
                "doseQuantity": {
                  "value": 2,
                  "unit": "tablets"
                },
                "rateQuantity": {
                  "value": 12,
                  "unit": "daily"
                }
              }
            ],
            "maxDosePerAdministration": {
              "value": 7,
              "unit": "days"
            }
          }
        ]
      },
      "search": {
        "mode": "match"
      },
      "response": {
        "status": "201 Created",
        "etag": "W/\"1\""
      }
    },
    {
      "fullUrl": "http://consento-erx-keycloak.kubocloud.io/fhir/PractitionerRole/5",
      "resource": {
        "resourceType": "PractitionerRole",
        "id": "5",
        "meta": {
          "versionId": "1",
          "lastUpdated": "2020-04-12T14:51:00.679+00:00"
        },
        "practitioner": {
          "reference": "Practitioner/4",
          "type": "Practitioner",
          "display": "д-р Иван Поляков"
        },
        "organization": {
          "reference": "Organization/2",
          "type": "Organization",
          "display": "ГППМ - МКЦ Моят лекар"
        },
        "code": [
          {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/practitioner-role",
                "code": "doctor",
                "display": "Doctor"
              }
            ]
          }
        ],
        "specialty": [
          {
            "coding": [
              {
                "system": "http://terminology.elmediko.com/CodeSystem/doctor-speciality-nhif",
                "code": "00",
                "display": "Общопрактикуващ лекар"
              }
            ]
          }
        ],
        "location": [
          {
            "reference": "Location/3",
            "type": "Location",
            "display": "клон Плевен"
          }
        ]
      },
      "search": {
        "mode": "include"
      },
      "response": {
        "status": "201 Created",
        "etag": "W/\"1\""
      }
    },
    {
      "fullUrl": "http://consento-erx-keycloak.kubocloud.io/fhir/Medication/12",
      "resource": {
        "resourceType": "Medication",
        "id": "12",
        "meta": {
          "versionId": "1",
          "lastUpdated": "2020-04-13T12:56:18.231+00:00"
        },
        "text": {
          "status": "generated",
          "div": "<div xmlns=\"http://www.w3.org/1999/xhtml\"><div class=\"hapiHeaderText\">AUGMENTIN FILM COATED TABLETS 875/125MG 14</div></div>"
        },
        "extension": [
          {
            "url": "http://terminology.elmediko.com/Extension/medication-brand-name",
            "valueString": "AUGMENTIN"
          },
          {
            "url": "http://terminology.elmediko.com/Extension/medication-brand-short-name",
            "valueString": "AUGMENTIN"
          },
          {
            "url": "http://terminology.elmediko.com/CodeSystem/medication-icd10-code",
            "valueCodeableConcept": {
              "coding": [
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/icd10-codes",
                  "code": "N10"
                },
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/icd10-codes",
                  "code": "N11.0"
                },
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/icd10-codes",
                  "code": "N11.1"
                },
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/icd10-codes",
                  "code": "N11.8"
                }
              ]
            }
          }
        ],
        "identifier": [
          {
            "type": {
              "coding": [
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/medication-identifier-type",
                  "code": "SAT_PT_ID"
                }
              ]
            },
            "value": "9715"
          },
          {
            "type": {
              "coding": [
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/medication-identifier-type",
                  "code": "SAT_PT_CODE"
                }
              ]
            },
            "value": "PT000277"
          },
          {
            "system": "http://terminology.elmediko.com/CodeSystem/medication-codes-who-atc",
            "value": "J01CR02"
          },
          {
            "system": "http://terminology.elmediko.com/CodeSystem/medication-codes-ephmra-atc",
            "value": "J01C01"
          },
          {
            "system": "http://terminology.elmediko.com/CodeSystem/medication-codes-nfc",
            "value": "ABC"
          },
          {
            "system": "http://terminology.elmediko.com/CodeSystem/medication-codes-nhif",
            "value": "JF381"
          }
        ],
        "code": {
          "coding": [
            {
              "system": "http://terminology.elmediko.com/CodeSystem/medication-codes-sat",
              "code": "PT000277",
              "display": "AUGMENTIN FILM COATED TABLETS 875/125MG 14"
            }
          ]
        },
        "manufacturer": {
          "display": "GSK"
        },
        "form": {
          "coding": [
            {
              "system": "http://terminology.elmediko.com/CodeSystem/medication-form",
              "code": "FILM COATED TABLETS",
              "display": "FILM COATED TABLETS"
            }
          ]
        },
        "amount": {
          "numerator": {
            "value": 14
          }
        }
      },
      "search": {
        "mode": "include"
      },
      "response": {
        "status": "201 Created",
        "etag": "W/\"1\""
      }
    },
    {
      "fullUrl": "http://consento-erx-keycloak.kubocloud.io/fhir/Patient/13",
      "resource": {
        "resourceType": "Patient",
        "id": "13",
        "meta": {
          "versionId": "1",
          "lastUpdated": "2020-04-13T12:56:18.231+00:00"
        },
        "text": {
          "status": "generated",
          "div": "<div xmlns=\"http://www.w3.org/1999/xhtml\"><div class=\"hapiHeaderText\">ГЕОРГИ ТОДОРОВ <b>ДИМИТРОВ </b></div><table class=\"hapiPropertyTable\"><tbody><tr><td>Identifier</td><td>7611064085</td></tr></tbody></table></div>"
        },
        "extension": [
          {
            "url": "http://terminology.elmediko.com/Extension/nhif-branch",
            "valueString": "08"
          },
          {
            "url": "http://terminology.elmediko.com/Extension/nhif-region",
            "valueString": "15"
          }
        ],
        "identifier": [
          {
            "use": "official",
            "type": {
              "coding": [
                {
                  "system": "http://terminology.elmediko.com/CodeSystem/person-identifier-codes/v1-bg",
                  "code": "NNBGR"
                }
              ]
            },
            "value": "7********5"
          }
        ],
        "active": true,
        "name": [
          {
            "use": "official",
            "family": "ДИМИТРОВ",
            "given": [
              "ГЕОРГИ",
              "ТОДОРОВ"
            ]
          }
        ]
      },
      "search": {
        "mode": "include"
      },
      "response": {
        "status": "201 Created",
        "etag": "W/\"1\""
      }
    }
  ]
}
```
 
This example contains of few resources need for the visualisation of the receipt. 

###### How to interpret the prescription

The Bundle is the resource which can contain different kind of resource. Each entry in the result has a resourceType attribute which determines its type
The **MedicationRequest** resource is the actual prescription entry. Each prescription has one or more entries for medication and the 
instruction about the dosage. The MedicationRequest resource has a _groupIdentifier_ attribute which is used
for a grouping purpose. All the MedicationRequests which have the same groupIdentifier are
prescribed in a single prescription. Additionally each MedicationRequest has the identifier with 
the value of the groupIdentifier.  
  
MedicationRequest has a category attribute. The value of this attribute determines the type of the 
prescription. The type is a CodeableConcept which is a simple code relative to FHIR's CodeSystem or ValueSet.
All the values of the category attribute in the MedicationRequest can be found here:
```
curl --location --request GET 'https://consento-erx.kubocloud.io/fhir/CodeSystem?name=%D0%9A%D0%B0%D1%82%D0%B5%D0%B3%D0%BE%D1%80%D0%B8%D0%B8%20%D1%80%D0%B5%D1%86%D0%B5%D0%BF%D1%82%D0%B8' \
    --header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjZklBRE5feHhDSm1Wa1d5Ti1QTlhFRXZNVVdzMnI2OEN4dG1oRUROelhVIn0.eyJleHAiOjE1ODY4MDYyOTIsImlhdCI6MTU4Njc3MDI5MiwianRpIjoiNzRlYTM1MjUtZWNkZi00MTI2LWJiNzQtNGFlZGZiOGExYWM0IiwiaXNzIjoiaHR0cDovL2NvbnNlbnRvLWVyeC1rZXljbG9hay5rdWJvY2xvdWQuaW8vYXV0aC9yZWFsbXMvcXVhcmt1cyIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiIwYzI0ZjhhZS00NWExLTRmOTEtYTZiMi0wMGZmZWFmZjk3OTkiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJiYWNrZW5kLXNlcnZpY2UiLCJzZXNzaW9uX3N0YXRlIjoiNTBhZjEwMmItMjJhNS00NmM4LWIwZWUtYjFkMDAyYjY0ODk1IiwiYWNyIjoiMSIsInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJvZmZsaW5lX2FjY2VzcyIsInVtYV9hdXRob3JpemF0aW9uIiwidXNlciJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFtZSI6IlRlc3QgUGVyc29uIiwicHJlZmVycmVkX3VzZXJuYW1lIjoidGVzdHVzZXIiLCJnaXZlbl9uYW1lIjoiVGVzdCIsImZhbWlseV9uYW1lIjoiUGVyc29uIiwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIn0.cWT-wpmdrS6kuhuePJSvIsAHtdP2H1WalJ8zlJbIoNO7-RZ-zE6GqGw_p-WgI6thwgIv-TE3XTnLeeIYxRB-OEGn6mqqh-iSwSZbvHMuWf7IZsUJDwtkA8z2B37RlCJ9MqyibwOo3Bjs8rBpxxKXucLVEf_Q8SmRSqw3kFJlwTORy-MtXyE7yrbdhTbsW7wgK0YDOZUcX3i8_R6hQCocLZAE7G504MpcNoWKBdBDVAIbjIJ9R18A_G2Qi73XLpEge8xFJ3pWhaMw-bNfr7CFlGhJR5-LHkH_21kzWNVIeuRSbH2pTtDep7YwoiRLttjaSWMMkjGapWW_hI4mitpQpQ'
```
This is simple GET request for `https://consento-erx.kubocloud.io/fhir/CodeSystem?name=Категории рецепти` 
The categories are specified in the concept property of the CodeSystem resource. Each category has code, display and description.

The MedicationRequest has different relations to other resources. In the common case these are
**Medication**, **Patient**, **PractitionerRole**. 

**Medication** resource is the actual medication which is prescribed. Each medication has various identifiers
related to the codes of the medication - ATC code, SAT_code and other. 
Additionally the Medication resource in the eRx database has an extensions which a formally
a way to extend the data model beyond the FHIR specifications.

The **PractitionerRole** resource describes the doctor which issued the prescription.
This resource consist of references to the **Organization** and the **Location**
where the prescription has been issued. The attribute named _practitioner_ refers resource 
of type **Practitioner** which is the actual doctor. The _speciality_ attribute describes the doctor's
specialities. The _code_ attribute shows his position in the organization.
The resource PractitionerRole along with the Practitioner has identifier attribute which
determines the doctor's UIN.

The **Patient** resource describes the patient's information. The national identifier 
of the resource can be found here. The Bulgarians national identifier is tagged with code 'NNBGR' which means 'ЕГН'.
 
##### Most useful search requests

- search for prescriptions issued by a doctor specified by UIN.  
    https://consento-erx.kubocloud.io/fhir/MedicationRequest?requester.identifier=<identifier.value>&_include=* 
- search for prescriptions issued by a doctor specified by а part of the name.  
    https://consento-erx.kubocloud.io/fhir/MedicationRequest?requester.name=Поляков&_include=* 
- search for prescriptions issued to a patient  
    https://consento-erx.kubocloud.io/fhir/MedicationRequest?patient.identifier=<identifier.value>&_include=*

##### Medication dispense workflow


##### Additional resources
- eRx-ui Swagger - https://consento-erx.kubocloud.io/erx-ui/swagger-ui/
