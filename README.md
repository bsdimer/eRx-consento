# eRx-consento
eRx Consento - electronic medical prescription platform

This tutorial describes how to deal with the eRx platform of Consento.


In order to read the medication prescription data and to dispense the requested medications you should use the FHIR API which requires authentication. The mechanism uses token based authentication which can be tested in the following way:

##### Signup request
In order to signup with the system execute the following request:

```
curl --location --request POST 'https://consento-erx.kubocloud.io/erx-ui/api/entry' \
--header 'Content-Type: application/json; charset=UTF-8' \
--data-raw '{
    "user": {
        "data": [
            {
                "username": "test2",
                "firstName": "User",
                "lastName": "Test",
                "email": "test2@gbg.bg",
                "credentials": [
                    {
                        "type": "password",
                        "value": "12345"
                    }
                ]
            }
        ],
        "action": "CREATE",
        "pharmacyRegRq": {
            "address": "Address 1",
            "state": "Sofia",
            "district": "Sofia",
            "city": "Sofia",
            "vatNumber": "324234234",
            "phone": "08888888",
            "pharmacyNo": "NO11111111",
            "pharmacyName": "PHARMACY1"
        }
    },
    "auth": null
}
'
```

The information related to the pharmacy account must be supplied in the _pharmacyRegRq_ attribute.
The result will looks like:

```
{
    "user": {
        "result": [
            {
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
        ]
    }
}
```


##### Signin request

In order to access the system's resource you must get a Bearer token with a such query:

```
curl --location --request POST 'https://consento-erx.kubocloud.io/erx-ui/api/entry' \
--header 'Content-Type: application/json' \
--data-raw '{
    "user": {
        "data": [
            {
                "username": "test2",
                "credentials": [
                    {
                        "type": "password",
                        "value": "12345"
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
If the token is valid you will receive an response in the following format:

```
{
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
}
```

##### FHIR overview

The data model and the REST API of the eRx platform is based on FHIR. The latest version of the API is R4. 
In order to understand the API you must check the following links:

 FHIR summary - https://www.hl7.org/fhir/summary.html  
 FHIR Restfull API - https://www.hl7.org/fhir/http.html  
 FHIR Search operations - https://www.hl7.org/fhir/search.html  
 All the resource types - https://www.hl7.org/fhir/resourcelist.html  
        _* For the purpose of the eRx we will work only with few of them - MedicationRequest, Medication, MedicationDispense, 
        Practitioner, PractitionerRole, Patient, Organization, Location  

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
curl --location --request GET 'https://consento-erx.kubocloud.io/erx-ui/api/prescription?identifier=0000000308&hash=65ae447fa6202741be085f53fac0c0241c1e496a39604e735f0c04e5c96f95a6&created=2020-04-15T12%3A30%3A52.568%2B00%3A00' \
--header 'Content-Type: text/html'
```
This will return the web page in mobile friendly format. 
However when the same request is executed with header "Content-type: application-json+fhir" the 
result will be different:

```
curl --location --request GET 'https://consento-erx.kubocloud.io/erx-ui/api/prescription?identifier=0000000308&hash=65ae447fa6202741be085f53fac0c0241c1e496a39604e735f0c04e5c96f95a6&created=2020-04-15T12%3A30%3A52.568%2B00%3A00' \
--header 'Content-Type: application/json; charset=UTF-8'
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

The result in the Bundle represents a single prescription and can contains and can contain different kind of resources. 
Each entry in the result has a resourceType attribute which determines its type
The **MedicationRequest** resource is the actual prescription entry. The MedicationRequest resource has a _groupIdentifier_ attribute which is used
for a grouping purpose. **All the MedicationRequests which have the same _groupIdentifier_ are
prescribed in a single prescription**. Additionally each MedicationRequest has the _identifier_ with 
the value of the groupIdentifier. The detailed description of the MedicationRequest resource is:

- MedicationRequest.identifier - The identifier of the medical prescription.
The Bundle can have many resources of type MedicationRequest
and each of them can have the same identifier. All the MedicationRequest with the same identifier are actually entries in 
a single medical prescription. 
- MedicationRequest.status - The status of the MedicationRequest. The value of this attribute can be _active | on-hold | cancelled | completed | entered-in-error | stopped | draft | unknown_
In most cases when the prescription has been issued it will have status of _active_. More information about the statuses can be found here: https://www.hl7.org/fhir/valueset-medicationrequest-status.html
- MedicationRequest.statusReason - currently not used
- MedicationRequest.intent - currently not used
- MedicationRequest.category
MedicationRequest has a category attribute. The value of this attribute determines the type of the 
prescription. The type is a CodeableConcept which is a simple code relative to FHIR's CodeSystem or ValueSet.
All the values of the category attribute in the MedicationRequest can be found here "https://consento-erx.kubocloud.io/fhir/CodeSystem?name=Категории рецепти" 
The categories are specified in the concept property of the CodeSystem resource. Each category has code, display and description.
- MedicationRequest.priority - currently not used
- MedicationRequest.doNotPerform - currently not used 
- MedicationRequest.reported - currently not used 
- 

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
    GET https://consento-erx.kubocloud.io/fhir/MedicationRequest?requester.identifier=<identifier.value>&_include=* 
- search for prescriptions issued by a doctor specified by а part of the name.  
    GET https://consento-erx.kubocloud.io/fhir/MedicationRequest?requester.name=Поляков&_include=* 
- search for prescriptions issued to a patient  
    GET https://consento-erx.kubocloud.io/fhir/MedicationRequest?patient.identifier=<identifier.value>&_include=*

##### Reverse searches

The actual prescription in a PDF file representation which can be authenticated by a digital signature. This file can be 
attached as DocumentReference resource with relation to the MedicationRequest. The actual MedicationRequest did not have
a reference to the file. This is the reason why we should execute a reverse search. For example in order to find the 
PDF printed attachment of the prescription issue the following request:

GET https://consento-erx.kubocloud.io/fhir/MedicationRequest?patient.identifier=<identifier.value>&_include=*&_revinclude=*  

You will receive a Bundle with the same resources as it was in the previous requests but with one additional. The DocumentReference 
resource specifies the metadata of the prescription attached as file.

##### Medication dispense workflow

The actual dispense should make two changes into the database. The both changes can be executed in single transaction by the following way.

Create FHIR bundle with type of transaction:

```
curl --location --request POST 'http://consento-erx.kubocloud.io/fhir' \
--header 'Authorization: Bearer <token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "resourceType": "Bundle",
    "type": "transaction",
    "entry": [
        {
            "resource": {
                "resourceType": "MedicationRequest",
                "id": "11",
                "meta": {
                    "versionId": "11",
                    "lastUpdated": "2020-04-14T13:15:14.515+00:00"
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
            "request": {
                "method": "PATCH",
                "url": "MedicationRequest/11"
            }
        },
        {
            "fullUrl": "urn:uuid:aa4abd42-985b-461a-b15d-c15c04f5e634",
            "resource": {
                "resourceType": "MedicationDispense",
                "status": "completed",
                "medicationReference": {
                    "reference": "Medication/12",
                    "display": "AUGMENTIN FILM COATED TABLETS 875/125MG 14"
                },
                "subject": {
                    "reference": "Patient/13",
                    "display": "ГЕОРГИ ДИМИТРОВ"
                },
                "receiver": {
                    "display": "Димитър Димитров"
                },
                "quantity": {
                    "value": "2",
                    "unit": "опаковки"
                },
                "authorizingPrescription": {
                    "reference": "MedicationRequest/11"
                }
            },
            "request": {
                "method": "POST"
            }
        }
    ]
}'
```
// ToDo: Example with transaction including PATCH update 

Thus will update the status of the MedicationRequests from _active_ to _completed_ and will create
a dispense for each of the requests in a single atomic transaction. This means that all the MedicationRequests and MedicationDispenses can be included in this single request.
The main thing here is the MedicationDispense resource. It represents the actual dispension(fulfill) of the medication. The full resource description can be found here https://www.hl7.org/fhir/medicationdispense.html
 
- MedicationDispense status can be one of the following values _"preparation | in-progress | cancelled | on-hold | completed | entered-in-error | stopped | declined | unknown
"_.
- "subject" is a reference to a Patient object. The reference is not mandatory so only the display property can be specified with the names of the subject written on the prescription.
- "receiver" is also a reference to the person which is the actual buyer of the medication. It could be a different person than the patient. For that reason we can specify only the names in the display attribute. The receiver property is also not mandatory.
- "quantity" is the amount of medication dispensed. It is a FHIR SimpleQuantity(https://www.hl7.org/fhir/datatypes.html#Quantity) object and can have value and units. In our case 2 peaces(опаковки).
- "authorizingPrescription" is a reference to the MedicationRequest. This property is mandatory for the data integrity of the eRx database.
   
##### Update with patch operation

Sometimes the preparation of the medication is more time consuming task. In these circumstances the FHIR model permits
to create a MedicationDispense with different state which can be updated after along with the process of the preparation of the medication.

```
curl --location --request PATCH 'https://consento-erx.kubocloud.io/fhir/MedicationRequest/13' 
--header 'Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOjU5NSwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJyb2xlcyI6W3sicm9sZSI6IlJPTEVfVVNFUiIsImNsaWVudCI6Imxtc3dlYiJ9LHsicm9sZSI6IlJPTEVfRE9DVE9SIiwiY2xpZW50IjoibG1zd2ViIn1dLCJpc3MiOiJodHRwczovL2VsbWVkaWtvLmNvbSIsImdpdmVuX25hbWUiOiLQkNC70LXQutGB0LDQvdC00YrRgCIsImNsaWVudF9pZCI6Imxtc3dlYiIsInBpY3R1cmUiOm51bGwsInNjb3BlIjoiKi8qLioiLCJwaG9uZV9udW1iZXIiOm51bGwsImV4cCI6MTU4NjkwMDQwNSwiZmFtaWx5X25hbWUiOiLQkNC70LXQutGB0LjQtdCyIiwiZW1haWwiOiJhbGV4YW5kZXIuYWxleGlldkBlLWhlYWx0aC5iZyIsInVzZXJuYW1lIjoiYWxleEBzYXQifQ.3sZOw5GbNtbGJ3Zav_Wi_ZouSj_d7E_CVuluVJR1d_1l6qflKBbtTd98T0fynhKLICHLySHnSNeZ7Xj3ssYKKA' 
--header 'Content-Type: application/json-patch+json' 
--header 'Content-Type: application/json' 
--data-raw '[
{ "op": "replace", "path": "/status", "value": "completed" }
]'
```

//ToDo: 

##### Additional resources
- eRx-ui Swagger - https://consento-erx.kubocloud.io/erx-ui/swagger-ui/
