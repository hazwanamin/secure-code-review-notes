# Python — SQL Injection

**OWASP:** A05:2025 — Injection

## The Problem

```python
import sqlite3

def get_user(username):
    conn = sqlite3.connect("users.db")
    cursor = conn.cursor()
    
    query = "SELECT * FROM users WHERE username = '" + username + "'"
    cursor.execute(query)
    
    return cursor.fetchone()
```

The issue is on this line:

```python
query = "SELECT * FROM users WHERE username = '" + username + "'"
```

Whatever the user types gets glued directly into the SQL query. The database has no way to tell the difference between your query and the user's input — it just runs whatever ends up in that string.

So if someone types `admin' OR '1'='1` the query becomes:

```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1'
```

`'1'='1'` is always true. Every row comes back. From there an attacker can go further — dump data, modify records, or delete tables depending on database permissions.

## The Fix

```python
import sqlite3

def get_user(username):
    conn = sqlite3.connect("users.db")
    cursor = conn.cursor()
    
    query = "SELECT * FROM users WHERE username = ?"
    cursor.execute(query, (username,))
    
    return cursor.fetchone()
```

The `?` is a placeholder. The database gets the query structure first, locks it in, then receives the user's value separately via `cursor.execute(query, (username,))`.

At that point it doesn't matter what the user types — the database already knows the structure. `admin' OR '1'='1` gets treated as a literal string to search for, not part of the query logic. No match found, nothing returned. Attack neutralised.

## One-liner for interviews
*"SQL injection happens when user input gets concatenated into a query string — parameterised queries fix this by separating the query structure from the data before the database ever sees it."*