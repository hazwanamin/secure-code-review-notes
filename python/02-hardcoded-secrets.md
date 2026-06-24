# Python — Hardcoded Secrets

**OWASP:** A04:2025 — Cryptographic Failures

## The Problem

```python
import requests

API_KEY = "sk-prod-9x8y7z6w5v4u3t2s1r0q"
DB_PASSWORD = "SuperSecret123!"

def get_payment_data(transaction_id):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    response = requests.get(
        f"https://api.payments.com/transactions/{transaction_id}",
        headers=headers
    )
    return response.json()
```

Credentials are sitting directly in the code. The moment this 
gets committed to a repo, those secrets are in git history 
permanently — deleting the file later doesn't remove them from 
the commit log.

Attackers run automated tools (Gitleaks, TruffleHog) that scan 
public GitHub repos specifically looking for this pattern. If the 
repo is public, the secret is public.

## The Fix

```python
import requests
import os

def get_payment_data(transaction_id):
    api_key = os.environ.get("PAYMENT_API_KEY")

    if not api_key:
        raise ValueError("PAYMENT_API_KEY environment variable not set")

    headers = {"Authorization": f"Bearer {api_key}"}
    response = requests.get(
        f"https://api.payments.com/transactions/{transaction_id}",
        headers=headers
    )
    return response.json()
```

Credentials come from environment variables — never from the code 
itself. The source file contains no secret, so committing it 
exposes nothing. The actual value lives in a `.env` file locally 
or a secrets manager (AWS Secrets Manager, Azure Key Vault) in 
production.

Add `.env` to `.gitignore` immediately:

# echo ".env" >> .gitignore

If a secret has already been committed — rotate it first, then 
clean the history. Removing it from the file does nothing, it's 
still in the git log. Use `git filter-repo` to rewrite history, 
then force push. Rotating takes priority — assume it's already 
been seen.
