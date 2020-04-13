# eRx-consento
eRx Consento - electronic medical prescription platform

This tutorial describes how to deal with the eRx platform of Consento.


In order to read the medication prescription data and to dispense the requested medications you should use the FHIR API which requires authentication. The mechanism uses token based authentication which can be tested in the following way:

###### Signup request

`curl --location --request POST 'http://consento-erx.kubocloud.io/erx-ui/api/entry' \
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
'`

###### Signin request

`curl --location --request POST 'http://consento-erx.kubocloud.io/erx-ui/api/entry' \
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
}'`

You will receive the bearer token which is used against the all authenticated endpoints. 

###### Whoami request

In order to check the validity and the information behind the issued token, just execute the following request.

`curl --location --request POST 'http://consento-erx.kubocloud.io/erx-ui/api/auth-entry' \
--header 'Authorization: Bearer <access token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "auth": {
        "whoAmI": true
    }
}'`



