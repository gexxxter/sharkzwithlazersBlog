---
title: "SharkyCTF 2020 - Logs in! Part 2"
date: 2020-05-12T13:51:40+02:00
author: "nscur0"
description: "Writeup of the challenge 'Logs in! Part 2'@SharkyCTF 2020"
---

![Challenge description](/img/2020-05-12_sharkyctf_logs-in-part2_challenge.png)

Building upon [part 1](https://ctftime.org/task/11536) of this challenge, we continue to gather useful insights
about how the target application works by using the enabled Symfony development tools. This time, we're using these
insights to hack our way into a database that's used by a backend service which we normally wouldn't be able to reach.

Via `/_profiler/open?file=<FILE_PATH>`, we can disclose the content of any file relative to the project's root.
This has been exploited in part 1 to identify the hidden `/debug` endpoint in `src/Controller/MainController.php`:

![Source code of MainController.php](/img/2020-05-12_sharkyctf_logs-in-part2_maincontroller.png)

`MainController`'s  primary function though, is to display request logs. Users are supposed to request the page
with the same HTTP method they're wishing to see the logs for: 

![Logs](/img/2020-05-12_sharkyctf_logs-in-part2_logs.png)

Looking at the code, we noticed that the page actually calls an external API at `10.0.142.5`. The method to fetch logs
for is determined by calling `$request-getMethod()`, where `$request` is an instance of Symfony's `Request` class. 
The documentation of [`Request#getMethod`](https://github.com/symfony/symfony/blob/5.0/src/Symfony/Component/HttpFoundation/Request.php#L1228) turned out to be quite helpful:

> If the X-HTTP-Method-Override header is set, and if the method is a POST,
> then it is used to determine the "real" intended HTTP method.

We couldn't tamper with our request's actual method, as the server would only process valid HTTP methods.
Using `X-HTTP-Method-Override` however, we were able to call the API at `10.0.142.5` with arbitrary `method` values.
This was trivially validated by `POST`ing against `/e48e13207341b6bffb7fb1622282247b`, providing the additional header `X-HTTP-Method-Override: PUT`:

![Logs with modified method](/img/2020-05-12_sharkyctf_logs-in-part2_logs-modified.png)

Our first reflex was to test for SQL injection, which turned out to be the right choice. Providing the value `PUT' or 1=1; -- ` resulted in all log entries being displayed:

![SQL injection proof of concept](/img/2020-05-12_sharkyctf_logs-in-part2_logs-sqli.png)

We ended up exploiting this as a blind boolean based SQLi, where responses with the log entry ID `15` 
(the ID of one of the PUT log entries) would be considered `true`. sqlmap did the remaining work for us:

```bash
$ podman run --rm -it paoloo/sqlmap \
    --url="http://logs_in.sharkyctf.xyz/e48e13207341b6bffb7fb1622282247b" \
    --method="POST" --headers="X-HTTP-Method-Override:PUT*" --dbms=mysql \
    --technique=B --string="<td>15</td>" --suffix="; -- " --tamper="charencode" \
    --dump --flush-session
```

It took us a while to notice that sqlmap wouldn't URL encode the payloads, which is why adding 
`--tamper="charencode"` was necessary. The flag was located in the `db.it_seems_secret` table:

```
Database: db
Table: it_seems_secret
[1 entry]
+----+--------------------------------------------------------------------------------------------+
| id | flag                                                                                       |
+----+--------------------------------------------------------------------------------------------+
| 1  | shkCTF{CVE-2019-10913_S33m3D_Bulls5H1T_B3F0R3_TH15_Ch4LL_69e28c7b0004fe05b05800596e64343b} |
+----+--------------------------------------------------------------------------------------------+
```
