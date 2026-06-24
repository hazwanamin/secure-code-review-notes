\# Python — Command Injection



\*\*OWASP:\*\* A05:2025 — Injection



\## The Problem



```python

import os



def ping\_host(hostname):

&#x20;   output = os.system("ping -c 4 " + hostname)

&#x20;   return output

```



Same root cause as SQL injection — user input gets glued into a 

system command. If someone types `google.com; cat /etc/passwd` 

the command becomes:



```bash

ping -c 4 google.com; cat /etc/passwd

```



The `;` ends the ping and starts a new command. The OS runs both. 

From there — read files, exfiltrate data, drop malware. Whatever 

the server's user account can do, the attacker can now do too.



`os.system()` passes the string directly to the shell — which 

treats `;`, `\&\&`, `|`, and `$()` as command separators. Any of 

these in user input chains additional commands onto the original.



\## The Fix



```python

import subprocess

import re



def ping\_host(hostname):

&#x20;   # Allowlist — only letters, numbers, dots, hyphens

&#x20;   if not re.match(r'^\[a-zA-Z0-9.\\-]+$', hostname):

&#x20;       raise ValueError("Invalid hostname")



&#x20;   output = subprocess.run(

&#x20;       \["ping", "-c", "4", hostname],

&#x20;       capture\_output=True,

&#x20;       text=True,

&#x20;       timeout=10

&#x20;   )

&#x20;   return output.stdout

```



Two things fixed:



`subprocess.run()` with a list passes each item directly to the 

OS as a separate argument — never through the shell. `hostname` 

is always treated as a single argument to `ping`, not part of a 

shell string. Typing `google.com; cat /etc/passwd` just tries to 

ping a host literally named that — fails cleanly.



The regex check is a second layer — rejects anything containing 

`;`, `|`, `\&`, spaces, or `$()` before it reaches subprocess. 

Even if someone bypasses the list-based call, the input check 

stops them earlier.



`timeout=10` prevents an attacker from hanging the server with 

a slow host — basic denial of service protection.

