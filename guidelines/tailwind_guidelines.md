# TailwindCSS Guidelines
## For HTMX + Jinja2 + FastAPI projects (NEESH family)

---

## Table of Contents

1. [Setup](#setup)
2. [Build Workflow](#build-workflow)
3. [Brand Theming](#brand-theming)
4. [Dark Mode](#dark-mode)
5. [Base Template Wiring](#base-template-wiring)
6. [Layout & Containers](#layout--containers)
7. [Card Patterns](#card-patterns)
8. [Typography](#typography)
9. [Buttons](#buttons)
10. [Forms](#forms)
11. [Tables](#tables)
12. [Alerts & Badges](#alerts--badges)
13. [Nav Active State Pattern](#nav-active-state-pattern)
14. [When to use @layer vs inline utilities](#when-to-use-layer-vs-inline-utilities)
15. [Dark Mode Caveats](#dark-mode-caveats)
16. [What NOT to do](#what-not-to-do)

---

## Setup

### package.json

Only one devDependency — tailwindcss itself. No PostCSS plugin needed for v3 CLI.

```json
{
  "scripts": {
    "build:css": "tailwindcss -i ./static/css/input.css -o ./static/css/output.css --minify",
    "watch:css": "tailwindcss -i ./static/css/input.css -o ./static/css/output.css --watch"
  },
  "devDependencies": {
    "tailwindcss": "^3.4.0"
  }
}
```

### tailwind.config.js

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  // Scan all Jinja2 templates and any JS that constructs class strings
  content: [
    "./app/templates/**/*.html",
    "./static/js/**/*.js",
  ],
  // Use class-based dark mode — toggled by adding `dark` to <html>
  darkMode: "class",
  theme: {
    extend: {
      colors: {
        // Brand palette — replace with your project's colours
        'brand-primary':   '#10b981',  // emerald-500
        'brand-secondary': '#3b82f6',  // blue-500
        'brand-accent':    '#f59e0b',  // amber-500
      },
    },
  },
  plugins: [],
}
```

### static/css/input.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ─── HTMX loading indicators ───────────────────────────────────── */
.htmx-indicator        { display: none; }
.htmx-request .htmx-indicator   { display: inline; }
.htmx-request.htmx-indicator    { display: inline; }

/* ─── Reusable component classes (@layer keeps specificity low) ─── */
@layer components {
  /* Buttons */
  .btn-primary {
    @apply bg-brand-primary hover:bg-emerald-600 text-white font-semibold
           py-2 px-4 rounded-lg transition-colors;
  }
  .btn-secondary {
    @apply bg-brand-secondary hover:bg-blue-600 text-white font-semibold
           py-2 px-4 rounded-lg transition-colors;
  }
  .btn-danger {
    @apply bg-red-600 hover:bg-red-700 text-white font-semibold
           py-2 px-4 rounded-lg transition-colors;
  }
  .btn-ghost {
    @apply text-gray-600 dark:text-gray-300 hover:text-gray-900
           dark:hover:text-white font-medium py-2 px-4 rounded-lg
           hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors;
  }

  /* Form inputs */
  .input-field {
    @apply w-full px-3 py-2 border border-gray-300 dark:border-gray-600
           dark:bg-gray-700 dark:text-white rounded-lg shadow-sm
           focus:ring-2 focus:ring-brand-primary focus:border-transparent
           sm:text-sm;
  }
  .input-label {
    @apply block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1;
  }
}
```

---

## Build Workflow

### Development (Windows)

`start_dev.bat` launches the watcher in a background terminal window:

```bat
if exist package.json (
    if not exist node_modules (call npm install --silent)
    start "Tailwind Watcher" /MIN cmd /c "npm run watch:css"
)
```

Save any `.html` template → watcher rebuilds `output.css` in under a second.

### Production / Deployment

**Rule: build on dev machine, commit `output.css`, never build on the server.**

```bash
# On your dev machine before pushing
npm run build:css
git add static/css/output.css
git commit -m "Rebuild CSS"
git push
```

Deployment servers (Raspberry Pi, BeagleBone, VPS) do not need Node.js installed.
`deploy.sh` only checks that the file exists:

```bash
if [ ! -f "static/css/output.css" ]; then
    warn "output.css missing — build on dev machine and commit."
fi
```

### Why commit output.css?

- Servers are often ARM with no Node.js
- Avoids a build step in deploy pipeline
- Makes the CSS diff reviewable in PRs — you can see exactly which classes changed

---

## Brand Theming

Define your project palette once in `tailwind.config.js`. Use the `brand-*` prefix
so colours are identifiable as project-specific rather than Tailwind built-ins.

```js
// tailwind.config.js
colors: {
  'brand-primary':   '#10b981',  // main action colour
  'brand-secondary': '#3b82f6',  // secondary / info
  'brand-accent':    '#f59e0b',  // warnings / highlights
  'brand-danger':    '#ef4444',  // destructive actions
}
```

Usage in templates:
```html
<button class="bg-brand-primary hover:bg-emerald-600 text-white ...">Save</button>
<span class="text-brand-accent">₹ 1,23,456</span>
```

---

## Dark Mode

### Enable it

`darkMode: "class"` in `tailwind.config.js`. A `dark` class on `<html>` activates it.

```html
<!-- base.html -->
<html lang="en" class="h-full dark">  <!-- hardcode for always-dark -->
```

For a user-toggled theme, add/remove the `dark` class via JS:

```js
// Toggle dark mode
document.documentElement.classList.toggle('dark');
// Persist to localStorage
localStorage.setItem('theme', document.documentElement.classList.contains('dark') ? 'dark' : 'light');
// On page load (before render to avoid flash):
if (localStorage.theme === 'dark') document.documentElement.classList.add('dark');
```

### Pairing every colour with its dark variant

Every foreground/background class must have a `dark:` counterpart. Common pairs:

| Light | Dark | Use |
|-------|------|-----|
| `bg-white` | `dark:bg-gray-800` | Card / panel background |
| `bg-gray-50` | `dark:bg-gray-900` | Page background |
| `text-gray-900` | `dark:text-white` | Headings |
| `text-gray-500` | `dark:text-gray-400` | Muted text / labels |
| `text-gray-700` | `dark:text-gray-300` | Body / secondary text |
| `border-gray-200` | `dark:border-gray-700` | Dividers |
| `bg-gray-100` | `dark:bg-gray-700` | Hover states |

### body tag

```html
<body class="h-full bg-gray-50 dark:bg-gray-900">
```

---

## Base Template Wiring

Minimum `base.html`:

```html
<!DOCTYPE html>
<html lang="en" class="h-full">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}App{% endblock %}</title>

    <!-- Built CSS — no CDN, no inline styles -->
    <link rel="stylesheet" href="/static/css/output.css">

    <!-- HTMX — vendored, not CDN -->
    <script src="/static/js/htmx.min.js"></script>

    <!-- Per-page scripts (Chart.js, etc.) -->
    {% block head_scripts %}{% endblock %}
</head>
<body class="h-full bg-gray-50 dark:bg-gray-900">
    {% include "components/nav.html" %}

    <!-- Flash messages -->
    {% if messages %}
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 mt-4">
        {% for msg in messages %}
        <div class="rounded-md p-4 mb-2
            {% if msg.type == 'error' %}bg-red-50 text-red-800
            {% elif msg.type == 'success' %}bg-green-50 text-green-800
            {% else %}bg-blue-50 text-blue-800{% endif %}">
            {{ msg.text }}
        </div>
        {% endfor %}
    </div>
    {% endif %}

    <main>{% block content %}{% endblock %}</main>

    {% block scripts %}{% endblock %}
</body>
</html>
```

---

## Layout & Containers

### Page wrapper (use on every page's outer div)

```html
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
```

`max-w-7xl` = 1280px cap. `px-4 sm:px-6 lg:px-8` gives responsive side padding.
`py-8` = consistent vertical breathing room.

### Narrow forms / single-column pages

```html
<div class="max-w-2xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
```

### Page header with action button

```html
<div class="flex justify-between items-center mb-8">
    <div>
        <h1 class="text-2xl font-bold text-gray-900 dark:text-white">Page Title</h1>
        <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">Subtitle / context</p>
    </div>
    <a href="/items/add" class="btn-primary">Add Item</a>
</div>
```

### Responsive grid (Tailwind utility classes)

```html
<!-- 1 col mobile → 2 col tablet → 4 col desktop -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
    ...
</div>
```

---

## Card Patterns

### The hybrid approach

Tailwind utility classes work great until dark mode requires complex background
combinations (e.g., specific slate tones that differ from gray). For these, define
`.card-panel` and `.card-grid` in the base template's `<style>` block so the classes
are guaranteed present even if Tailwind's purge misconfigures content paths.

```html
<!-- base.html <style> block -->
<style>
.card-panel {
    background: #ffffff;
    border-radius: 0.75rem;
    border: 1px solid #e5e7eb;       /* gray-200 */
    padding: 1.25rem;
    box-shadow: 0 1px 3px rgba(0,0,0,0.08);
}
.dark .card-panel {
    background: #1e293b;             /* slate-800 — warmer than gray-800 */
    border: 1px solid rgba(255,255,255,0.10);
    box-shadow: 0 1px 8px rgba(0,0,0,0.5), inset 0 1px 0 rgba(255,255,255,0.05);
}

/* Responsive card grids */
.card-grid   { display: grid; grid-template-columns: 1fr; gap: 1.5rem; }
@media (min-width: 768px) {
    .card-grid   { grid-template-columns: repeat(2, 1fr); }
    .card-grid-3 { grid-template-columns: repeat(3, 1fr); }
    .card-grid-4 { grid-template-columns: repeat(4, 1fr); }
}
@media (min-width: 1280px) {
    .card-grid-auto4 { grid-template-columns: repeat(4, 1fr); }
}
</style>
```

Usage:

```html
<!-- 4-column stat cards -->
<div class="card-grid card-grid-4 mb-8">
    <div class="card-panel">
        <p class="text-3xl font-bold text-indigo-600 dark:text-indigo-400">42</p>
        <p class="text-gray-600 dark:text-gray-400 mt-1">Items</p>
    </div>
    ...
</div>

<!-- 2-column content cards -->
<div class="card-grid mb-8">
    <div class="card-panel">...</div>
    <div class="card-panel">...</div>
</div>
```

### Standard Tailwind card (when dark mode is simple)

```html
<div class="bg-white dark:bg-gray-800 rounded-lg shadow p-6">
    ...
</div>
```

Use `.card-panel` for dashboard stat cards and content panels.
Use the inline version for simple one-off cards in forms/pages.

---

## Typography

```html
<!-- Page heading -->
<h1 class="text-2xl font-bold text-gray-900 dark:text-white">Title</h1>

<!-- Section heading -->
<h2 class="text-lg font-semibold text-gray-900 dark:text-white">Section</h2>

<!-- Body text -->
<p class="text-sm text-gray-600 dark:text-gray-400">Description</p>

<!-- Muted / helper text -->
<p class="text-xs text-gray-400 dark:text-gray-500">Last updated 5 min ago</p>

<!-- Large data value (stat cards) -->
<p class="text-2xl font-bold text-gray-900 dark:text-white">₹1,23,456</p>

<!-- Positive / negative values -->
<span class="text-green-600 dark:text-green-400">▲ 3.2%</span>
<span class="text-red-600 dark:text-red-400">▼ 1.1%</span>
```

---

## Buttons

Defined in `@layer components` in `input.css` — use the class names directly:

```html
<!-- Primary action -->
<button class="btn-primary">Save</button>
<a href="/items/add" class="btn-primary">Add Item</a>

<!-- Secondary / cancel -->
<button class="btn-secondary">View Details</button>

<!-- Destructive -->
<button class="btn-danger">Delete</button>

<!-- Ghost / subtle -->
<button class="btn-ghost">Cancel</button>
```

For icon buttons or custom sizing, build inline:

```html
<button class="inline-flex items-center gap-2 px-4 py-2 text-sm font-medium
               rounded-md text-white bg-indigo-600 hover:bg-indigo-700
               transition-colors">
    <svg class="w-4 h-4" ...></svg>
    Export
</button>
```

### Button groups (tab toggle)

```html
<div class="flex rounded-md shadow-sm">
    <a href="/view/list"
       class="px-4 py-2 text-sm font-medium bg-indigo-600 text-white rounded-l-md">
        List
    </a>
    <a href="/view/grid"
       class="px-4 py-2 text-sm font-medium bg-white dark:bg-gray-700
              text-gray-700 dark:text-gray-200 rounded-r-md
              border border-gray-300 dark:border-gray-600
              hover:bg-gray-50 dark:hover:bg-gray-600">
        Grid
    </a>
</div>
```

---

## Forms

### Form container

```html
<div class="bg-white dark:bg-gray-800 rounded-lg shadow">
    <!-- Header -->
    <div class="px-6 py-4 border-b border-gray-200 dark:border-gray-700">
        <h1 class="text-xl font-semibold text-gray-900 dark:text-white">Add Item</h1>
    </div>
    <!-- Body -->
    <form method="POST" class="p-6 space-y-6">
        ...
    </form>
</div>
```

### Field + label

```html
<div>
    <label for="name" class="input-label">
        Name <span class="text-red-500">*</span>
    </label>
    <input type="text" name="name" id="name" required
           class="input-field">
</div>
```

### Select

```html
<select name="type" id="type" class="mt-1 block w-full rounded-md
    border-gray-300 dark:border-gray-600 dark:bg-gray-700 dark:text-white
    shadow-sm focus:border-indigo-500 focus:ring-indigo-500 sm:text-sm">
    <option value="">Choose...</option>
    <option value="a">Option A</option>
</select>
```

### Two-column field row

```html
<div class="grid grid-cols-2 gap-4">
    <div>
        <label class="input-label">Date</label>
        <input type="date" name="date" class="input-field">
    </div>
    <div>
        <label class="input-label">Amount</label>
        <input type="number" name="amount" class="input-field">
    </div>
</div>
```

### Form action row

```html
<div class="flex justify-end gap-3 pt-4 border-t border-gray-200 dark:border-gray-700">
    <a href="/items" class="btn-ghost">Cancel</a>
    <button type="submit" class="btn-primary">Save</button>
</div>
```

---

## Tables

```html
<div class="overflow-x-auto">
    <table class="min-w-full divide-y divide-gray-200 dark:divide-gray-700">
        <thead class="bg-gray-50 dark:bg-gray-800">
            <tr>
                <th class="px-6 py-3 text-left text-xs font-medium
                           text-gray-500 dark:text-gray-400 uppercase tracking-wider">
                    Name
                </th>
                <th class="px-6 py-3 text-right text-xs font-medium
                           text-gray-500 dark:text-gray-400 uppercase tracking-wider">
                    Value
                </th>
            </tr>
        </thead>
        <tbody class="bg-white dark:bg-gray-900
                      divide-y divide-gray-200 dark:divide-gray-700">
            {% for item in items %}
            <tr class="hover:bg-gray-50 dark:hover:bg-gray-800 transition-colors">
                <td class="px-6 py-4 text-sm text-gray-900 dark:text-white">
                    {{ item.name }}
                </td>
                <td class="px-6 py-4 text-sm text-right font-mono
                           text-gray-900 dark:text-white">
                    {{ item.value | format_number }}
                </td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
</div>
```

---

## Alerts & Badges

### Alert banner (flash message style)

```html
<!-- Success -->
<div class="rounded-md p-4 bg-green-50 dark:bg-green-900/30
            text-green-800 dark:text-green-200" role="alert">
    Item saved successfully.
</div>

<!-- Error -->
<div class="rounded-md p-4 bg-red-50 dark:bg-red-900/30
            text-red-800 dark:text-red-200" role="alert">
    Something went wrong.
</div>

<!-- Warning -->
<div class="rounded-md p-4 bg-yellow-50 dark:bg-yellow-900/30
            text-yellow-800 dark:text-yellow-200" role="alert">
    Please review before continuing.
</div>
```

Extract to `components/alert.html` when reused across multiple templates.

### Badges / pills

```html
<!-- Status badge -->
<span class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs
             font-medium bg-green-100 text-green-800
             dark:bg-green-900/30 dark:text-green-300">
    Active
</span>

<!-- Count badge (e.g., on nav icon) -->
<span class="inline-flex items-center justify-center w-5 h-5
             rounded-full text-xs font-bold
             bg-red-500 text-white">
    3
</span>
```

---

## Nav Active State Pattern

Use Jinja2 conditional to toggle active vs inactive classes on the same element.
Keep both branches in one `class="..."` attribute — don't split with if/else blocks.

```html
<a href="/dashboard"
   class="{% if request.url.path == '/dashboard' %}
              border-indigo-500 text-gray-900 dark:text-white
          {% else %}
              border-transparent text-gray-500 hover:border-gray-300
              hover:text-gray-700 dark:text-gray-300 dark:hover:text-white
          {% endif %}
          inline-flex items-center px-1 pt-1 border-b-2 text-sm font-medium">
    Dashboard
</a>
```

For prefix matching (sub-routes should also highlight the parent nav item):

```html
class="{% if '/holdings' in request.url.path %}border-indigo-500 ...{% else %}...{% endif %}"
```

Mobile nav uses `block pl-3 border-l-4` instead of `inline-flex border-b-2`:

```html
<a href="/dashboard"
   class="{% if request.url.path == '/dashboard' %}
              bg-indigo-50 border-indigo-500 text-indigo-700
          {% else %}
              border-transparent text-gray-600 hover:bg-gray-50
              hover:border-gray-300 hover:text-gray-800
          {% endif %}
          block pl-3 pr-4 py-2 border-l-4 text-base font-medium">
    Dashboard
</a>
```

---

## When to use @layer vs inline utilities

| Use `@layer components` in input.css | Use inline utilities in HTML |
|--------------------------------------|------------------------------|
| Classes used in 5+ places (buttons, inputs) | One-off layout (`max-w-2xl mx-auto`) |
| Dark mode combos that are fragile to repeat | Page-specific spacing (`mb-8`, `gap-4`) |
| HTMX indicator classes | Hover states that vary per element |
| Brand-colour shortcuts | Grid definitions that differ per page |

**Rule of thumb:** If you're copying the same 4+ class string for the third time, move it to `@layer components`. Otherwise, write it inline — premature abstraction in CSS is as harmful as in code.

---

## Dark Mode Caveats

### Select dropdowns

Browser-native `<select>` ignores most CSS in dark mode. Force background + text colour
explicitly (put this in the base template `<style>` block, not in input.css, so it
applies regardless of Tailwind build):

```html
<style>
.dark select,
.dark select option {
    background-color: rgb(55, 65, 81);  /* gray-700 */
    color: rgb(255, 255, 255);
}
select option {
    background-color: rgb(255, 255, 255);
    color: rgb(0, 0, 0);
}
</style>
```

### Opacity modifiers for dark overlays

Tailwind's `bg-opacity` utility doesn't work with custom colours. Use the slash syntax:

```html
<!-- Correct -->
<div class="dark:bg-red-900/30">

<!-- Broken with custom colours -->
<div class="dark:bg-red-900 dark:bg-opacity-30">
```

### Don't use `bg-black` directly in dark mode

`bg-black` (#000000) looks harsh. Use `dark:bg-gray-900` (the page background) or
`dark:bg-gray-800` / `dark:bg-slate-800` for elevated surfaces.

---

## What NOT to do

### Don't use CDN links in templates

```html
<!-- BAD — breaks offline, inconsistent builds, potential CORS -->
<link href="https://cdn.tailwindcss.com" rel="stylesheet">

<!-- GOOD — pre-built, vendored, works offline -->
<link rel="stylesheet" href="/static/css/output.css">
```

### Don't add arbitrary values for things Tailwind already has

```html
<!-- BAD -->
<div class="w-[384px] mt-[32px]">

<!-- GOOD — use the scale -->
<div class="w-96 mt-8">
```

Use arbitrary values only when you genuinely need an exact pixel measurement that has
no equivalent in the Tailwind scale (e.g., `w-[350px]` for a specific chart dimension).

### Don't inline critical layout in `<style>` tags on individual pages

If you need a custom class that affects layout, put it in `input.css` under
`@layer components`. Inline `<style>` blocks in individual templates are only
acceptable for page-specific overrides (e.g., a third-party widget background fix).

### Don't forget `content:` in tailwind.config.js

Every template directory that uses Tailwind classes must be listed. A missing path
means those classes are purged from `output.css` in production.

```js
// BAD — misses admin templates
content: ["./app/templates/dashboard/**/*.html"]

// GOOD — catches everything
content: ["./app/templates/**/*.html", "./static/js/**/*.js"]
```

### Don't rebuild CSS on the server

Never run `npm run build:css` in `deploy.sh`. Build on dev, commit the file, deploy
the committed file. See [Build Workflow](#build-workflow).
