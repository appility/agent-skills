# Next.js (App Router)

Server Components render real HTML, and Server Actions let a `<form>` post to the server with no client code. The traps are in Client Components.

## What works without JavaScript

- **Forms in Server Components.** A `<form action={serverAction}>` rendered by a Server Component submits even if JavaScript hasn't loaded yet or is disabled.
- **Forms in Client Components** queue submissions until hydration and are prioritised for hydrating. Don't count on them for a no-JS baseline.
- **`next/link`** renders a real `<a href>`.
- **Search params.** A `<form method="get">` (or `next/form` with a string `action`) navigates to `?q=…`, and the page reads `searchParams` on the server.

So: keep the `<form>` element in a Server Component, pass it a Server Action, and push only the interactive bits (pending button, optimistic list) into small Client Components inside it.

## Mutations

```tsx
// app/posts/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = String(formData.get('title') ?? '').trim();
  if (!title) redirect('/posts/new?error=title'); // no-JS-friendly error path
  await db.post.create({ data: { title } });
  revalidatePath('/posts');
  redirect('/posts');
}
```

```tsx
// app/posts/new/page.tsx (Server Component)
import { createPost } from '../actions';
import { SubmitButton } from './submit-button';

export default async function NewPost({ searchParams }: { searchParams: Promise<{ error?: string }> }) {
  const { error } = await searchParams;
  return (
    <form action={createPost}>
      <label htmlFor="title">Title</label>
      <input aria-describedby={error ? 'title-error' : undefined} id="title" name="title" required />
      {error === 'title' && <p id="title-error">Enter a title</p>}
      <SubmitButton />
    </form>
  );
}
```

```tsx
// app/posts/new/submit-button.tsx
'use client';

import { useFormStatus } from 'react-dom';

export function SubmitButton() {
  const { pending } = useFormStatus();
  // Server HTML renders an ordinary, enabled submit button.
  return <button aria-disabled={pending} type="submit">{pending ? 'Saving…' : 'Save'}</button>;
}
```

- `redirect()` after a successful action is the POST → redirect → GET pattern; refreshes don't resubmit.
- For richer error state with JS, `useActionState` in a Client Component returns state and a pending flag. Remember that a form owned by a Client Component won't submit before hydration; if no-JS matters, keep the error path in the URL or the server-rendered page as above.
- `useOptimistic` gives optimistic UI on top of a working form.

## Watch for

- `'use client'` at the top of a page or layout turns it, and everything it imports, into Client Components, so its forms no longer submit before hydration.
- `onClick={() => router.push(...)}` instead of `<Link>`.
- Filter selects with no baseline. Render a `get` form with an Apply button. Once hydrated, `onChange` can call `router.replace(url, { scroll: false })`: it updates in place, keeps focus and doesn't add history. Avoid `router.push`, which adds a history entry per change and scrolls to the top. A full reload or a focus jump on change would fail WCAG 3.2.2 (see `on-input.md`).
- Content fetched in `useEffect` that could have been fetched in a Server Component.
