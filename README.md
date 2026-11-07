# hop

**A whole Next.js app that does exactly one thing: 308 your entire domain somewhere else.**
Four small files. One dynamic route. Zero database, zero components worth mentioning.

Moving a site from `old.dev` to `new.dev`? Don't keep a server alive just to say "go there".
Deploy `hop`, point both domains at it, forget it exists.

## Why?

- Domain migrations, `www` → apex canonicalization, killing a side project's old URL.
- The "classic" solution is a VPS running nginx for a single `return 301`.
- hop runs on Vercel's edge for free, deploys in 30 seconds, and the whole codebase fits in your head.

## Quick start

1. Create the files below (`package.json`, `app/layout.jsx`, `app/[[...slug]]/route.js`, `app/page.jsx`). Plain JS on purpose — no TypeScript toolchain required.
2. `npm install next react react-dom`
3. `TARGET=https://example.com npm run dev`
4. `curl -i localhost:3000/docs/getting-started?x=1` → `308` to `https://example.com/docs/getting-started?x=1`

## The whole program

`package.json`:

```json
{
  "name": "hop",
  "private": true,
  "scripts": { "dev": "next dev", "build": "next build", "start": "next start" },
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

`app/layout.jsx` — the mandatory root layout, nothing more:

```jsx
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

`app/[[...slug]]/route.js` — the entire "backend":

```js
import { NextResponse } from 'next/server';

export const dynamic = 'force-dynamic';

export function GET(req) {
  const base = process.env.TARGET;
  if (!base) {
    return new NextResponse('Set the TARGET env var', { status: 500 });
  }
  const target = new URL(req.nextUrl.pathname + req.nextUrl.search, base);
  return NextResponse.redirect(target, 308);
}
```

`app/page.jsx` — a landing page, so `/` isn't a 404:

```jsx
export default function Home() {
  return (
    <main style={{ fontFamily: 'system-ui', padding: '4rem', lineHeight: 1.6 }}>
      <h1>hop</h1>
      <p>
        Every path except this one permanently redirects to <code>TARGET</code>.
      </p>
    </main>
  );
}
```

## Deploy

Push to GitHub → "Import Project" in Vercel → add env var `TARGET=https://your-new-domain.com` → deploy.
Attach the old domain in project settings. Done — requests now bounce at the edge.

## How it works

- `app/[[...slug]]/route.js` is an optional catch-all route handler: it matches every path except `/` (which `app/page.jsx` claims).
- It reads `process.env.TARGET`, glues the incoming `pathname + search` onto it, and answers with `NextResponse.redirect(..., 308)` — a permanent redirect that preserves HTTP method semantics.
- `dynamic = 'force-dynamic'` keeps it out of any static optimization: the redirect target stays an env var, not a build-time constant you have to redeploy to change.
- The only state is one environment variable.

## Caveats

- Don't set `TARGET` to the domain hop itself serves — infinite loop, browser gives up first.
- The handler answers `GET` only; `POST` to a redirected path gets 405. For form-heavy legacy flows use 307 and add a `POST` export.
- If you need per-path redirect maps (old URL → *different* new URL), you want `redirects()` in `next.config.js`, not this.

## License

MIT. Contains less application code than most `loading.tsx` spinners.
