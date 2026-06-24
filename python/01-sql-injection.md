\# Python 01 — SQL Injection



\## OWASP Category

A05:2025 — Injection



\## Vulnerable Code

```python

import sqlite3



def get\_user(username):

&#x20;   conn = sqlite3.connect("users.db")

&#x20;   cursor = conn.cursor()

&#x20;   

&#x20;   query = "SELECT \* FROM users WHERE username = '" + username + "'"

&#x20;   cursor.execute(query)

&#x20;   

&#x20;   return cursor.fetchone()



\# Example call

user\_input = "admin' OR '1'='1"

result = get\_user(user\_input)

print(result)

```



\## What is wrong

The `username` value from user input is concatenated directly into 

the SQL query string. The database cannot distinguish between the 

query structure and the user-supplied data.



Submitting `admin' OR '1'='1` changes the query to:



```sql

SELECT \* FROM users WHERE username = 'admin' OR '1'='1'

```



`'1'='1'` is always true — the query returns every row in the 

users table regardless of the username supplied. An attacker 

can also extend this to extract, modify, or delete data using 

UNION-based or stacked query payloads.



\## Secure Version

```python

import sqlite3



def get\_user(username):

&#x20;   conn = sqlite3.connect("users.db")

&#x20;   cursor = conn.cursor()

&#x20;   

&#x20;   query = "SELECT \* FROM users WHERE username = ?"

&#x20;   cursor.execute(query, (username,))

&#x20;   

&#x20;   return cursor.fetchone()



\# Example call — same input, safe handling

user\_input = "admin' OR '1'='1"

result = get\_user(user\_input)

print(result)

```



\## Explanation

The `?` placeholder separates the query structure from the data. 

The database receives and compiles the query structure first — 

before the value is supplied. When `username` is bound via 

`cursor.execute(query, (username,))`, it is treated as a literal 

string to match, not executable SQL syntax.



Submitting `admin' OR '1'='1` now searches for a user whose 

username is literally `admin' OR '1'='1` — which doesn't exist, 

so the query returns nothing. The attack is neutralised.

