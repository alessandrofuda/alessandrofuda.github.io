---
layout: post
title: "SEO for React SPAs Without SSR: Puppeteer Prerendering in Production"
---

*React SPAs are nearly invisible to social media crawlers and slower to index on Google. Here's how I solved SEO for [LongTermMemory](https://longtermemory.com) without migrating to Next.js , using a two-variant routing pattern and a Puppeteer script that prerenders the landing page at build time.*

---

## The Problem

A React SPA with client-side routing serves one thing to every visitor: a nearly empty `index.html` with a `<div id="root"></div>` and a JavaScript bundle. Google's crawler can execute JavaScript and will eventually index the content, but it does so in a second wave , days or weeks after the initial crawl. Social media crawlers (Facebook, Twitter/X, LinkedIn, Slack) don't execute JavaScript at all. They see the empty shell, find no `og:title` or `og:description` meta tags, and either show a blank card or scrape the minimal fallback tags from `<head>`.

For a SaaS landing page, this is a real problem. Pricing, FAQ, feature descriptions , all the content that matters for SEO and social sharing , exists only in JavaScript. It never lands in the raw HTML that crawlers read.

The standard answer is server-side rendering: Next.js, Remix, or a similar framework. But LongTermMemory's frontend is a standalone Vite + React 19 SPA that has been in production for months. Migrating to Next.js would mean rewriting routing, data fetching patterns, authentication callbacks, Stripe integration, and the Tailwind configuration , weeks of work for a feature that benefits one route.

The alternative: **prerender the landing page at build time using Puppeteer**, and serve the resulting static HTML as `dist/index.html`.

---

## The Architecture: Two Landing Page Variants

The core idea is a split: one version of the landing page for authenticated users (the full interactive app), and a separate static version for everyone else (which search engines and crawlers see).

In `App.tsx`, the root route renders a `LandingRoute` component instead of directly mounting a page:

```tsx
// src/App.tsx
function LandingRoute() {
  const isAuthenticated = !!localStorage.getItem('auth_token');
  const [searchParams] = useSearchParams();
  const unsubscribed = searchParams.get('unsubscribed') === '1';

  return (
    <>
      {unsubscribed && (
        <div className="fixed top-4 left-1/2 -translate-x-1/2 z-[60] ...">
          <p className="text-sm text-green-800">
            You have successfully disabled your notifications.
          </p>
        </div>
      )}
      <Suspense fallback={<PageLoader />}>
        {isAuthenticated ? <Landing /> : <LandingPublic />}
      </Suspense>
    </>
  );
}
```

- **`Landing`** , the full authenticated experience: upload forms, Stripe checkout, `UserContext` for user state, real-time plan limits.
- **`LandingPublic`** , a stateless version with no `UserContext`, no Stripe, no authenticated API calls. Its only job is to render all the landing page content as static HTML that Puppeteer can capture.

Both components look identical to users. The split is invisible at runtime.

---

## `LandingPublic`: Designed for Prerendering

`LandingPublic` has two responsibilities: look like the real landing page, and be fully renderable by a headless browser.

**The `data-prerender-ready` marker.** The prerender script needs to know when React has finished mounting. Rather than relying on arbitrary timeouts, `LandingPublic` puts a `data-prerender-ready` attribute on its root element:

```tsx
<div className="min-h-screen bg-slate-50" data-prerender-ready>
  {/* ...full page content... */}
</div>
```

The Puppeteer script waits for `[data-prerender-ready]` before proceeding.

**Lazy sections with `IntersectionObserver`.** Below-fold sections (Pricing, FAQ, Educational modals) use a `LazySection` wrapper that only renders children when the container scrolls into view:

{% raw %}
```tsx
function LazySection({ children, fallback, id }) {
  const [visible, setVisible] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) { setVisible(true); observer.disconnect(); } },
      { rootMargin: '200px', threshold: 0 }
    );
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref} id={id}>
      {visible ? children : (fallback || <div style={{ minHeight: '200px' }} />)}
    </div>
  );
}
```
{% endraw %}

This keeps the initial bundle fast for real users. For Puppeteer, the script needs to scroll through the entire page to fire the observers and reveal all sections before capturing the HTML.

**Pricing data from the API.** The static variant still fetches live pricing from the backend:

```tsx
useEffect(() => {
  const fetchPlans = async () => {
    const fetchedPlans = await plansApi.getCommercialPlans();
    setPlans(fetchedPlans.slice(1)); // skip Free plan
    setIsLoading(false);
  };
  fetchPlans();
}, []);
```

This means the prerendered HTML contains real pricing numbers, not hardcoded values that would go stale.

**`react-helmet-async` for full meta coverage.** All SEO meta tags , Open Graph, Twitter Card, canonical URL, keywords, JSON-LD structured data , are injected via `<Helmet>` in `LandingPublic`. Because Puppeteer captures the page after JavaScript executes, these tags end up in the final HTML:

```tsx
<Helmet>
  <title>LongTerm Memory - AI-Powered Study & Exam Preparation</title>
  <meta name="description" content="Master any subject with AI-powered question-answer generation..." />
  <meta property="og:title" content="LongTerm Memory - AI-Powered Study & Exam Preparation" />
  <meta property="og:image" content="https://longtermemory.com/og-image.jpg" />
  <meta name="twitter:card" content="summary_large_image" />
  <link rel="canonical" href="https://longtermemory.com" />
  <script type="application/ld+json">
    {JSON.stringify({ "@context": "https://schema.org", "@type": "SoftwareApplication", ... })}
  </script>
</Helmet>
```

The JSON-LD block includes the live pricing offers built from the API response, making it valid structured data for Google's Rich Results.

---

## The Prerender Script

`scripts/prerender.mjs` runs after the Vite build and overwrites `dist/index.html` with the prerendered HTML. Key parts of the script:

```js
import puppeteer from 'puppeteer';
import { promises as fs } from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
import http from 'http';
import handler from 'serve-handler';

const distPath = path.join(path.dirname(fileURLToPath(import.meta.url)), '../dist');
const indexPath = path.join(distPath, 'index.html');

function tryListenOnPort(port) {
  return new Promise((resolve, reject) => {
    const server = http.createServer((req, res) => handler(req, res, { public: distPath }));
    server.on('error', reject);
    server.listen(port, () => resolve({ server, port }));
  });
}

async function startServer(startPort = 3010) {
  const maxPort = startPort + 3;
  for (let port = startPort; port <= maxPort; port++) {
    try {
      return await tryListenOnPort(port);
    } catch (err) {
      if (err.code === 'EADDRINUSE') continue;
      throw err;
    }
  }
  throw new Error(`All ports ${startPort},${maxPort} are already in use`);
}

async function prerenderPage() {
  const { server, port } = await startServer(3010);

  try {
    const browser = await puppeteer.launch({
      headless: 'new',
      args: ['--no-sandbox', '--disable-setuid-sandbox']
    });
    const page = await browser.newPage();

    // Simulate unauthenticated user: clear localStorage before any script runs
    await page.evaluateOnNewDocument(() => { localStorage.clear(); });

    await page.goto(`http://localhost:${port}`, {
      waitUntil: 'networkidle0',
      timeout: 30000
    });

    // Wait for React to finish mounting
    await page.waitForSelector('[data-prerender-ready]', { timeout: 10000 })
      .catch(() => console.log('⚠ data-prerender-ready not found, continuing...'));

    // Scroll to trigger all IntersectionObserver-gated sections
    await page.evaluate(async () => {
      const delay = ms => new Promise(r => setTimeout(r, ms));
      for (let y = 0; y < document.body.scrollHeight; y += 400) {
        window.scrollTo(0, y);
        await delay(100);
      }
      window.scrollTo(0, 0);
    });

    // Wait for pricing plan cards to appear
    await page.waitForFunction(
      () => {
        const pricing = document.querySelector('#pricing');
        if (!pricing) return false;
        return pricing.querySelectorAll('.rounded-lg.bg-white.p-5').length >= 2;
      },
      { timeout: 15000 }
    ).catch(() => console.log('⚠ Pricing data did not load in time, continuing...'));

    // Final wait for animations
    await new Promise(resolve => setTimeout(resolve, 2000));

    const html = await page.content();
    await fs.writeFile(indexPath, html, 'utf8');

    await browser.close();
  } finally {
    server.close();
  }
}

prerenderPage();
```

A few design choices worth noting:

**`evaluateOnNewDocument` vs `evaluate`.** Using `evaluateOnNewDocument` to clear `localStorage` runs the code *before any page script executes*, including React's initial render. If you cleared `localStorage` after navigation with `evaluate`, the React component tree would already have read `auth_token`, rendered `Landing` instead of `LandingPublic`, and it would be too late.

**`waitUntil: 'networkidle0'`** waits until there are no more than 0 in-flight network requests for 500ms. This is the right choice here because `LandingPublic` fetches pricing data on mount , you need the API response to arrive before capturing the HTML.

**The scroll loop.** `IntersectionObserver` only fires when elements enter the viewport. A headless browser has a viewport but doesn't scroll automatically. The 400px step with a 100ms delay gives each observer time to fire and its component time to mount before moving to the next section.

**The pricing check** uses `waitForFunction` with a DOM selector rather than a fixed timeout. If the API is slow, a 2-second `setTimeout` would produce HTML with a loading spinner instead of pricing data. Polling for the actual DOM element is reliable regardless of API latency.

**Port cycling** tries 3010 through 3013. CI environments often have ports occupied by other services; retrying automatically avoids flaky build failures.

---

## Build Commands

```json
"scripts": {
  "build": "tsc -b && vite build",
  "build:prerender": "npm run build:dev && node scripts/prerender.mjs",
  "build:production": "tsc -b && vite build --mode production && node scripts/prerender.mjs"
}
```

- **`build:prerender`** , development build + prerender (uses `localhost:8080` API, for local testing)
- **`build:production`** , production build + prerender (uses `https://api.longtermemory.com`)

The prerender step requires the backend API to be reachable during the build, because `LandingPublic` fetches live pricing. In the CI/CD pipeline this means the production build runs against the live API.

---

## What This Solves and What It Doesn't

**Solved:**
- All landing page text (hero copy, feature descriptions, FAQ answers) is in the raw HTML , Google indexes it in the first crawl wave, no JavaScript execution required
- `og:title`, `og:description`, `og:image`, `twitter:card` are baked into the HTML , social share previews work correctly on all platforms
- JSON-LD structured data with real pricing is present , eligible for Google Rich Results (price, availability, product type)
- `<title>` and `<meta name="description">` exist as static HTML, not injected by JavaScript , the minimal fallback in `index.html` is a backup, not the primary

**Not solved:**
- Dynamic routes (`/study-plan/pr/:id`, `/study-session/pr/:id`, `/dashboard`) are not prerendered , they require authentication anyway, so they don't need to be indexed
- The `/privacy` and `/terms` routes are plain React pages with no prerendering , they're text-heavy and could benefit from it, but haven't been a priority
- On-page SEO beyond the landing page (canonical tags, sitemap) is handled separately

---

## The Fallback Layer: `index.html`

Before the prerender script runs, `index.html` contains minimal static meta tags as a safety net:

```html
<title>LongTerm Memory - AI-Powered Study & Exam Preparation</title>
<meta name="description" content="Master any subject with AI-powered question-answer generation and spaced repetition..." />
```

If the prerender fails (API unreachable, timeout, Puppeteer crash), the build still succeeds and the fallback tags are served. They're not as rich as the fully rendered HTML , no OG tags, no JSON-LD , but they're better than nothing and they prevent the build pipeline from blocking on an SEO failure.

---

## What I'd Do Differently

**Prerender `/privacy` and `/terms` too.** These pages are static content and would benefit from prerendering. The current script only handles `/`. Extending it to run against multiple routes and write each to its own `dist/{path}/index.html` would be straightforward.

**Decouple pricing from the prerender.** Requiring the live API to be reachable during the build is a fragile dependency. A better approach: cache the pricing data in a JSON file committed to the repo (updated by a separate scheduled job), and have `LandingPublic` read from that file during prerendering. The build would then be fully offline-capable.

**Add a prerender verification step.** The script has no post-check that the output HTML actually contains expected content. A simple `grep` for a known FAQ string or a pricing number would catch cases where the API timed out and the HTML was captured in a loading state.

---

The full setup , `LandingRoute`, `LandingPublic`, `LazySection`, and the prerender script , took about a day to build and deploy. The authenticated app was entirely untouched. Google Search Console now shows the landing page content indexed in the first crawl, and social share previews work correctly across all platforms.
