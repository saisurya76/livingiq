# livingiq (hub) — Reference

Single static page (`index.html`), no backend. Deployed via GitHub Pages,
custom domain `livingiqweb.com` (see `CNAME`).

For the auth/payments architecture this hub links into, see the
companion `ARCHITECTURE_PAYMENTS.md` shipped alongside the
`livingiq-auth` and `dealiq` repos.

## Theme tokens (CSS variables, defined in `:root`)

```css
--bg: #0B1220;        /* page background */
--surface: #121A2B;   /* card background */
--surface-2: #17223A; /* secondary surface */
--line: #253048;      /* borders */
--text: #E7ECF5;      /* primary text */
--muted: #8592AD;     /* secondary text */
--teal: #5EEAD4;      /* accent — live status, links, primary CTA */
--amber: #F5B841;     /* accent — roadmap notes, avatar gradient */
```

Fonts: Space Grotesk (headings/wordmark), Inter (body), IBM Plex Mono
(labels, tags, monospace figures) — same three fonts used across every
LivingIQ app for visual consistency.

## Card pattern

Each app gets a `.card` inside the `.grid` of its themed section
(`#work`, `#assets`, `#protect`, `#spending`, `#self`, `#health`).

**Live app card:**
```html
<div class="card live">
<div class="card-top"><div class="card-name-group"><div class="card-icon" data-icon="owl"></div><div class="card-name">AppName</div></div><div class="status live">Live</div></div>
<div class="verb-tag">Assess</div>
<p>One or two sentence description.</p>
<a class="card-link" href="https://auth.livingiqweb.com/login?redirect_to=https://appname.livingiqweb.com" target="_blank" rel="noopener">appname.livingiqweb.com →</a>
</div>
```

**Coming-soon card:**
```html
<div class="card soon">
<div class="status soon badge-float">Coming soon</div>
<div class="card-body">
<div class="card-top"><div class="card-name-group"><div class="card-icon" data-icon="owl"></div><div class="card-name">AppName</div></div></div>
<div class="verb-tag">Assess</div>
<p>Description.</p>
<span class="card-link">In planning</span>
</div>
<div class="frost"></div>
</div>
```

**Important:** a live app card's link goes through the central login
redirect (`auth.livingiqweb.com/login?redirect_to=...`), never directly to
the app's own domain — this is what makes "click any app card" and "log in
once across the family" the same click. This was the exact pattern applied
when DealIQ replaced SalaryIQ's coming-soon slot.

## Mascot icons

Six hand-drawn SVG icons (`owl`, `turtle`, `car`, `piggy`, `cat`, `panda`)
defined once in the `ICONS` object in the first `<script>` block, reused
both for each card's small badge (`.card-icon[data-icon]`) and for the
"Meet the family" mascot strip. To add a new icon for a new product
category, add a new key to `ICONS` and reference it via `data-icon="..."`
on a `.card-icon` div — no other wiring needed, the fill-from-map logic
runs automatically on page load.

## Files in this repo

| File | Purpose |
|---|---|
| `index.html` | The entire site — single file, HTML/CSS/JS inline |
| `CNAME` | GitHub Pages custom domain config (`livingiqweb.com`) |
| `original.html` | Pre-redesign backup — kept for reference, not served |
| `assets/profile.jpg` | Founder photo used in the `#founder` section |

**Note:** `original.html` and `assets/profile.jpg` are not reconstructed
in this package — carry those two over from your existing repo checkout,
since one is a historical backup and the other is a binary image, neither
of which changed as part of the DealIQ card integration.

## Current card status (as of this package)

| Section | App | Status |
|---|---|---|
| Work | ProjectIQ | Live |
| Work | DealIQ | Live — replaced the former SalaryIQ "coming soon" slot |
| Assets | PropertyIQ | Live |
| Protect | AccidentIQ | Live |
| Spending | PriceIQ | Coming soon |
| Self | StyleIQ | Coming soon |
| Self | SpecsIQ | Coming soon |
| Health | HealthIQ | Planned, not started |
