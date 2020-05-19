Implemented and installed on staging. Please follow the following steps:

Create user:
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
        "action": "CREATE",
        "data": [
            {
                "credentials": [
                    {
                        "type": "password",
                        "value": "sd:Youroery8u"
                    }
                ],
                "email": "user@email.com",
                "firstName": "Triphon",
                "lastName": "Test",
                "username": "user-test"
            }
        ],
        "pharmacyRegRq": {
            "address": "Test street 9",
            "city": "Sofia",
            "district": "Sofia",
            "pharmacistFamilyName": "Doe",
            "pharmacistFirstName": "John",
            "pharmacyName": "TEST PHARMACY",
            "pharmacyNo": "NO12345678",
            "phone": "+359 222222",
            "vatNumber": "123456789"
        }
    }
}
```

Get the user id returned from response json: user.result[0].id. E.g: 96a5cc60-0f65-453f-a9c7-697a31c445e9

Login with Pharma Star Power User and obtain the token:

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
Token can be found under user.token

Use the power user token to fix the newly created user:

```
Request method: GET
Request URI:    https://stagingerx.e-health.bg/fhir/erx-phs/api/fix-user/<PUT_USER_ID_HERE>
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Headers:                Authorization=Bearer <PUT_PHARMA_STAR_POWER_USER_TOKEN_HERE>
                                Accept=*/*
                                Content-Type=application/json; charset=UTF-8
Cookies:                <none>
Multiparts:             <none>
Body:                   <none>
```
For success you will receive 201 HttpStatus. If there is no such user you will get the 404 status. If you have no rights to perform such action: 403. In case of unexpected error you will receive an ISE status: 500
