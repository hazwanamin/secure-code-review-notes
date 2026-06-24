# JavaScript — Prototype Pollution

**OWASP:** A05:2025 — Injection

## The Problem

```javascript
function merge(target, source) {
    for (let key in source) {
        if (typeof source[key] === 'object') {
            target[key] = merge(target[key] || {}, source[key]);
        } else {
            target[key] = source[key];
        }
    }
    return target;
}

const userConfig = JSON.parse(userInput);
const config = merge({}, userConfig);
```

`for...in` iterates over all keys in user input and sets them on 
the target with no validation. There's nothing blocking dangerous 
keys like `__proto__`.

Every JavaScript object inherits from `Object.prototype`. If an 
attacker sends this:

```json
{
    "__proto__": { "isAdmin": true }
}
```

The merge walks into `__proto__` and sets `isAdmin` on 
`Object.prototype` itself. Now every object in the application 
inherits it:

```javascript
const user = {};
console.log(user.isAdmin); // true
```

Any authorisation check relying on `isAdmin` being false is 
bypassed — application-wide, not just for one user. `constructor` 
and `prototype` keys carry the same risk.

## The Fix

```javascript
function merge(target, source) {
    for (let key in source) {
        if (key === '__proto__' || 
            key === 'constructor' || 
            key === 'prototype') {
            continue;
        }

        if (typeof source[key] === 'object' && source[key] !== null) {
            target[key] = merge(target[key] || {}, source[key]);
        } else {
            target[key] = source[key];
        }
    }
    return target;
}
```

Skip `__proto__`, `constructor`, and `prototype` entirely — they 
should never come from user input.

Additional options: use `Object.create(null)` as the target so 
there's no prototype chain to pollute, or validate input with 
`joi` or `ajv` before it reaches the merge function.