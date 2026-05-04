# API Gateway Custom Domain + HTTPS Setup (Route 53 + ACM)

This guide documents the exact flow for setting up a clean API domain:

`https://api.yourdomain.com`

For an existing AWS API Gateway + Lambda backend.

---

## Target Architecture

```txt
Client
  → Route 53 DNS
  → API Gateway Custom Domain
  → API Gateway HTTP API
  → Lambda
```

---

## Prerequisites

Before starting, make sure you already have:

- A registered domain
- Route 53 hosted zone already connected to your domain nameservers
- Existing API Gateway
- Existing Lambda integration
- AWS region confirmed for your API Gateway

Example domain:

```txt
api.osayskitchenandcafe.com
```

---

# Step 1 — Request ACM Certificate

Go to:

```txt
AWS Certificate Manager (ACM)
```

Important:

```txt
Use the SAME region as your API Gateway.
```

Example:

```txt
ap-southeast-1
```

Request a public certificate for:

```txt
api.yourdomain.com
```

Choose:

```txt
DNS validation
```

Keep:

```txt
Allow export: Disable export
Key algorithm: RSA 2048
```

Click:

```txt
Request
```

---

# Step 2 — Add ACM DNS Validation Record

Open the new ACM certificate.

Copy the DNS validation values:

```txt
CNAME Name
CNAME Value
```

Example from ACM:

```txt
Name:
_abc123.api.yourdomain.com

Value:
_xyz456.acm-validations.aws
```

Go to:

```txt
Route 53 → Hosted zones → yourdomain.com → Create record
```

Create:

```txt
Record type: CNAME
Record name: _abc123.api
Value: _xyz456.acm-validations.aws
TTL: 300
```

Important:

```txt
Do not include the root domain in the Record name field.
Route 53 automatically appends it.
```

Correct:

```txt
_abc123.api
```

Wrong:

```txt
_abc123.api.yourdomain.com
```

Wait until ACM status becomes:

```txt
ISSUED
```

---

# Step 3 — Create API Gateway Custom Domain

Go to:

```txt
API Gateway → Custom domain names → Add domain name
```

Set:

```txt
Domain name: api.yourdomain.com
Domain type: Public
Routing mode: API mappings only
Endpoint type: Regional
IP address type: IPv4
Mutual TLS: Off
```

Security policy:

```txt
TLS 1.2
```

Use this for HTTP APIs because API mapping may fail with TLS 1.3.

Select the ACM certificate you created.

Click:

```txt
Create domain
```

Wait until status becomes:

```txt
Available
```

---

# Step 4 — Add API Mapping

Open:

```txt
API Gateway → Custom domain names → api.yourdomain.com
```

Go to:

```txt
API mappings
```

Click:

```txt
Configure API mappings
```

Add mapping:

```txt
API: your HTTP API
Stage: $default
Path: blank
```

Save.

Expected mapping:

```txt
api.yourdomain.com
  → your-api
  → $default stage
```

Important:

```txt
Leave Path blank if you want the custom domain root to map directly to your API.
```

---

# Step 5 — Get API Gateway Domain Name

Inside the custom domain page, find:

```txt
API Gateway domain name
```

Example:

```txt
d-xxxxxxxxxx.execute-api.ap-southeast-1.amazonaws.com
```

Copy this exact value.

Important:

```txt
This is NOT the same as your default API invoke URL.
```

Default API invoke URL looks like:

```txt
https://abc123.execute-api.ap-southeast-1.amazonaws.com
```

Custom domain target looks like:

```txt
d-xxxxxxxxxx.execute-api.ap-southeast-1.amazonaws.com
```

Use the value from:

```txt
API Gateway → Custom domain names → your custom domain → API Gateway domain name
```

---

# Step 6 — Create Route 53 DNS Record

Go to:

```txt
Route 53 → Hosted zones → yourdomain.com → Create record
```

Create:

```txt
Record name: api
Record type: CNAME
Value: d-xxxxxxxxxx.execute-api.ap-southeast-1.amazonaws.com
Routing policy: Simple
TTL: 300
```

This creates:

```txt
api.yourdomain.com
```

Wait a few minutes for DNS/API Gateway propagation.

---

# Step 7 — Test the Custom Domain

Test the root URL:

```txt
https://api.yourdomain.com/
```

Possible result:

```json
{
  "statusCode": 404,
  "message": "Cannot GET /",
  "error": "Not Found"
}
```

That is okay.

It means:

```txt
DNS works
Custom domain works
API mapping works
API Gateway reaches your backend
```

Then test a real API route:

```txt
https://api.yourdomain.com/api/inventory
```

If the endpoint requires auth, expected response may be:

```json
{
  "statusCode": 401,
  "message": "Unauthorized",
  "error": "Unauthorized"
}
```

That is also okay.

It means the request reached your application correctly.

---

# Common Issues

## 403 Forbidden

If you see:

```json
{
  "message": "Forbidden"
}
```

Check these:

1. API mapping exists
2. Stage is correct
3. Path is blank
4. Route 53 CNAME points to the API Gateway domain name
5. Custom domain status is Available
6. Wait a few minutes for propagation

---

## Cannot map HTTP API

If you see:

```txt
You can only map REST APIs to this custom domain name.
To map an HTTP API, change the security policy to TLS 1.0 or TLS 1.2.
```

Fix:

```txt
Edit custom domain → Security policy → TLS 1.2
```

Then retry API mapping.

---

## Lambda ImportModuleError

If Lambda returns:

```txt
Runtime.ImportModuleError: Cannot find module 'index'
```

It means the Lambda handler is wrong.

Handler format:

```txt
<filename>.<exported function>
```

Example:

If your deployed file is:

```txt
lambda.js
```

And your code exports:

```ts
export const handler = ...
```

Then Lambda handler should be:

```txt
lambda.handler
```

Not:

```txt
index.handler
```

For this setup, the correct handler was:

```txt
lambda.handler
```

---

# Final Result

```txt
https://api.yourdomain.com
```

Now your backend API is accessible through a clean, secure, production-ready domain.

---

## Summary

What was configured:

```txt
ACM public certificate
DNS validation in Route 53
API Gateway custom domain
TLS 1.2 security policy
API mapping to $default stage
Route 53 CNAME to API Gateway domain
Lambda handler fix: lambda.handler
```
