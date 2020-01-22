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

After registering an user and logging in, we observed that we were issued a JSON Web Token (*JWT*).
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
From the header section we now know that the algorithm used to sign the JWT is RSA256 and the public certificate to validate the signature can be found at *https://filemanager.insomnihack.ch/jwk.json*.

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
Filename and base64 encoded filecontent are transmitted in the json request body.
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
# TODO public LINK 
Enough informations lets get hacking.

# Exploit
![letsGo](/img/letsgo.png)