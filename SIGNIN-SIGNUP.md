# eRx-consento
eRx Consento - electronic medical prescription platform

This tutorial describes how to deal with the eRx platform of Consento.


In order to read the medication prescription data and to dispense the requested medications you should use the FHIR API which requires authentication. The mechanism uses token based authentication which can be tested in the following way:

#### Signup request
Implemented and installed on staging. Please follow the following steps:

1. Login with Pharma Star Power User and obtain the token:
```
Request method:	POST
Request URI:	https://stagingerx.e-health.bg/erx-ui/api/entry
Proxy:			<none>
Request params:	<none>
Query params:	<none>
Form params:	<none>
Path params:	<none>
Headers:		Accept=*/*
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
        "action": "LOGIN",
    }
}
```
2. Token can be found under `user.token`
3. Use the power user token to create the user:
4. Create user:
```
Request method: POST
Request URI:    https://stagingerx.e-health.bg/erx-ui/api/auth-entry
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Headers:                Authorization=Bearer <PUT YOUR TOKEN HERE>
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

6. You always will receive `200` Http Status, however if is ok you will receive the user details in response. You can check the presence of `user.result.[0].id`

```
{
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
                "disableableCredentialTypes": [
                    
                ],
                "requiredActions": [
                    
                ],
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
                }
            }
        ]
    },
    "auth": {
        
    }
}
```

#### Signin request

In order to access the system's resource you must get a Bearer token with a such query:

```
curl --location --request POST 'https://stagingerx.e-health.bg/erx-ui/api/entry' \
--header 'Content-Type: application/json' \
--data-raw '{
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
}'
```

You will receive the bearer token which is used against the all authenticated endpoints. 

#### Whoami request

In order to check the validity and the information behind the issued token, execute the following request:

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
