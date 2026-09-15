# Linky — Production-Grade URL Shortener

Linky is a full-stack URL shortener built for security and analytics, not just link compression.
It goes beyond basic shortening with password-protected links, one-time-use links, QR code
generation, and detailed click analytics powered by device and browser parsing.

---

## Features

- **Core Shortening** - Generate short links with random codes or custom aliases
- **Access Control** - Private links, password-protected links, and one-time-use (self-destructing) links
- **QR Codes** - Auto-generated downloadable QR codes for every short link
- **Link Organization** - Tag, favorite, archive, and duplicate links
- **Analytics Dashboard** - Track clicks with browser, OS, device, and referrer breakdowns
- **Authentication** - JWT-based auth with access/refresh tokens, plus API keys for programmatic access
- **Rate Limiting** - Redis-backed protection against abuse

## Tech Stack

**Backend** - [`linky-backend-springboot`](https://github.com/AjinkyaD3/linky-backend-springboot)
- Java 21, Spring Boot 3.5.4
- PostgreSQL - primary data store
- Redis - caching and rate-limiting
- Spring Security + JWT (jjwt)
- Resend - transactional email (password reset)
- ZXing - QR code generation
- Deployed on Render

**Frontend** - [`linky-frontend-nextjs`](https://github.com/AjinkyaD3/linky-frontend-nextjs)
- Next.js 16 (App Router), React 19, TypeScript
- Tailwind CSS v4
- TanStack Query, React Hook Form + Zod
- Radix UI primitives with a custom design system
- Deployed on Vercel

## Architecture

This repository ties the frontend and backend together as git submodules so the whole project
is browsable from one place, while each half deploys independently (frontend to Vercel, backend
to Render) without triggering a redeploy of the other:

```
linky/
  linky-backend-springboot/   Spring Boot REST API (own repo, own deploy)
  linky-frontend-nextjs/      Next.js frontend (own repo, own deploy)
  .gitmodules
```

### Clone with submodules

```bash
git clone --recurse-submodules https://github.com/AjinkyaD3/linky.git
```

If already cloned without submodules:

```bash
git submodule update --init --recursive
```

## Getting Started

### Backend

```bash
cd linky-backend-springboot
cp .env.example .env
./mvnw spring-boot:run
```

Fill in your own DB/Redis/JWT/Resend values in `.env`. Requires PostgreSQL and Redis running
locally (see the backend's own README for details).

### Frontend

```bash
cd linky-frontend-nextjs
cp .env.local.example .env.local
npm install
npm run dev
```

Point `NEXT_PUBLIC_API_URL` in `.env.local` at your running backend.

## Why Linky

Most URL shorteners just compress a link. Linky is built as a secure link-management platform.
Password protection, private links, and one-time-use links make it suitable for sharing sensitive
documents or exclusive content, while built-in analytics remove the need for third-party tracking
tools.

## License

MIT
