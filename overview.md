---
title: Centralized Auth Documentation
layout: default
permalink: /
audience: overview
---

# Centralized Auth Documentation

Choose the guide for your responsibility. The two guides are separate so application teams do not need to edit the Auth Service, and Auth Service owners do not need to follow the client-app implementation steps.

## Application integration teams

Use this when connecting Portal, HRMS, POS, SCMS, or OOS to the Centralized Auth Service. It includes the Next.js OIDC setup, refresh tokens, permission gate, complete logout, cross-app WebSocket logout, production variables, and optional .NET API validation.

<p class="form-callout"><a class="form-button" href="{{ '/application-teams/' | relative_url }}">Open the application-team guide →</a></p>

## Auth Service owners

Use this when reviewing an integration request, setting the Auth Service production environment, registering OIDC callback and post-logout URLs, and confirming a deployment.

<p class="form-callout"><a class="form-button" href="{{ '/auth-service-owner/' | relative_url }}">Open the Auth Service owner runbook →</a></p>

## Submit an integration request

Application teams must submit their deployed root URL, callback URL, and post-logout URL before production rollout. Do not submit client secrets, tokens, passwords, or database credentials.

<p class="form-callout"><a class="form-button" href="https://forms.gle/Z7VwH5k4ZPhTFQoC9" target="_blank" rel="noopener noreferrer">Submit an integration request ↗</a></p>
