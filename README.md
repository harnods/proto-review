# @ds/proto-review

Figma-style comment annotations for Vue/Nuxt prototypes — drop pins anywhere on
a page, thread replies, resolve, and toggle visibility. Backed by Supabase so
comments persist and sync across anyone with the link, no login required.

Built for sharing design prototypes with PMs/engineers who can't leave Figma
comments on a coded prototype.

## Install

```bash
npm install github:harnods/proto-review
```

Peer dependencies (install alongside if not already present):

```bash
npm install @supabase/supabase-js vue vue-router
```

If your project bundles with Vite/Nuxt, add the package to `transpile` (Nuxt)
or make sure your bundler doesn't skip `node_modules/@ds/proto-review` — it
ships as source `.ts`/`.vue`, not a pre-built bundle.

## Quick setup (Nuxt)

After installing, run the bundled init script — it patches `nuxt.config.ts`
(`build.transpile`), creates `plugins/proto-review.client.ts` pre-filled with
the shared Supabase project (see below), and mounts `<ProtoReviewOverlay />`
in your `app.vue`. Safe to re-run — it skips anything already set up.

```bash
npm install github:harnods/proto-review
npx proto-review init
```

The generated plugin defaults to the shared `proto-review` Supabase project
so every prototype can reuse the same database — just make sure `projectId`
(auto-filled from your `package.json` name) is unique per prototype so
comments don't mix. Open `plugins/proto-review.client.ts` afterwards to
double check or swap in your own Supabase project.

**The floating "Review" button is a fallback, not the goal.** Before writing
the plugin file, init scans your project for something that looks like an
existing user/account menu (a `.vue` file mentioning both a sign-out action
and `MpAvatar`/`MpPopover`/a menu-ish component name) and asks:

```
Add the Review mode toggle to an existing user menu instead of a floating button? (Y/n)
Path to that file [components/AppUserMenu.vue]:
```

Say yes and it bakes `showLauncher: false` into the plugin and prints the
exact snippet to paste into that file (import `useReviewMode`, add a button
that calls `toggleReviewMode`). Say no, or if nothing was found, or if you're
running init non-interactively (no TTY), it defaults to the floating button
— so the toggle always exists somewhere, it just prefers to live where users
already expect account-level settings.

Not a Nuxt project, or want to do it by hand? See Manual setup below.

## Manual setup

### 0. Nuxt transpile config

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  build: { transpile: ['@ds/proto-review'] },
})
```

### 1. Supabase tables

Run this once against the Supabase project you want to store comments in
(one project can serve comments for many prototypes — see `projectId` below):

```sql
create table proto_review_annotations (
  id uuid primary key default gen_random_uuid(),
  project_id text not null,
  route_key text not null,
  path text, -- concrete URL at creation time, used by the All comments panel to navigate
  x_pct float not null,  -- viewport %, fallback position
  y_pct float not null,
  anchor_selector text,  -- CSS selector of the element the pin is attached to
  anchor_x_pct float,    -- offset within that element, % of its box
  anchor_y_pct float,
  author text not null default 'Anonymous',
  body text not null,
  resolved boolean default false,
  created_at timestamptz default now()
);

create table proto_review_replies (
  id uuid primary key default gen_random_uuid(),
  annotation_id uuid references proto_review_annotations(id) on delete cascade,
  author text not null default 'Anonymous',
  body text not null,
  created_at timestamptz default now()
);

alter table proto_review_annotations enable row level security;
alter table proto_review_replies enable row level security;

create policy "allow_all_annotations" on proto_review_annotations for all using (true) with check (true);
create policy "allow_all_replies" on proto_review_replies for all using (true) with check (true);
```

### 2. Register the plugin (skip if you used `npx proto-review init`)

```ts
// plugins/proto-review.client.ts (Nuxt) or main.ts (Vue)
import { createProtoReview } from '@ds/proto-review'

app.use(createProtoReview({
  supabaseUrl: 'https://<your-project>.supabase.co',
  supabaseKey: '<anon-key>',
  projectId: 'my-new-prototype', // unique per prototype sharing the same Supabase project
  showLauncher: true, // optional, default true — see below
}))
```

### 3. Mount the overlay (skip if you used `npx proto-review init`)

```vue
<!-- App.vue -->
<template>
  <RouterView />
  <ProtoReviewOverlay />
</template>
```

## Using it

- Anyone opens the app and clicks **Review** (bottom-right floating button) —
  or visits any URL with `?review` appended — to turn on review mode.
- First time, they type a name. Every comment/pin from that name gets the
  same color automatically (hashed from the name), so threads stay easy to
  scan across all pinned pages.
- Press **`c`** (or click **+ Add Comment**) to start dropping a pin, then
  click anywhere on the page to place it. The comment box opens already
  focused, so you can type right away. The pin anchors to the DOM element
  under the cursor, so it scrolls with the content and stays glued to what
  was commented on even when the layout shifts. If that element ever
  disappears, the pin falls back to its viewport position.
- Drag any pin to reposition it — a quick click still opens its popover;
  the new position (and its new element anchor) saves as soon as you let go.
- Comments are anchored to a normalized route pattern (`/warehouses/:id`, not
  the exact id) — so all instances of a detail-page-shaped route share the
  same thread. Different pages (`/warehouse` vs `/product`) never bleed into
  each other.
- Review mode persists across page navigation for the browser session
  (`sessionStorage`), so it doesn't drop when the app's router changes the URL.
- **Show comments** (in the toolbar) opens a cross-page inbox of every comment
  in the project, in a dark panel that pushes the whole app left instead of
  covering it. Clicking a comment navigates to the exact page it was left on
  and opens its popover automatically.

## Cleaning up comments when you reset demo data

If your prototype has a "reset demo data" action (wiping seed/localStorage
data back to a clean slate), pages with dynamically-created records — e.g.
`/warehouses/wh-042` — no longer exist after the reset, but any comment
pinned there still does. The pin ends up on an empty page.

Call `clearDynamicAnnotations()` alongside your own reset, before reloading:

```ts
import { clearDynamicAnnotations } from '@ds/proto-review'

async function resetDemoData() {
  myAppResetFunction()
  await clearDynamicAnnotations().catch(() => {}) // don't block the reset on a network hiccup
  window.location.reload()
}
```

This deletes every comment anchored to a parameterized route (any `route_key`
containing `:id`, e.g. `/warehouses/:id`) for your `projectId`. Comments on
static pages (`/warehouse`, `/settings`, anything without an id segment)
aren't touched — that feedback isn't tied to demo data, so it survives.

## Migrating an existing database

If you set up the tables before the `path` column existed, add it:

```sql
alter table proto_review_annotations add column if not exists path text;
```

If you set them up before element anchoring (v0.2.0), add:

```sql
alter table proto_review_annotations
  add column if not exists anchor_selector text,
  add column if not exists anchor_x_pct float,
  add column if not exists anchor_y_pct float;
```

Rows without an anchor keep working — they render at their stored viewport
position, exactly as before.

Older rows without a `path` fall back to their `route_key` in the All
comments panel (works for static pages, not exact for parameterized ones).

## Own toggle instead of the floating launcher

If your project already has a user/account menu, wire the toggle there and
hide the package's own floating button:

```ts
createProtoReview({ /* ...config */, showLauncher: false })
```

```vue
<script setup>
import { useReviewMode } from '@ds/proto-review'
const { isReviewMode, toggleReviewMode } = useReviewMode()
</script>

<template>
  <button @click="toggleReviewMode">
    Review mode <span v-if="isReviewMode">On</span>
  </button>
</template>
```

## Moving the toolbar off bottom-right

The floating launcher and the review toolbar dock to the bottom-right corner
by default. If your app already puts something there (a scenario/environment
switcher, a chat widget, etc.), move proto-review to the opposite corner:

```ts
createProtoReview({ /* ...config */, corner: 'bottom-left' })
```

## Updating this package

This ships as source (no build step). To pull the latest version into a
consumer project:

```bash
npm install github:harnods/proto-review
```

To change the package itself, clone this repo, edit, commit, push — then
re-run `npm install` in each consumer project to pick up the new commit.
