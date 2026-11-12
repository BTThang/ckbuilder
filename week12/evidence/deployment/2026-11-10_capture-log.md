# Deployment Endpoint Capture Log

Captured (UTC): 2026-11-10T02:49:20Z
Captured from: local environment using Invoke-WebRequest (read-only GET)

| Endpoint | URL | HTTP status | Content-Type | Cache-Control |
|---|---|---|---|---|
| Frontend version | https://ckb-digital-credential-web.vercel.app/version.json | 200 | application/json; charset=utf-8 | no-store |
| Backend version | https://ckb-digital-credential-service.onrender.com/api/version | 200 | application/json; charset=utf-8 | no-store |
| Backend health | https://ckb-digital-credential-service.onrender.com/api/health | 200 | application/json; charset=utf-8 | no-store |
| Backend families | https://ckb-digital-credential-service.onrender.com/api/verify/families | 200 | application/json; charset=utf-8 |  |

Raw response bodies are stored beside this log as .json files. No credentials,
tokens, private keys or environment variable values are included.
