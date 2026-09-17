---
title: Centralized Auth Integration Guide
---

# Centralized Auth Integration Guide

This is the copy-and-paste guide for connecting **Portal, HRMS, POS, SCMS, and OOS** to the Centralized Auth Service. The Auth Service is the one place that signs users in. Each application is an OIDC client: it sends a user to the Auth Service to sign in, receives a safe short-lived token, and checks whether that user may open that application.

The guide uses the working implementations in `internal-auth-service/apps/web/portal` and `trellis`. Follow the steps in order. Do not put passwords, client secrets, or `.env.local` files in Git.

## What you will build

```text
Browser → HRMS / POS / SCMS / OOS / Portal
              ↓
       Centralized Auth Service
              ↓
      PostgreSQL users and permissions
```

After this setup:

1. A visitor who is not signed in is sent to the Auth Service login page.
2. The Auth Service returns the visitor to the same application.
3. The application allows entry only when its system code is in the user's `systems` claim.
4. Logging out ends the central Auth Service session and clears the local application session.
5. A logout in one connected application also signs the user out of other open connected applications.

## Before you start

You need:

- Node.js 22 or later, npm, and a Next.js App Router application.
- The Auth Service running at `https://localhost:5001/` for local development.
- `mkcert` for local HTTPS. OIDC redirect addresses must use HTTPS.
- The final public URL of your application before production deployment.
- A client ID and secret listed below. These are development values; production secrets must be handled as deployment secrets.

Install and trust a local certificate once:

```zsh
mkcert -install
cd apps/web/<your-app>
mkcert localhost
```

This produces `localhost.pem` and `localhost-key.pem`. Keep both files out of Git.

## The fixed application map

Use this table exactly. `System code` is case-sensitive.

| Application | Local address            | Client ID       | Client secret                               | System code                             |
| ----------- | ------------------------ | --------------- | ------------------------------------------- | --------------------------------------- |
| Portal      | `https://localhost:3000` | `portal-client` | Obtain securely from the Auth Service owner | No system-code gate; it is the launcher |
| HRMS        | `https://localhost:3001` | `hrms-client`   | Obtain securely from the Auth Service owner | `HRMS`                                  |
| POS         | `https://localhost:3002` | `pos-client`    | Obtain securely from the Auth Service owner | `POS`                                   |
| SCMS        | `https://localhost:3003` | `scms-client`   | Obtain securely from the Auth Service owner | `SCMS`                                  |
| OOS         | `https://localhost:3004` | `oos-client`    | Obtain securely from the Auth Service owner | `OOS`                                   |

The Auth Service has a separate `crms-client` for CRMS. Its setup follows the same pattern with system code `CRMS`.

## 1. Give the Auth Service owner the deployed URL

**Do this before deploying your web application.** Send this completed information to the Auth Service owner. A home-page URL alone is not enough: OIDC accepts only callback and logout addresses registered by the server.

```text
Application: HRMS                 # change for POS, SCMS, or OOS
Client ID: hrms-client
System code: HRMS
Public application URL: https://hrms.example.com/
OIDC callback URL: https://hrms.example.com/api/auth/callback/authservice
Post-logout return URL: https://hrms.example.com/
```

Also provide the root URL for the environment variable that matches your application:

```dotenv
HRMS_URL=https://hrms.example.com/
POS_URL=https://pos.example.com/
SCMS_URL=https://scms.example.com/
OOS_URL=https://oos.example.com/
PORTAL_URL=https://portal.example.com/
```

The Auth Service owner must set these values in the **Auth Service deployment environment** (not in a web app), at:

```text
internal-auth-service/apps/api/internal-auth-service/.env
```

For hosted deployments, add the same variables in the hosting provider's environment-variable screen. Never submit an `.env` file to Git.

### What the Auth Service owner must register

In `internal-auth-service/apps/api/internal-auth-service/Seeders/DbSeeder.cs`, the matching client must contain both the local and public URLs. Example for HRMS:

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

Repeat this exact change for `pos-client`, `scms-client`, or `oos-client` using its own public URL. On startup the seeder adds missing URLs to an already registered client. It does not remove old URLs. The public root values (`HRMS_URL`, `POS_URL`, and so on) also allow CORS requests and make the Portal system cards point to the deployed app.

> Do not use a wildcard such as `https://*.example.com`. The callback and post-logout URLs must be exact addresses.

## 2. Add environment variables to the client app

Create this file in the root of the application you are integrating:

```text
apps/web/hrms/.env.local
```

For POS, SCMS, or OOS, replace `hrms` with that app name and replace every HRMS value with the row from the application map.

```dotenv
# apps/web/hrms/.env.local
# Generate with: openssl rand -base64 32
AUTH_SECRET=paste-a-new-random-secret-here

# This app's own URL. It must match the registered public root URL exactly.
AUTH_URL=https://localhost:3001

# The Auth Service client assigned to HRMS.
AUTH_CLIENT_ID=hrms-client
AUTH_CLIENT_SECRET=replace-with-the-secret-provided-securely-by-the-auth-service-owner

# Include the ending slash.
AUTH_ISSUER=https://localhost:5001/

# Development only. Do not set this to 0 in hosted environments.
NODE_TLS_REJECT_UNAUTHORIZED=0
```

Create a safe committed template beside it:

```text
apps/web/hrms/.env.example
```

```dotenv
AUTH_SECRET=
AUTH_URL=
AUTH_CLIENT_ID=
AUTH_CLIENT_SECRET=
AUTH_ISSUER=
```

Make sure the app's `.gitignore` contains these exact lines. The exception keeps the template visible to other developers.

```gitignore
.env
.env.*
!.env.example
```

For production, put `AUTH_SECRET`, `AUTH_URL`, `AUTH_CLIENT_ID`, `AUTH_CLIENT_SECRET`, and `AUTH_ISSUER` in the web application's deployment secrets. Set `AUTH_URL` to the public root, for example `https://hrms.example.com`. Do not add `NODE_TLS_REJECT_UNAUTHORIZED=0` in production.

## 3. Install the client dependency

Run this inside the Next.js application folder only if `next-auth` is not already in `package.json`:

```zsh
cd apps/web/hrms
npm install next-auth@beta
```

The examples below use `@/` imports. If the application does not already map `@/*` in `tsconfig.json`, change those imports to relative paths.

## 4. Add the Auth.js configuration

Create this file:

```text
apps/web/hrms/auth.ts
```

Copy this complete file. For POS, SCMS, and OOS, replace only `HRMS` in `THIS_SYSTEM_CODE` later in the layout; this authentication file is the same for all four applications.

```ts
import NextAuth from "next-auth";
import { NextResponse } from "next/server";
import { refreshAccessToken } from "@/lib/auth/token-refresh";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    {
      id: "authservice",
      name: "Auth Service",
      type: "oidc",
      issuer: process.env.AUTH_ISSUER,
      clientId: process.env.AUTH_CLIENT_ID,
      clientSecret: process.env.AUTH_CLIENT_SECRET,
      authorization: {
        params: {
          scope: "openid profile email roles systems offline_access",
        },
      },
    },
  ],
  pages: { signIn: "/signin" },
  callbacks: {
    async jwt({ token, profile, account }) {
      if (account) {
        token.accessToken = account.access_token;
        token.refreshToken = account.refresh_token;
        token.expiresAt = account.expires_at;
        token.error = undefined;
      }

      if (profile?.systems)
        token.systems = (profile.systems as string).split(",");
      if (profile) {
        const rawRoles = profile.role ?? profile.roles ?? [];
        token.roles = Array.isArray(rawRoles)
          ? rawRoles
          : typeof rawRoles === "string"
            ? [rawRoles]
            : [];
      }

      if (!token.expiresAt || Date.now() < (token.expiresAt - 60) * 1000)
        return token;
      return token.refreshToken
        ? refreshAccessToken(token)
        : { ...token, error: "RefreshAccessTokenError" };
    },
    async session({ session, token }) {
      session.systems = (token.systems as string[]) ?? [];
      session.roles = (token.roles as string[]) ?? [];
      session.accessToken = (token.accessToken as string) ?? "";
      if (token.error) session.error = token.error as string;
      return session;
    },
    async authorized({ auth, request }) {
      const { pathname } = request.nextUrl;
      const isAuthPath = pathname.startsWith("/api/auth");
      const isSignInPage = pathname === "/signin";

      if (auth?.error === "RefreshAccessTokenError") {
        return isAuthPath || isSignInPage;
      }
      if (auth?.user && (isAuthPath || isSignInPage)) {
        return Response.redirect(
          new URL(process.env.AUTH_URL || "https://localhost:3001/"),
        );
      }
      if (!auth?.user && !isAuthPath && !isSignInPage) return false;

      if (auth?.user) {
        const response = NextResponse.next();
        response.headers.set("Cache-Control", "no-store");
        return response;
      }
      return true;
    },
  },
});
```

### Add refresh-token support

Create the folder and file:

```text
apps/web/hrms/lib/auth/token-refresh.ts
```

```ts
import type { JWT } from "next-auth/jwt";

type RefreshTokenResponse = {
  access_token: string;
  expires_in: number;
  refresh_token?: string;
};

export async function refreshAccessToken(token: JWT): Promise<JWT> {
  if (!token.refreshToken)
    return { ...token, error: "RefreshAccessTokenError" };

  try {
    const issuer = (
      process.env.AUTH_ISSUER || "https://localhost:5001/"
    ).replace(/\/?$/, "/");
    const response = await fetch(`${issuer}connect/token`, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        client_id: process.env.AUTH_CLIENT_ID || "",
        client_secret: process.env.AUTH_CLIENT_SECRET || "",
        grant_type: "refresh_token",
        refresh_token: token.refreshToken,
      }),
      signal: AbortSignal.timeout(10_000),
    });
    const payload: unknown = await response.json();
    if (!response.ok || !isRefreshTokenResponse(payload))
      throw new Error("Refresh was rejected");

    return {
      ...token,
      accessToken: payload.access_token,
      expiresAt: Math.floor(Date.now() / 1000 + payload.expires_in),
      refreshToken: payload.refresh_token ?? token.refreshToken,
      error: undefined,
    };
  } catch {
    return { ...token, error: "RefreshAccessTokenError" };
  }
}

function isRefreshTokenResponse(value: unknown): value is RefreshTokenResponse {
  if (typeof value !== "object" || value === null) return false;
  const data = value as Record<string, unknown>;
  return (
    typeof data.access_token === "string" && typeof data.expires_in === "number"
  );
}
```

## 5. Add the Auth.js route and TypeScript types

Create the route file:

```text
apps/web/hrms/app/api/auth/[...nextauth]/route.ts
```

```ts
import { handlers } from "@/auth";

export const { GET, POST } = handlers;
```

Create the type extension:

```text
apps/web/hrms/types/next-auth.d.ts
```

```ts
import type { DefaultSession } from "next-auth";

declare module "next-auth" {
  interface Session {
    systems: string[];
    roles: string[];
    accessToken: string;
    error?: string;
    user: DefaultSession["user"];
  }

  interface Profile {
    systems?: string;
    role?: string | string[];
    roles?: string | string[];
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    systems?: string[];
    roles?: string[];
    accessToken?: string;
    refreshToken?: string;
    expiresAt?: number;
    error?: string;
  }
}
```

## 6. Protect every page with middleware

Create:

```text
apps/web/hrms/middleware.ts
```

```ts
export { auth as middleware } from "@/auth";

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api/logout).*)"],
};
```

This lets Auth.js handle protected requests while excluding static Next.js files and the two logout routes. Do not protect `api/logout`: it must be able to clear cookies without refreshing the session.

## 7. Send signed-out visitors to the Auth Service

Create:

```text
apps/web/hrms/components/auth/RedirectToLogin.tsx
```

```tsx
"use client";

import { useEffect } from "react";
import { signIn } from "next-auth/react";

export function RedirectToLogin() {
  useEffect(() => {
    signIn("authservice", { callbackUrl: "/" });
  }, []);

  return <p className="p-6">Taking you to the secure sign-in page…</p>;
}
```

## 8. Block users who do not have this application's permission

Edit the root layout at:

```text
apps/web/hrms/app/layout.tsx
```

Keep your existing fonts, providers, and visual shell. Add the `auth` and `RedirectToLogin` imports, then use this authorization decision around your current application UI:

```tsx
import { auth } from "@/auth";
import { RedirectToLogin } from "@/components/auth/RedirectToLogin";

const THIS_SYSTEM_CODE = "HRMS"; // POS, SCMS, or OOS in the matching app

export default async function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  const session = await auth();

  if (!session) {
    return (
      <html lang="en">
        <body>
          <RedirectToLogin />
        </body>
      </html>
    );
  }

  if (!session.systems.includes(THIS_SYSTEM_CODE)) {
    return (
      <html lang="en">
        <body>
          <main className="p-6">
            <h1 className="text-xl font-semibold">Access denied</h1>
            <p>
              You are signed in, but do not have access to {THIS_SYSTEM_CODE}.
            </p>
            <a className="underline" href="/api/logout">
              Sign out
            </a>
          </main>
        </body>
      </html>
    );
  }

  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

`middleware.ts` verifies a session exists. This layout check is still required because it verifies the logged-in person has the correct system permission. Never trust a client-only check for this.

## 9. Implement complete logout

Calling `signOut()` by itself logs out of only the current Next.js application. Users must be sent through the Auth Service logout endpoint so it can revoke central authorizations too.

Create:

```text
apps/web/hrms/app/api/logout/route.ts
```

```ts
import { NextRequest, NextResponse } from "next/server";
import { signOut } from "@/auth";

export async function GET(request: NextRequest) {
  await signOut({ redirect: false });

  const issuer = process.env.AUTH_ISSUER;
  if (!issuer) throw new Error("AUTH_ISSUER is required");

  const postLogoutUrl = (
    process.env.AUTH_URL || "https://localhost:3001/"
  ).replace(/\/?$/, "/");
  const logoutUrl = new URL("connect/logout", issuer);
  logoutUrl.searchParams.set("post_logout_redirect_uri", postLogoutUrl);

  const response = NextResponse.redirect(logoutUrl);
  clearAuthCookies(request, response);
  return response;
}

function clearAuthCookies(request: NextRequest, response: NextResponse) {
  for (const { name } of request.cookies.getAll()) {
    if (!/^(__Secure-|__Host-)?authjs\./.test(name)) continue;
    response.cookies.set(name, "", {
      path: "/",
      expires: new Date(0),
      httpOnly: true,
      sameSite: "lax",
      secure: name.startsWith("__Secure-") || name.startsWith("__Host-"),
    });
  }
}
```

Create the local-only endpoint used when another app reports a central logout:

```text
apps/web/hrms/app/api/logout/local/route.ts
```

```ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const response = NextResponse.redirect(new URL("/signin", request.url));
  for (const { name } of request.cookies.getAll()) {
    if (!/^(__Secure-|__Host-)?authjs\./.test(name)) continue;
    response.cookies.set(name, "", {
      path: "/",
      expires: new Date(0),
      httpOnly: true,
      sameSite: "lax",
      secure: name.startsWith("__Secure-") || name.startsWith("__Host-"),
    });
  }
  return response;
}
```

Point your Logout button/menu item to a normal browser navigation:

```tsx
<a href="/api/logout">Log out</a>
```

Do not use `signOut()` alone in a click handler.

## 10. Receive logout events from other applications

Add these two files so an HRMS user is also logged out if they log out in Portal, POS, SCMS, or OOS.

```text
apps/web/hrms/components/auth/logout-event-connection.ts
```

```ts
type Options = { accessToken: string; issuer: string };

export function connectLogoutEvents({ accessToken, issuer }: Options) {
  let active = true;
  let socket: WebSocket | undefined;
  let retry: ReturnType<typeof setTimeout> | undefined;
  let attempts = 0;

  const connect = () => {
    const url = new URL("connect/logout-events", issuer);
    url.protocol = url.protocol === "https:" ? "wss:" : "ws:";
    url.searchParams.set("access_token", accessToken);
    socket = new WebSocket(url);
    socket.onopen = () => {
      attempts = 0;
    };
    socket.onmessage = (event) => {
      if (event.data === '{"type":"logout"}')
        window.location.replace("/api/logout/local");
    };
    socket.onclose = () => {
      if (!active) return;
      retry = setTimeout(connect, Math.min(1000 * 2 ** attempts++, 30_000));
    };
  };

  connect();
  return () => {
    active = false;
    if (retry) clearTimeout(retry);
    socket?.close();
  };
}
```

```text
apps/web/hrms/components/auth/LogoutEventListener.tsx
```

```tsx
"use client";

import { useEffect } from "react";
import { connectLogoutEvents } from "./logout-event-connection";

export function LogoutEventListener({
  accessToken,
  issuer,
}: {
  accessToken: string;
  issuer: string;
}) {
  useEffect(
    () => connectLogoutEvents({ accessToken, issuer }),
    [accessToken, issuer],
  );
  return null;
}
```

Place the listener inside the signed-in branch of `app/layout.tsx`, before your application shell:

```tsx
<LogoutEventListener
  accessToken={session.accessToken}
  issuer={process.env.AUTH_ISSUER || "https://localhost:5001/"}
/>
```

The local route clears browser cookies only. It must **not** call `/connect/logout` again, or every browser tab can trigger a logout loop.

## 11. Trust the local mkcert certificate

When Next.js fetches the Auth Service during local development, Node.js needs to trust mkcert's local certificate authority. Create:

```text
apps/web/hrms/instrumentation.ts
```

```ts
export async function register() {
  if (
    process.env.NEXT_RUNTIME !== "nodejs" ||
    process.env.NODE_ENV !== "development"
  )
    return;

  const fs = await import("fs");
  const { execSync } = await import("child_process");
  const { Agent, setGlobalDispatcher } = await import("undici");
  const caRoot = execSync("mkcert -CAROOT").toString().trim();
  const ca = fs.readFileSync(`${caRoot}/rootCA.pem`);
  setGlobalDispatcher(new Agent({ connect: { ca } }));
}
```

`undici` is already a Portal dependency. If it is not present in your application, add it before using this file:

```zsh
npm install undici
```

## 12. Start and verify locally

In separate terminals, start the Auth Service and the integrated application:

```zsh
cd internal-auth-service/apps/api/internal-auth-service
dotnet run
```

```zsh
cd internal-auth-service/apps/web/hrms
npm run dev
```

Open `https://localhost:3001` and check each result:

- You are redirected to `https://localhost:5001/Account/Login`.
- After login, you return to `https://localhost:3001/`.
- A user with the `HRMS` system code sees the app.
- A user without `HRMS` sees Access denied.
- `Log out` returns to the registered HRMS root URL and a second visit asks for login.
- With Portal and HRMS open in two tabs, log out in one; the other should return to its sign-in page shortly afterward.

If the callback fails with an `invalid_redirect_uri` error, compare the entire callback address with the Auth Service registration. Check scheme (`https`), domain, port, path, and trailing slash on the post-logout root.

## Production handoff checklist

Before asking for deployment approval, submit all of these to the Auth Service owner:

- [ ] Application name, client ID, and system code.
- [ ] Final public root URL for `HRMS_URL`, `POS_URL`, `SCMS_URL`, `OOS_URL`, or `PORTAL_URL`.
- [ ] Exact callback URL ending in `/api/auth/callback/authservice`.
- [ ] Exact post-logout root URL ending in `/`.
- [ ] Confirmation that the app's `AUTH_URL` matches the public root.
- [ ] Confirmation that the app's secret values are in deployment secrets, not Git.
- [ ] Confirmation that `NODE_TLS_REJECT_UNAUTHORIZED=0` is absent from production.
- [ ] A successful login, permission-denied, refresh, and logout test.

## Common problems

| Symptom                                            | Usually means                                                   | Fix                                                                                                        |
| -------------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `invalid_redirect_uri`                             | Callback is not registered exactly                              | Submit the complete deployed callback URL and have the Auth Service owner add it to the matching client.   |
| Browser says certificate is unsafe locally         | mkcert is not installed/trusted                                 | Run `mkcert -install`, regenerate `localhost` certificates, restart the apps.                              |
| Login works but Access denied appears              | The user lacks the app system code                              | In Portal administration, grant the user access to `HRMS`, `POS`, `SCMS`, or `OOS`; sign out and in again. |
| Login repeats forever                              | Wrong `AUTH_URL`, missing auth route, or cookies cannot persist | Check `.env.local`, `app/api/auth/[...nextauth]/route.ts`, and HTTPS.                                      |
| User remains signed in after logout                | The button used `signOut()` only                                | Navigate to `/api/logout` so central logout and explicit cookie clearing both happen.                      |
| Production app loads but Portal links to localhost | The Auth Service deployment lacks the matching `*_URL`          | Submit the deployed root URL and set it in the Auth Service environment.                                   |

## Security rules worth remembering

- Treat `AUTH_CLIENT_SECRET` and `AUTH_SECRET` like passwords.
- Keep client secrets server-only; never prefix them with `NEXT_PUBLIC_`.
- Use HTTPS in every environment that performs OIDC login.
- Check system access on the server in `app/layout.tsx`; hiding buttons is not authorization.
- Use exact registered callback addresses, never broad wildcards.
- Rotate a secret immediately if it is committed or exposed.
