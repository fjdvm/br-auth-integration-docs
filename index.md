---
title: Centralized Auth Integration Guide
layout: default
permalink: /application-teams/
audience: application-team
---

# Centralized Auth Integration Guide

This is the copy-and-paste guide for connecting **Portal, HRMS, POS, SCMS, and OOS** to the Centralized Auth Service. The Auth Service is the one place that signs users in. Each application is an OIDC client: it sends a user to the Auth Service to sign in, receives a safe short-lived token, and checks whether that user may open that application.

The guide uses the working implementations in `internal-auth-service/apps/web/portal` and `trellis`. Follow the steps in order. Do not put passwords, client secrets, or `.env.local` files in Git.

Use the fixed application map below to select the correct client ID, system code, port, and directory for your app. Do not copy another application's values.

## For application integration teams

This section is for the team building or deploying an application. It tells you what to add to **your own Next.js app** and what URLs to submit. It does not ask you to edit the Auth Service.

## What you will build

<div class="mermaid">
flowchart TD
  User[Employee in a browser] --> App[Portal, HRMS, POS, SCMS, or OOS]
  App -->|OIDC authorization code + PKCE| Auth[Centralized Auth Service]
  Auth -->|Checks identity and permissions| Database[(PostgreSQL)]
  Auth -->|ID token, access token, refresh token| App
  App -->|Allows only its system code| User
  Auth -. logout event .-> App
</div>

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

## Application implementation steps

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

Submit these three URLs through the [integration request form](https://forms.gle/Z7VwH5k4ZPhTFQoC9). The Auth Service owner must register them before production login will work. Do not try to change the Auth Service configuration yourself.

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

```

Do **not** add `NODE_TLS_REJECT_UNAUTHORIZED=0`. Step 11 configures Node.js to trust only the local mkcert certificate authority during development. That is safer and prevents an insecure TLS setting from reaching production.

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

Copy this complete file. The file itself is shared by all client apps; their `.env.local` values supply the correct client ID, secret, issuer, and app URL. In Step 8, set the matching `THIS_SYSTEM_CODE` (`HRMS`, `POS`, `SCMS`, or `OOS`).

```ts
import NextAuth from "next-auth";
import { NextResponse } from "next/server";
import { refreshAccessToken } from "@/lib/auth/token-refresh";

const issuer = requiredEnvironment("AUTH_ISSUER");
const clientId = requiredEnvironment("AUTH_CLIENT_ID");
const clientSecret = requiredEnvironment("AUTH_CLIENT_SECRET");

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    {
      id: "authservice",
      name: "Auth Service",
      type: "oidc",
      issuer,
      clientId,
      clientSecret,
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
        return Response.redirect(new URL(requiredEnvironment("AUTH_URL")));
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

function requiredEnvironment(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} must be configured.`);
  return value;
}
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

type RefreshCacheEntry = { promise: Promise<JWT>; expiresAt: number };

const refreshCache = new Map<string, RefreshCacheEntry>();
const REFRESH_CACHE_TTL_MS = 30_000;

export async function refreshAccessToken(token: JWT): Promise<JWT> {
  const refreshToken = token.refreshToken;
  if (!refreshToken) return { ...token, error: "RefreshAccessTokenError" };

  const cached = refreshCache.get(refreshToken);
  if (cached && Date.now() < cached.expiresAt) return cached.promise;

  const promise = requestTokenRefresh(token, refreshToken);
  refreshCache.set(refreshToken, {
    promise,
    expiresAt: Date.now() + REFRESH_CACHE_TTL_MS,
  });

  promise.finally(() => {
    setTimeout(() => {
      const entry = refreshCache.get(refreshToken);
      if (entry?.promise === promise) refreshCache.delete(refreshToken);
    }, REFRESH_CACHE_TTL_MS);
  });

  return promise;
}

async function requestTokenRefresh(
  token: JWT,
  refreshToken: string,
): Promise<JWT> {
  try {
    const issuer = requiredEnvironment("AUTH_ISSUER").replace(/\/?$/, "/");
    const response = await fetch(`${issuer}connect/token`, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        client_id: requiredEnvironment("AUTH_CLIENT_ID"),
        client_secret: requiredEnvironment("AUTH_CLIENT_SECRET"),
        grant_type: "refresh_token",
        refresh_token: refreshToken,
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
      refreshToken: payload.refresh_token ?? refreshToken,
      error: undefined,
    };
  } catch {
    return { ...token, error: "RefreshAccessTokenError" };
  }
}

function requiredEnvironment(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} must be configured.`);
  return value;
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

Create the sign-in page as a **Server Component** that redirects immediately, before any HTML is sent to the browser:

```text
apps/web/hrms/app/signin/page.tsx
```

```tsx
import { signIn } from "@/auth";

export const dynamic = "force-dynamic";

export default async function SignInPage({
  searchParams,
}: {
  searchParams: Promise<{ callbackUrl?: string }>;
}) {
  const { callbackUrl } = await searchParams;
  await signIn("authservice", { redirectTo: callbackUrl ?? "/" });
}
```

`middleware.ts` (Step 6) already sends unauthenticated visitors here with `?callbackUrl=<original path>` before this page renders. `signIn()` runs on the server during the page's own render and throws a redirect straight to the Auth Service.

Do **not** implement this with a `"use client"` component that calls `signIn()` inside a `useEffect`. That pattern loads a blank page, hydrates it, runs the effect, and only then redirects — a full extra render+hydrate cycle in front of the OIDC redirect. It is the single biggest source of visible delay when switching between apps, and it is unnecessary: the server-side redirect above skips straight to the Auth Service with no client JavaScript involved.

## 8. Block users who do not have this application's permission

Edit the root layout at:

```text
apps/web/hrms/app/layout.tsx
```

Keep your existing fonts, providers, and visual shell. Add the `auth` import, then use this authorization decision around your current application UI:

```tsx
import { auth } from "@/auth";

const THIS_SYSTEM_CODE = "HRMS"; // POS, SCMS, or OOS in the matching app

export default async function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  const session = await auth();

  if (!session) {
    return (
      <html lang="en">
        <body>{children}</body>
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

  const appUrl = process.env.AUTH_URL;
  if (!appUrl) throw new Error("AUTH_URL is required");
  const postLogoutUrl = appUrl.replace(/\/?$/, "/");
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
  issuer={process.env.AUTH_ISSUER!}
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

## 13. If your application has a .NET backend API

If the application is only a Next.js web app, skip this section. If HRMS, POS, SCMS, OOS, or Portal has its own ASP.NET Core API, **yes, the API must validate the access token and enforce access rules too**. A frontend check alone does not protect API endpoints.

The .NET API does not need `AUTH_SECRET` or `AUTH_CLIENT_SECRET`. It receives the user's bearer access token from the web app and validates it against the Auth Service.

### A. Add the required package

From the application's .NET API directory, add this only if it is not already installed and after following your team's dependency approval process:

```zsh
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 10.0.10
```

### B. Add backend environment variables

Follow the existing `trellis/services/api-crms/.env.example` naming pattern. In the application's API `.env` file locally—or in the API hosting provider's environment-variable screen in production—set:

```dotenv
# Local: https://localhost:5001
# Production: the public HTTPS URL of the Auth Service.
JWT_AUTHORITY=https://auth.example.com/

# Kept for future audience-scoped tokens. The current Auth Service does not
# issue resource audiences, so validation remains disabled in Program.cs.
JWT_AUDIENCE=hrms-client

# The allowed browser origin for this application's API CORS policy.
WEB_HRMS_URL=https://hrms.example.com
```

For POS use `JWT_AUDIENCE=pos-client` and `WEB_POS_URL`; for SCMS use `scms-client` and `WEB_SCMS_URL`; for OOS use `oos-client` and `WEB_OOS_URL`. The API does **not** need `AUTH_SECRET` or `AUTH_CLIENT_SECRET`.

### C. Create a module-permission authorization handler

Create this file in the application's API project:

```text
Authorization/AppPermissionAuthorizationHandler.cs
```

```csharp
using System.Security.Claims;
using System.Text.Json;
using Microsoft.AspNetCore.Authorization;

namespace YourApp.Authorization;

public sealed class AppPermissionRequirement(
    string systemCode,
    string moduleName,
    string action) : IAuthorizationRequirement
{
    public string SystemCode { get; } = systemCode;
    public string ModuleName { get; } = moduleName;
    public string Action { get; } = action;
}

public sealed class AppPermissionAuthorizationHandler
    : AuthorizationHandler<AppPermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        AppPermissionRequirement requirement)
    {
        if (IsSuperUser(context.User) || HasPermission(context.User, requirement))
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }

    private static bool IsSuperUser(ClaimsPrincipal user) =>
        bool.TryParse(user.FindFirst("isSuperUser")?.Value, out var value) && value;

    private static bool HasPermission(ClaimsPrincipal user, AppPermissionRequirement requirement)
    {
        var permissions = user.FindFirst("permissions")?.Value;
        if (string.IsNullOrWhiteSpace(permissions)) return false;

        try
        {
            using var document = JsonDocument.Parse(permissions);
            return document.RootElement.TryGetProperty(requirement.SystemCode, out var app)
                && app.TryGetProperty(requirement.ModuleName, out var module)
                && module.TryGetProperty(requirement.Action, out var allowed)
                && allowed.ValueKind == JsonValueKind.True;
        }
        catch (JsonException)
        {
            return false;
        }
    }
}
```

This matches the working `api-crms` pattern. The module name must match the Auth Service module name exactly—for example, `Employee Information` in HRMS—and an action is one of `canRead`, `canWrite`, `canUpdate`, `canDelete`, `canApprove`, or `canExport`.

### D. Configure token validation in `Program.cs`

In the application's API `Program.cs`, add these imports at the top:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using YourApp.Authorization;
```

Before `var app = builder.Build();`, add this configuration. It follows `api-crms`: replace the CORS variable and policy module names with the modules used by your own application.

```csharp
var webHrmsUrl = Environment.GetEnvironmentVariable("WEB_HRMS_URL")
    ?? "https://localhost:3001";
var jwtAuthority = Environment.GetEnvironmentVariable("JWT_AUTHORITY")
    ?? "https://localhost:5001";
var jwtAudience = Environment.GetEnvironmentVariable("JWT_AUDIENCE")
    ?? "hrms-client";

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(webHrmsUrl)
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = jwtAuthority;
        options.Audience = jwtAudience;
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
        // The current Auth Service does not issue resource audiences.
        // Retain JWT_AUDIENCE for the future, but do not enable this yet.
        options.TokenValidationParameters.ValidateAudience = false;
    });

builder.Services.AddSingleton<IAuthorizationHandler, AppPermissionAuthorizationHandler>();
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("EmployeeInformationCanRead", policy =>
    {
        policy.AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme);
        policy.RequireAuthenticatedUser();
        policy.AddRequirements(new AppPermissionRequirement(
            "HRMS", "Employee Information", "canRead"));
    });
});
```

After `var app = builder.Build();`, ensure middleware is in this order:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

If the API receives browser requests directly, also put CORS before authentication and authorization:

```csharp
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
```

### E. Protect every API controller or endpoint

Add the policy to each controller that belongs to the application:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
[Authorize(Policy = "EmployeeInformationCanRead")]
public sealed class EmployeesController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok();
    }
}
```

The API should return `401` when a request has no valid token and `403` when the token is valid but lacks the required permission. Create separate policies for write, approval, export, update, and delete actions; do not rely on the frontend to protect them.

## Google Form: integration request template

<p class="form-callout"><strong>Ready to submit?</strong> <a class="form-button" href="https://forms.gle/Z7VwH5k4ZPhTFQoC9" target="_blank" rel="noopener noreferrer">Submit an integration request in Google Forms ↗</a></p>

Use the form to provide the deployed URLs and application details that the Auth Service owner needs. Never submit client secrets, `AUTH_SECRET`, passwords, tokens, or database credentials.

The Auth Service owner reviews the submitted URLs, registers the exact callback and post-logout addresses, and then confirms the client configuration with the requester.

## Production variables at a glance

Use this table as the final reminder before deploying the web application. These values belong to your application, not to the Auth Service.

### Copy this into the web application's production environment

For a hosted application, enter these values in the hosting provider's **environment variables / secrets** screen. Do not create or upload a real `.env` file to the repository. If the hosting provider requires a file, use this exact content as `apps/web/hrms/.env.local` locally only, and keep it gitignored.

```dotenv
# HRMS production example — change every HRMS value for POS, SCMS, OOS, or Portal.
# Generate a different value for every application:
# openssl rand -base64 32
AUTH_SECRET=put-a-new-random-secret-here

# The public root URL of THIS application. No trailing slash is required here.
AUTH_URL=https://hrms.example.com

# The OIDC client assigned by the Auth Service owner.
AUTH_CLIENT_ID=hrms-client
AUTH_CLIENT_SECRET=put-the-secret-received-through-the-approved-secure-channel-here

# The public HTTPS root URL of the Auth Service. Keep the final slash.
AUTH_ISSUER=https://auth.example.com/
```

Do **not** put any of these in the client/browser-visible environment:

```dotenv
# Never create these values in production:
NEXT_PUBLIC_AUTH_CLIENT_SECRET=
NEXT_PUBLIC_AUTH_SECRET=
NODE_TLS_REJECT_UNAUTHORIZED=0
```

### Replace only these values for each application

| Integrating | `AUTH_URL`                   | `AUTH_CLIENT_ID` | System code in `app/layout.tsx` |
| ----------- | ---------------------------- | ---------------- | ------------------------------- |
| HRMS        | `https://hrms.example.com`   | `hrms-client`    | `HRMS`                          |
| POS         | `https://pos.example.com`    | `pos-client`     | `POS`                           |
| SCMS        | `https://scms.example.com`   | `scms-client`    | `SCMS`                          |
| OOS         | `https://oos.example.com`    | `oos-client`     | `OOS`                           |
| Portal      | `https://portal.example.com` | `portal-client`  | No system-code gate             |

Each application needs its **own** `AUTH_SECRET`. Do not copy the HRMS secret to POS, SCMS, OOS, or Portal. `AUTH_CLIENT_SECRET` is also different per client and must be received from the Auth Service owner through a secure channel.

| Where to configure it | Variable                       | Required production value / reminder                                                                                             |
| --------------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Web application       | `AUTH_SECRET`                  | A newly generated high-entropy secret for that application. Generate with `openssl rand -base64 32`; never reuse or commit it.   |
| Web application       | `AUTH_URL`                     | The application's exact public root, such as `https://hrms.example.com`. It must agree with the registered post-logout root URL. |
| Web application       | `AUTH_CLIENT_ID`               | The assigned client ID, for example `hrms-client`.                                                                               |
| Web application       | `AUTH_CLIENT_SECRET`           | The secret supplied through an approved secure channel. Add it only to deployment secrets.                                       |
| Web application       | `AUTH_ISSUER`                  | The public HTTPS URL of the Auth Service, ending in `/`, for example `https://auth.example.com/`.                                |
| Web application       | `NODE_TLS_REJECT_UNAUTHORIZED` | **Do not set this in production.** It is not needed with a valid public certificate.                                             |

## Production handoff checklist

Before asking for deployment approval, submit all of these to the Auth Service owner:

- [ ] Application name, client ID, and system code.
- [ ] Final public root URL for the application you are deploying.
- [ ] Exact callback URL ending in `/api/auth/callback/authservice`.
- [ ] Exact post-logout root URL ending in `/`.
- [ ] Confirmation that the app's `AUTH_URL` matches the public root.
- [ ] Confirmation that the app's secret values are in deployment secrets, not Git.
- [ ] Confirmation that `NODE_TLS_REJECT_UNAUTHORIZED=0` is absent from production.
- [ ] A successful login, permission-denied, refresh, and logout test.

## Common problems

| Symptom                                    | Usually means                                                   | Fix                                                                                                        |
| ------------------------------------------ | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `invalid_redirect_uri`                     | Callback is not registered exactly                              | Submit the complete deployed callback URL and have the Auth Service owner add it to the matching client.   |
| Browser says certificate is unsafe locally | mkcert is not installed/trusted                                 | Run `mkcert -install`, regenerate `localhost` certificates, restart the apps.                              |
| Login works but Access denied appears      | The user lacks the app system code                              | In Portal administration, grant the user access to `HRMS`, `POS`, `SCMS`, or `OOS`; sign out and in again. |
| Login repeats forever                      | Wrong `AUTH_URL`, missing auth route, or cookies cannot persist | Check `.env.local`, `app/api/auth/[...nextauth]/route.ts`, and HTTPS.                                      |
| User remains signed in after logout        | The button used `signOut()` only                                | Navigate to `/api/logout` so central logout and explicit cookie clearing both happen.                      |

## Security rules worth remembering

- Treat `AUTH_CLIENT_SECRET` and `AUTH_SECRET` like passwords.
- Keep client secrets server-only; never prefix them with `NEXT_PUBLIC_`.
- Use HTTPS in every environment that performs OIDC login.
- Check system access on the server in `app/layout.tsx`; hiding buttons is not authorization.
- Use exact registered callback addresses, never broad wildcards.
- Rotate a secret immediately if it is committed or exposed.

{% unless page.audience == "application-team" %}

## For Auth Service owners

This section is only for the team operating `internal-auth-service`. Application teams submit their URLs through the Google Form; they do not edit the Auth Service. Your job is to validate the request, register exact OIDC URLs, configure production variables, deploy, and confirm the result.

### 1. Review the integration request

Before changing anything, confirm all of these details from the request:

- The application, client ID, and system code agree with the fixed application map.
- The root URL uses public `https://` and ends in `/`.
- The callback URL is exactly `https://your-app.example.com/api/auth/callback/authservice`.
- The post-logout URL is the exact root URL, including the final `/`.
- The requested client has been approved and its secret will be shared only through an approved secure channel.

Reject or return the request for correction if any URL is incomplete, uses a wildcard, uses `http://` in production, or belongs to an unapproved domain.

### 2. Configure the Auth Service production environment

Set these values in `apps/api/internal-auth-service/.env` on a self-hosted server, or in the Auth Service hosting provider's environment-variable screen. Do not place these values in HRMS, POS, SCMS, OOS, or Portal.

```dotenv
# Secret infrastructure values: obtain from the approved database and email providers.
DATABASE_URL=postgresql-connection-string-from-the-approved-secret-store
SMTP_HOST=your-smtp-host
SMTP_PORT=587
SMTP_USERNAME=your-smtp-username
SMTP_PASSWORD=your-smtp-password
SMTP_FROM=noreply@yourdomain.com

# Public web application roots. Every value must be the deployed HTTPS URL and end in /.
PORTAL_URL=https://portal.example.com/
HRMS_URL=https://hrms.example.com/
POS_URL=https://pos.example.com/
SCMS_URL=https://scms.example.com/
OOS_URL=https://oos.example.com/
CRMS_URL=https://crms.example.com/

# Public login page served by the Auth Service itself.
LOGIN_URL=https://auth.example.com/Account/Login
```

`PORTAL_URL`, `HRMS_URL`, `POS_URL`, `SCMS_URL`, `OOS_URL`, and `CRMS_URL` update the Portal system links and the CORS allowlist. They are required, but they do **not** register OIDC callbacks by themselves.

### 3. Register callback and logout URLs

In this file:

```text
apps/api/internal-auth-service/Seeders/DbSeeder.cs
```

add the approved public callback and post-logout URL to the correct client. Keep its local URL too. Example for HRMS:

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

Use the matching client and port for the other applications:

| Application | Client ID       | Local port | System code |
| ----------- | --------------- | ---------- | ----------- |
| Portal      | `portal-client` | `3000`     | None        |
| HRMS        | `hrms-client`   | `3001`     | `HRMS`      |
| POS         | `pos-client`    | `3002`     | `POS`       |
| SCMS        | `scms-client`   | `3003`     | `SCMS`      |
| OOS         | `oos-client`    | `3004`     | `OOS`       |
| CRMS        | `crms-client`   | `3005`     | `CRMS`      |

The existing seeder adds missing callback and post-logout URLs on startup. It does not remove prior URLs. Never use wildcard redirect URLs.

### 4. Deploy and confirm

After deployment, check the following before replying to the requester:

- The Auth Service starts successfully with its PostgreSQL and SMTP configuration.
- The deployed root URL appears in the Portal system catalog.
- The application can complete login using the approved callback URL.
- The user returns to the approved root after `/connect/logout`.
- The correct system code is present in the user's access claim.
- The client secret was supplied through a secure channel, never in the Google Form or Git.

Reply with the registered callback URL, registered post-logout URL, client ID, and confirmation that the deployment is ready for the application team's final test.

{% endunless %}
