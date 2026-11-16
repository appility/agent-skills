# Framework-free implementation

Use this with no framework, or translate it to the stack in hand (Rails, Django, Laravel, Express, Astro, HTMX, etc.).

## One endpoint, two responses

Let the same route answer both the baseline and the enhanced request. The browser's normal form post asks for HTML; your script asks for JSON.

```js
// Express example
app.post('/posts/:id/like', async (req, res) => {
  const liked = await toggleLike(req.params.id, req.user);
  if (req.accepts(['html', 'json']) === 'json') return res.json({ liked });
  res.redirect(303, `${req.get('Referer') ?? '/'}#post-${req.params.id}`); // POST → redirect → GET
});
```

## Enhancing any form

A small helper that upgrades forms marked `data-enhance`. Without it (or if it fails), the forms post normally.

```js
for (const form of document.querySelectorAll('form[data-enhance]')) {
  const status = form.querySelector('[role="status"]');
  let pending = false;

  form.addEventListener('submit', async (event) => {
    event.preventDefault();
    if (pending) return;
    pending = true;
    const button = event.submitter;
    button?.setAttribute('aria-disabled', 'true');

    try {
      const res = await fetch(form.action, {
        method: form.method,
        body: new FormData(form, event.submitter),
        headers: { Accept: 'application/json' }
      });
      if (!res.ok) throw new Error(res.statusText);
      form.dispatchEvent(new CustomEvent('enhanced:success', { detail: await res.json() }));
    } catch {
      if (status) status.textContent = 'Something went wrong. Please try again.';
    } finally {
      pending = false;
      button?.removeAttribute('aria-disabled');
    }
  });
}
```

Notes: `new FormData(form, submitter)` includes the clicked button's `name`/`value` (current browsers). For `method="get"` forms, build the URL from `new URLSearchParams(new FormData(form))` instead of sending a body.

## Example: dependent select (WCAG 3.2.2)

The server renders the full page from `?code=`; a tiny JSON endpoint returns the same data for the background fetch.

### Server

One function produces the data. The page route and the JSON route both call it.

```js
// Express example — adapt to your server.
const descriptionsByCode = {
  EDUC: ['Educational Assistance', 'Educational Assistance CL1A'],
  SUBS: ['Subs and Professional Fees', 'Subscriptions and Fees']
};

const getDescriptions = (code = '') => ({ code, descriptions: descriptionsByCode[code] ?? [] });

app.get('/benefits/add', (req, res) => {
  res.render('add-benefit', getDescriptions(req.query.code)); // full page
});

app.get('/benefits/descriptions.json', (req, res) => {
  res.json(getDescriptions(req.query.code)); // background fetch
});
```

### HTML (server-rendered)

```html
<form method="get" action="/benefits/add" data-dependent-select>
  <label for="code">Benefit code</label>
  <select id="code" name="code" data-parent>
    <option value="">Choose a benefit code</option>
    <option value="EDUC">EDUC — Education</option>
    <option value="SUBS" selected>SUBS — Subscriptions</option>
  </select>

  <button type="submit" data-fallback>Continue</button>

  <p role="status" aria-live="polite" data-status></p>

  <label for="description">Description</label>
  <select id="description" name="description" data-child>
    <option>Subs and Professional Fees</option>
    <option>Subscriptions and Fees</option>
  </select>
</form>
```

### Script

Loaded with `defer` or `type="module"`. If it never runs, the button stays and the form works.

```js
const form = document.querySelector('[data-dependent-select]');

if (form) {
  const parent = form.querySelector('[data-parent]');
  const child = form.querySelector('[data-child]');
  const status = form.querySelector('[data-status]');
  const fallback = form.querySelector('[data-fallback]');
  let controller;

  const announce = (code, count) => {
    status.textContent = code
      ? `${count} ${count === 1 ? 'description' : 'descriptions'} for ${code}`
      : 'No benefit code chosen';
  };

  parent.addEventListener('change', async () => {
    controller?.abort(); // fast arrow presses: only the latest request counts
    controller = new AbortController();
    const code = parent.value;
    child.disabled = true;

    try {
      const res = await fetch(`/benefits/descriptions.json?code=${encodeURIComponent(code)}`, {
        signal: controller.signal
      });
      if (!res.ok) throw new Error(res.statusText);
      const { descriptions } = await res.json();

      // Replace options, clearing any stale choice.
      child.replaceChildren(...descriptions.map((d) => new Option(d, d)));
      announce(code, descriptions.length);
      child.disabled = false;
    } catch (error) {
      if (error.name === 'AbortError') return; // a newer request owns the UI now
      status.textContent = 'Could not load descriptions. Use Continue to try again.';
      fallback.hidden = false;
      child.disabled = false;
    }
  });

  // Upgrade is wired: now, and only now, hide the fallback.
  fallback.hidden = true;
}
```

### Notes

- Don't remove the status `<p>` from the server HTML and add it with JS later; it must exist before its text changes.
- `hidden` on the button (rather than removing it) lets the error path bring it back.
- HTMX: `hx-get` on the parent with `hx-trigger="change"`, `hx-target` on the child, and `hx-push-url="false"` gives the same behaviour. Still render the fallback button in the server HTML, hide it once htmx has initialised, and still update a live region.
- Never use `form.submit()` / `requestSubmit()` in the change handler — that's the navigation this pattern exists to avoid.
