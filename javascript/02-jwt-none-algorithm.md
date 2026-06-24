# JavaScript — JWT None Algorithm Bypass

**OWASP:** A07:2025 — Authentication Failures

## The Problem

```javascript
const jwt = require('jsonwebtoken');

function verifyToken(token) {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return decoded;
}
```

Looks fine at first glance. The issue isn't in this code 
directly — it's in how some JWT libraries handle the `alg` field 
in the token header.

A JWT has three parts: header, payload, signature. The header 
tells the server which algorithm was used to sign it:

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

Some libraries read the `alg` field from the token itself and 
use it to decide how to verify the signature. An attacker can 
exploit this by crafting a token with `alg` set to `none`:

```json
{
    "alg": "none",
    "typ": "JWT"
}
```

With a payload of:

```json
{
    "userId": 1,
    "role": "admin"
}
```

If the library accepts `alg: none` and skips signature 
verification as a result, the attacker has forged a valid admin 
token with no secret required. No brute force, no key theft — 
just changing one field.

The token looks like this when assembled (only header.payload. - [missing .signature]):

eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VySWQiOjEsInJvbGUiOiJhZG1pbiJ9.

Note the empty signature at the end. The server accepts it 
anyway if the library is vulnerable.

## The Fix

```javascript
const jwt = require('jsonwebtoken');

function verifyToken(token) {
    const decoded = jwt.verify(token, process.env.JWT_SECRET, {
        algorithms: ['HS256']
    });
    return decoded;
}
```

Explicitly specify the allowed algorithm in the verification 
options. The library now ignores whatever `alg` the token 
claims and only accepts tokens signed with HS256. A token 
with `alg: none` gets rejected immediately.

Never trust the algorithm declared in the token header — the 
server should always decide which algorithm is valid, not the 
client.

Two additional things to lock down:

Validate `exp` (expiration) and `iss` (issuer) claims so stolen 
tokens have a limited window and can't be replayed across 
services:

```javascript
const decoded = jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ['HS256'],
    issuer: 'your-app-name',
    audience: 'your-api'
});
```

If using asymmetric keys (RS256), never accept HS256 as a 
fallback — an attacker can sign a token with your public key 
as the HMAC secret and the server will verify it successfully 
if algorithm switching is allowed.
