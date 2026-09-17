---
title: Auth Service Owner Runbook
layout: default
permalink: /auth-service-owner/
audience: auth-service-owner
---

# Auth Service Owner Runbook

This page is only for the team operating `internal-auth-service`. Application teams submit their URLs through the Google Form; they do not edit the Auth Service.

## 1. Review the integration request

Confirm these values before approving a request:

- Application, client ID, and system code match the approved application map.
- Root URL uses public `https://` and ends in `/`.
- Callback URL is exactly `https://your-app.example.com/api/auth/callback/authservice`.
- Post-logout URL is the exact root URL, including the final `/`.
- Secrets will be sent only through an approved secure channel.

Reject requests with wildcard URLs, `http://` production URLs, incomplete values, or unapproved domains.

## 2. Configure the Auth Service production environment

Set these values in `apps/api/internal-auth-service/.env` for a self-hosted server, or in the Auth Service hosting provider's environment-variable screen.

```dotenv
DATABASE_URL=postgresql-connection-string-from-the-approved-secret-store
SMTP_HOST=your-smtp-host
SMTP_PORT=587
SMTP_USERNAME=your-smtp-username
SMTP_PASSWORD=your-smtp-password
SMTP_FROM=noreply@yourdomain.com

PORTAL_URL=https://portal.example.com/
HRMS_URL=https://hrms.example.com/
POS_URL=https://pos.example.com/
SCMS_URL=https://scms.example.com/
OOS_URL=https://oos.example.com/
CRMS_URL=https://crms.example.com/
LOGIN_URL=https://auth.example.com/Account/Login
```

The `*_URL` values update Portal links and the CORS allowlist. They do not register OIDC callbacks by themselves.

## 3. Register callback and logout URLs

Update this file:

```text
apps/api/internal-auth-service/Seeders/DbSeeder.cs
```

Keep the local addresses and add the approved public URLs to the matching client. HRMS example:

```csharp
(
    "hrms-client",
    "READ_THE_SECRET_FROM_A_SECURE_DEPLOYMENT_SECRET",
    new[]
    {
        "https://localhost:3001/api/auth/callback/authservice",
        "https://hrms.example.com/api/auth/callback/authservice"
    },
    new[]
    {
        "https://localhost:3001/",
        "https://hrms.example.com/"
    }
),
```

| Application | Client ID       | Local port | System code |
| ----------- | --------------- | ---------- | ----------- |
| Portal      | `portal-client` | `3000`     | None        |
| HRMS        | `hrms-client`   | `3001`     | `HRMS`      |
| POS         | `pos-client`    | `3002`     | `POS`       |
| SCMS        | `scms-client`   | `3003`     | `SCMS`      |
| OOS         | `oos-client`    | `3004`     | `OOS`       |
| CRMS        | `crms-client`   | `3005`     | `CRMS`      |

The seeder adds missing callback and post-logout URLs on startup. It does not remove previous URLs. Never use wildcard redirect URLs.

## 4. Deploy and confirm

Before replying to the requester, verify:

- The Auth Service starts with its PostgreSQL and SMTP configuration.
- Portal shows the deployed application URL.
- Login completes through the approved callback URL.
- `/connect/logout` returns the user to the approved root URL.
- The user receives the expected system and permission claims.
- The client secret was never sent through Google Forms or committed to Git.

Reply with the registered callback URL, post-logout URL, client ID, and confirmation that the application team can perform its final test.
