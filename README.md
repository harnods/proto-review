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

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  build: { transpile: ['@ds/proto-review'] },
})
```

## Setup

### 1. Supabase tables

Run this once against the Supabase project you want to store comments in
(one project can serve comments for many prototypes — see `projectId` below):

```sql
create table proto_review_annotations (
  id uuid primary key default gen_random_uuid(),
  project_id text not null,
  route_key text not null,
  x_pct float not null,
  y_pct float not null,
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

### 2. Register the plugin

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

### 3. Mount the overlay

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
- Click anywhere on the page to drop a pin and leave a comment.
- Comments are anchored to a normalized route pattern (`/warehouses/:id`, not
  the exact id) — so all instances of a detail-page-shaped route share the
  same thread. Different pages (`/warehouse` vs `/product`) never bleed into
  each other.
- Review mode persists across page navigation for the browser session
  (`sessionStorage`), so it doesn't drop when the app's router changes the URL.

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

## Updating this package

This ships as source (no build step). To pull the latest version into a
consumer project:

```bash
npm install github:harnods/proto-review
```

To change the package itself, clone this repo, edit, commit, push — then
re-run `npm install` in each consumer project to pick up the new commit.
