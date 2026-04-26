# Browser Internals — "What Happens When You Type a URL"

## What it is
The full lifecycle of a web request: DNS resolution, TCP/TLS handshake, HTTP, HTML parsing, the render pipeline, and how the browser turns bytes into pixels.

## Why it matters
"What happens when you type a URL in the browser?" is asked in roughly 70% of frontend and full-stack interviews. It tests networking, HTTP, and rendering knowledge all at once. A weak answer signals shallow understanding.

---

## The Full Flow

```
You type: https://shop.example.com/products

1. URL Parse      — protocol (https), host (shop.example.com), path (/products)
2. DNS Lookup     — shop.example.com → 93.184.216.34
3. TCP Handshake  — establish connection to 93.184.216.34:443
4. TLS Handshake  — negotiate encryption (because https)
5. HTTP Request   — GET /products HTTP/2
6. Server Response — HTML bytes
7. HTML Parsing   — bytes → DOM
8. CSS Parsing    — bytes → CSSOM
9. Render Tree    — DOM + CSSOM merged
10. Layout        — calculate positions and sizes
11. Paint         — fill pixels
12. Composite     — GPU layers assembled, displayed
```

---

## Step 1 — DNS Resolution

Translates a human-readable hostname to an IP address.

```
Browser checks (in order):
1. Browser DNS cache    (recent lookups cached for TTL seconds)
2. OS DNS cache         (/etc/hosts on Linux, hosts file on Windows)
3. Router DNS cache
4. ISP's DNS resolver   (recursive resolver)
5. Root DNS servers     (knows where .com lives)
6. TLD DNS servers      (.com nameservers → knows example.com)
7. Authoritative DNS    (example.com nameserver → returns 93.184.216.34)

Result: IP address + TTL (time-to-live, how long to cache it)
```

**Why it matters for performance:** DNS lookup is 20–120ms. That's why sites prefetch DNS for third-party domains:
```html
<link rel="dns-prefetch" href="//cdn.example.com">
```

---

## Step 2 — TCP Handshake

TCP is connection-oriented — must establish a connection before sending data.

```
Client → SYN          →  Server   (I want to connect)
Client ← SYN-ACK      ←  Server   (OK, I'm ready)
Client → ACK          →  Server   (Great, let's go)

Cost: 1 round-trip (RTT) before any data can be sent
```

HTTP/2 and HTTP/3 mitigate this — one TCP connection reused for many requests (multiplexing).

---

## Step 3 — TLS Handshake (HTTPS)

Adds encryption on top of TCP. Happens after TCP connection.

```
1. Client Hello — supported TLS version, cipher suites, random bytes
2. Server Hello — chosen cipher suite, server certificate (public key)
3. Client verifies certificate (issued by trusted CA, not expired, hostname matches)
4. Key exchange — both sides derive the same session key without sending it
5. Encrypted communication begins

Cost: 1–2 additional RTTs (TLS 1.3 = 1 RTT, TLS 1.2 = 2 RTTs)
```

**Preconnect hint** — start TCP+TLS early for known third-party origins:
```html
<link rel="preconnect" href="https://api.example.com">
```

---

## Step 4 — HTTP Request & Response

```
GET /products HTTP/2
Host: shop.example.com
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, deflate, br
Cookie: session=abc123
```

Server responds with HTML (gzip/br compressed), CSS, JS references.

---

## Step 5-6 — HTML & CSS Parsing

### HTML → DOM

The browser parses HTML **incrementally** (doesn't wait for full file):
```
Bytes → Characters → Tokens → Nodes → DOM tree
```

When the parser hits `<script src="app.js">` — **it stops** and waits for the script to download and execute (parser-blocking). This is why scripts go at the bottom or use `defer`/`async`.

```html
<!-- Parser-blocking — delays page -->
<script src="app.js"></script>

<!-- async — downloads in parallel, executes as soon as ready (order not guaranteed) -->
<script async src="analytics.js"></script>

<!-- defer — downloads in parallel, executes after HTML parsed (order preserved) -->
<script defer src="app.js"></script>
```

### CSS → CSSOM

CSS is **render-blocking** — the browser won't render anything until all CSS is downloaded and parsed. This is why CSS goes in `<head>`.

---

## Step 7 — Render Tree

DOM + CSSOM combined. Only **visible** nodes (excludes `display: none`, `<head>`, `<script>`).

```
DOM:           CSSOM:          Render Tree:
<html>         body { }        body
  <body>       h1 { color }      h1 (with styles)
    <h1>       p { margin }      p  (with styles)
    <p>        .hidden           (display:none excluded)
    <div class="hidden">
```

---

## Step 8 — Layout (Reflow)

Calculate the exact position and size of every render tree node. Output: a box model for every element.

```
"Where does each element go and how big is it?"
Expensive — touches every affected element and its ancestors/descendants
```

---

## Step 9 — Paint

Fill pixels for each layer: text, colors, images, borders, shadows.

```
"What color is each pixel?"
```

---

## Step 10 — Composite

The browser splits the page into **layers** (GPU-accelerated), paints each separately, then the GPU assembles them into the final image.

```
Transforms and opacity changes happen at the composite layer —
they skip Layout and Paint entirely → very fast (60fps animations)
```

---

## Reflow vs Repaint vs Composite

| Operation | What triggers it | Cost | Examples |
|---|---|---|---|
| **Reflow (Layout)** | Size/position change | Most expensive | width, height, margin, padding, font-size, adding/removing DOM nodes |
| **Repaint** | Visual change, no layout | Medium | color, background, border-color, visibility |
| **Composite only** | Transform/opacity | Cheapest (GPU) | `transform: translateX()`, `opacity`, `will-change` |

**For smooth animations:** use `transform` and `opacity` — they only trigger composite.

```css
/* SLOW — triggers reflow + repaint on every frame */
.bad-animation { left: 0; transition: left 0.3s; }
.bad-animation:hover { left: 100px; }

/* FAST — only composite layer, GPU accelerated */
.good-animation { transform: translateX(0); transition: transform 0.3s; }
.good-animation:hover { transform: translateX(100px); }
```

---

## Layout Thrashing

Reading layout properties forces the browser to flush pending layout calculations — doing this in a loop is catastrophic.

```javascript
// LAYOUT THRASHING — read then write, repeat in loop
for (let i = 0; i < 1000; i++) {
  const width = element.offsetWidth;   // forces layout flush
  element.style.width = width + 1 + 'px'; // invalidates layout
}
// 1000 layout recalculations!

// FIX — batch reads first, then batch writes
const widths = elements.map(el => el.offsetWidth); // all reads
elements.forEach((el, i) => el.style.width = widths[i] + 1 + 'px'); // all writes
```

**Layout-triggering properties** (reading these forces a flush): `offsetWidth`, `offsetHeight`, `scrollTop`, `clientWidth`, `getBoundingClientRect()`.

---

## Critical Rendering Path

The sequence of steps the browser must complete before showing content.
```
HTML → DOM
CSS  → CSSOM
      ↓
   Render Tree → Layout → Paint → Composite → Display
```

**Optimizing it:**
- Minimize render-blocking CSS (critical CSS inline, rest deferred)
- Defer non-critical JS (`defer`, `async`, or move to bottom)
- Preload key resources: `<link rel="preload" as="font" href="...">` 
- Reduce DOM size — more nodes = more layout work

---

## Core Web Vitals (what Google measures)

| Metric | Measures | Target |
|---|---|---|
| **LCP** (Largest Contentful Paint) | When the largest content element is visible | < 2.5s |
| **CLS** (Cumulative Layout Shift) | Visual stability — elements jumping around | < 0.1 |
| **INP** (Interaction to Next Paint) | Responsiveness to user input | < 200ms |

---

## Common Interview Questions

1. Walk me through what happens when you type a URL and press Enter.
2. What is DNS and why do we need it?
3. What is the difference between reflow and repaint?
4. Why should CSS be in `<head>` and scripts at the bottom (or use `defer`)?
5. What is layout thrashing and how do you avoid it?
6. Why is `transform` better than `left/top` for animations?
7. What are Core Web Vitals?

---

## Common Mistakes

- Confusing DNS and HTTP ("DNS sends the request" — no, DNS only resolves the hostname)
- Saying TLS and HTTPS are different things (HTTPS = HTTP over TLS)
- Forgetting that CSS is render-blocking, not just JS
- Mixing up reflow/layout (position/size) with repaint (visual only)
- Not knowing that `transform` bypasses layout — it's the key to smooth animations

---

## How It Connects

- `defer` on your React app's `<script>` is why Vite puts scripts at the end
- CSS critical path is why Next.js inlines critical CSS per page
- Layout thrashing is why React batches DOM writes — you don't update the DOM directly
- TLS certificate validation is why HTTPS and valid certs are required in production
- HTTP/2 multiplexing (from `http-rest.md`) eliminates the multiple TCP connection workaround

---

## My Confidence Level
- `[ ]` DNS resolution — full lookup chain
- `[ ]` TCP handshake — SYN/SYN-ACK/ACK
- `[ ]` TLS handshake — certificate, key exchange
- `[ ]` HTML parsing → DOM, CSS parsing → CSSOM
- `[ ]` script async vs defer vs blocking
- `[ ]` Render tree, layout, paint, composite — the pipeline
- `[ ]` Reflow vs repaint vs composite-only
- `[ ]` Layout thrashing — what causes it, how to fix
- `[ ]` Core Web Vitals — LCP, CLS, INP

## My Notes
<!-- Personal notes -->
