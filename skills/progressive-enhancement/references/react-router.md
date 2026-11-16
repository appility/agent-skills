# React Router (v7 framework mode / Remix)

React Router is built for progressive enhancement: route `loader`s and `action`s are plain HTTP handlers, and `<Form>` and `<Link>` render real `<form>` and `<a>` elements. Most of the work is not breaking that.

## The building blocks

| Need | Baseline (no JS) | With JS |
| --- | --- | --- |
| Navigate | `<Link to>` renders `<a href>` | Client transition, optional `prefetch="intent"` |
| Read / search | `<Form method="get">` → loader reads `request.url` | Client navigation, URL updated |
| Write | `<Form method="post">` → route `action` | Submitted with fetch, loaders revalidate |
| Write in place (like, toggle) | `<fetcher.Form method="post">` is a normal POST | Background submit, page stays put |
| Load data without navigating | — (use a `get` Form with a button) | `fetcher.load(href)` |

Rules of thumb:

- **Mutations:** post to the current route's `action` with a hidden `intent` field (`<input type="hidden" name="intent" value="like" />`) and branch on it. With JS, `fetcher.Form` submits in the background and loaders revalidate. Without JS it's an ordinary POST and the same route re-renders with fresh data. Return a `redirect` after create-style actions so a refresh doesn't resubmit.
- **Validation errors:** return them from the action (`return data({ errors }, { status: 400 })`, with `data` imported from `react-router`) and render them from `useActionData()` (or `fetcher.data`). Works identically with and without JS.
- **Pending UI:** `useNavigation().state` for page-level submissions, `fetcher.state` for fetchers. Never render a pending or disabled state in the server HTML.
- **Optimistic UI:** read `fetcher.formData` while `fetcher.state !== 'idle'` to show the expected result; it falls back to real data once the action settles.
- **`onChange` must not cause a change of context.** No full reload, no focus jump, no scroll reset, no history entry per arrow press. Which tool depends on whether the URL should change:
  - **Filters and sort on a results page** (the URL should reflect them, so the view is shareable): `submit(event.currentTarget.form, { method: 'get', replace: true, preventScrollReset: true })`. It's a client-side update, focus stays put, and Back still works. Render an Apply button in the server HTML and remove it once hydrated.
  - **One field in a larger form** (a dependent select): `fetcher.load`, so the URL doesn't change at all (see the example below).
  - Avoid plain `navigate()` in a change handler: it pushes a history entry for every change.
- **Index routes:** a fetcher or form aimed at an index route needs `?index` in the URL, otherwise it hits the parent route.

## Example: like button

```tsx
import { useFetcher } from 'react-router';

import type { Route } from './+types/post';

export async function action({ request, params }: Route.ActionArgs) {
  const form = await request.formData();
  if (form.get('intent') === 'like') await likePost(params.postId, form.get('liked') === 'true');
  return null; // loaders revalidate
}

function LikeButton({ post }: { post: { id: string; liked: boolean } }) {
  const fetcher = useFetcher();
  // Optimistic: while submitting, show what we asked for.
  const liked = fetcher.formData ? fetcher.formData.get('liked') === 'true' : post.liked;

  return (
    <fetcher.Form method="post">
      <input name="intent" type="hidden" value="like" />
      <input name="liked" type="hidden" value={String(!liked)} />
      <button aria-pressed={liked} type="submit">
        {liked ? 'Liked' : 'Like'}
      </button>
    </fetcher.Form>
  );
}
```

## `useHydrated`

Use it to switch between baseline and enhanced UI. `useSyncExternalStore` returns the server snapshot during hydration, so server HTML and the first client render match, then it flips to `true`.

```ts
import { useSyncExternalStore } from 'react';

const subscribe = () => () => {};

export default function useHydrated() {
  return useSyncExternalStore(subscribe, () => true, () => false);
}
```

## Example: dependent select (WCAG 3.2.2)

1. **One loader** reads the parent value from the URL. `<Form method="get">` (full load) and `fetcher.load` (background) both call it.
2. **`useFetcher`** runs the loader without changing the URL, adding history or moving focus. A newer load cancels an older one.
3. **`useHydrated`** hides the Continue button and enables the fetch in the same step.
4. **Fetched data wins:** `fetcher.data?.x ?? data.x`.
5. **`key` on the child select** clears a stale choice.

```tsx
import { useEffect, useState } from 'react';

import { Form, useFetcher, useLoaderData } from 'react-router';

import useHydrated from '~/hooks/useHydrated';

const descriptionsByCode: Record<string, string[]> = {
  EDUC: ['Educational Assistance', 'Educational Assistance CL1A'],
  SUBS: ['Subs and Professional Fees', 'Subscriptions and Fees']
};

// Name the code, so two codes with the same count still produce different text.
const announcementFor = (code: string, count: number) => {
  if (!code) return 'No benefit code chosen';
  return `${count} ${count === 1 ? 'description' : 'descriptions'} for ${code}`;
};

// Same loader for Continue (full page) and for fetcher.load (page stays put).
export const loader = ({ request }: { request: Request }) => {
  const code = new URL(request.url).searchParams.get('code') ?? '';

  return { code, descriptions: descriptionsByCode[code] ?? [] };
};

export default function AddBenefit() {
  const data = useLoaderData<typeof loader>();
  const fetcher = useFetcher<typeof loader>();
  const isHydrated = useHydrated();
  const descriptions = fetcher.data?.descriptions ?? data.descriptions;
  const [announcement, setAnnouncement] = useState('');

  // Only after a background fetch. A full page load is already announced.
  useEffect(() => {
    if (!fetcher.data) return;
    setAnnouncement(announcementFor(fetcher.data.code, fetcher.data.descriptions.length));
  }, [fetcher.data]);

  return (
    <Form method="get">
      <label htmlFor="code">Benefit code</label>
      <select
        defaultValue={data.code}
        id="code"
        name="code"
        onChange={(event) => {
          // Keeps the two modes consistent: button visible means no fetch.
          if (!isHydrated) return;
          const nextCode = event.currentTarget.value;
          fetcher.load(nextCode ? `?code=${encodeURIComponent(nextCode)}` : '?');
        }}>
        <option value="">Choose a benefit code</option>
        <option value="EDUC">EDUC — Education</option>
        <option value="SUBS">SUBS — Subscriptions</option>
      </select>

      {/* In the server HTML. Removed after hydration. */}
      {isHydrated ? null : <button type="submit">Continue</button>}

      {/* Present and empty from first render. */}
      <p aria-live="polite" role="status">
        {announcement}
      </p>

      <label htmlFor="description">Description</label>
      <select id="description" key={descriptions.join('|')} name="description">
        {descriptions.map((description) => (
          <option key={description} value={description}>
            {description}
          </option>
        ))}
      </select>
    </Form>
  );
}
```
