---
name: cosmic
description: Build applications with Cosmic headless CMS using the JavaScript SDK. Covers Objects, Media, AI generation (text, images, video), content modeling with Metafields, and queries. Also covers agent self-signup (POST /v3/agents/sign-up) for provisioning a free-tier project + bucket when the user has no Cosmic account yet. Use when working with Cosmic CMS, @cosmicjs/sdk, headless content, building JAMstack applications, or onboarding a user who does not yet have Cosmic credentials.
---

# Cosmic CMS

Build content-powered applications with the [Cosmic](https://www.cosmicjs.com) headless CMS.

## SDK-First Principle

**Always use the `@cosmicjs/sdk` package.** The SDK provides type-safe methods, better error handling, and cleaner syntax than raw REST calls.

```bash
bun add @cosmicjs/sdk
# or: npm install @cosmicjs/sdk
```

## No Cosmic account? Provision one (Agent signup)

**When to use:** the user wants to build with Cosmic but has no `COSMIC_BUCKET_SLUG`/`COSMIC_READ_KEY`/`COSMIC_WRITE_KEY`, no `.env`, and no account on `app.cosmicjs.com`. Do **not** ask them to manually sign up: provision a project for them with one API call instead.

This flow is not in `@cosmicjs/sdk` yet (the SDK is bucket-scoped). Use `fetch` for the three signup endpoints, then drop the returned keys straight into `createBucketClient`.

### Step 1: Sign up (no auth required)

```typescript
const signup = await fetch('https://dapi.cosmicjs.com/v3/agents/sign-up', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    human_email: 'user@example.com',   // ASK THE USER, do not guess
    project_name: 'Recipe Blog',       // derive from what they're building
    agent_id: 'cursor',                // your platform: 'cursor', 'claude-code', etc.
  }),
}).then((r) => r.json());

// signup => {
//   agent_key:  'agk_...',           // persist this; needed for verify/status
//   bucket:     { slug, read_key, write_key },  // raw keys, ready to use
//   claim_url:  'https://app.cosmicjs.com/claim?token=agk_...',
//   limits:     { ai_credits_remaining: 0, media_mb_total: 5, objects_max: 50 },
//   auto_delete_after_days: 14,
// }
```

Immediately write the bucket keys + `agent_key` to `.env` (or wherever the project stores secrets), and gitignore them:

```bash
COSMIC_BUCKET_SLUG=recipe-blog-...
COSMIC_READ_KEY=...
COSMIC_WRITE_KEY=...
COSMIC_AGENT_KEY=agk_...
```

You can now use the SDK normally for objects, media, and reading. **AI generation is blocked** until step 3.

### Step 2: Tell the human about the OTP

Cosmic emailed the human a 6-digit code and a claim URL. Surface both:

```
A 6-digit verification code was sent to user@example.com.
Paste it here when you have it, or open this link to verify in the dashboard:
https://app.cosmicjs.com/claim?token=agk_...
```

You (the agent) will never see the code yourself: the user has to fetch it from their inbox and hand it back. Wait for them.

### Step 3: Verify (lifts restricted-mode limits)

```typescript
await fetch('https://dapi.cosmicjs.com/v3/agents/verify', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${process.env.COSMIC_AGENT_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ code: '123456' }),  // field is `code`, NOT `otp_code`
});
```

After this returns 200, the bucket is on standard free-tier limits and `cosmic.ai.*` works.

### Status & recovery

If you lose the sign-up response (e.g. user closed the chat), recover via:

```typescript
const status = await fetch('https://dapi.cosmicjs.com/v3/agents/status', {
  headers: { Authorization: `Bearer ${process.env.COSMIC_AGENT_KEY}` },
}).then((r) => r.json());
// status.bucket.{slug,read_key,write_key}, status.auth_type, status.limits
```

### Rules

- **`human_email` must come from the user.** Never make one up or use a placeholder. Ask if you don't have it.
- **The signup is idempotent** by `human_email` + `agent_id`. Re-running it issues a fresh OTP and returns the existing project instead of creating duplicates.
- **On `409 user_already_exists`**: stop. Do not retry with a different email. The response includes a `claim_existing_url`. Tell the user: "An account already exists for that email. Log in at `app.cosmicjs.com` to grant access to your existing buckets."
- **AI generation returns `402 { code: "agent_unclaimed_limit", action: "ai_generate" }` while unclaimed.** Catch that, and prompt the user to complete step 3 before retrying.
- **Restricted limits while unclaimed:** 50 objects, 5 MB total media, 0 AI credits.
- **Unclaimed projects are hard-deleted after 14 days.** Plan the build around getting to step 3 in the same session.
- **Persist `agent_key`** (not just bucket keys). Without it, verify/status are inaccessible and the user has to fall back to the dashboard claim URL.

If the user already has Cosmic credentials, **skip this section entirely** and jump to Setup below.

## Setup

```typescript
import { createBucketClient } from '@cosmicjs/sdk'

const cosmic = createBucketClient({
  bucketSlug: 'BUCKET_SLUG',
  readKey: 'BUCKET_READ_KEY',
  writeKey: 'BUCKET_WRITE_KEY' // Required for create/update/delete
})
```

Find keys in Cosmic Dashboard → Bucket → Settings → API Access.

## Objects (Content)

Objects are the core content units. Each Object belongs to an Object type (like "posts", "products").

### Read Objects

```typescript
// Get multiple Objects
const posts = await cosmic.objects
  .find({ type: 'posts' })
  .props(['id', 'title', 'slug', 'metadata'])
  .limit(10)

// Get single Object by slug
const post = await cosmic.objects
  .findOne({ type: 'posts', slug: 'hello-world' })
  .props(['title', 'slug', 'metadata'])

// Get single Object by id
const obj = await cosmic.objects
  .findOne({ id: 'object-id' })
  .props(['title', 'metadata'])
```

### Props Syntax

Use GraphQL-style syntax for nested properties:

```typescript
const props = `{
  id
  slug
  title
  metadata {
    content
    author {
      title
      metadata {
        avatar { imgix_url }
      }
    }
  }
}`
await cosmic.objects.find({ type: 'posts' }).props(props)
```

### Create Object

```typescript
await cosmic.objects.insertOne({
  title: 'My Post',
  type: 'posts', // Object type SLUG, not title
  metadata: {
    content: 'Post content here...',
    author: 'author-object-id',        // Object relationship: use id
    featured_image: 'image-name.jpg'   // Media: use name property
  }
})
```

### Update Object

```typescript
await cosmic.objects.updateOne('object-id', {
  title: 'Updated Title',
  metadata: {
    featured: true,
    categories: ['cat1-id', 'cat2-id'] // Multiple objects: array of ids
  }
})
```

### Delete Object

```typescript
await cosmic.objects.deleteOne('object-id')
```

### Batch Operations

Create, update, and delete multiple Objects in a single call (max 25 operations). Each operation succeeds or fails independently.

```typescript
// Using the SDK
const result = await cosmic.objects.batch([
  { method: 'add', object: { title: 'Post 1', type: 'posts', metadata: { content: '...' } } },
  { method: 'add', object: { title: 'Post 2', type: 'posts', metadata: { content: '...' } } },
  { method: 'edit', object_id: 'OBJECT_ID', object: { title: 'Updated' } },
  { method: 'delete', object_id: 'OBJECT_ID_2' },
])
// result.operations: [{ method, status, object/message }, ...]
```

## Object Types

Define content structure with Object types.

```typescript
// List all Object types
const types = await cosmic.objectTypes.find()

// Get single Object type
const blogType = await cosmic.objectTypes.findOne('posts')

// Create Object type with Metafields
await cosmic.objectTypes.insertOne({
  title: 'Blog Posts',
  slug: 'posts',
  singular: 'Post',
  emoji: '📝',
  metafields: [
    { title: 'Content', key: 'content', type: 'markdown', required: true },
    { title: 'Image', key: 'image', type: 'file', media_validation_type: 'image' },
    { title: 'Author', key: 'author', type: 'object', object_type: 'authors' },
    { title: 'Tags', key: 'tags', type: 'objects', object_type: 'tags' }
  ]
})
```

## Metafield Types

| Type | Description | Value Format |
|------|-------------|--------------|
| `text` | Single line text | `string` |
| `textarea` | Multi-line text | `string` |
| `markdown` | Markdown editor | `string` |
| `html-textarea` | Rich HTML editor | `string` |
| `number` | Numeric value | `number` |
| `date` | Date picker | `"YYYY-MM-DD"` |
| `switch` | Boolean toggle | `true/false` |
| `select` | Single selection (preferred) | `string` |
| `multi-select` | Multiple selection | `string[]` |
| `select-dropdown` | Dropdown selection (deprecated, use `select`) | `{key: string, value: string}` |
| `radio-buttons` | Radio selection | `string` |
| `check-boxes` | Multiple selection | `string[]` |
| `file` | Single media | Media `name` |
| `files` | Multiple media | Media `name[]` |
| `object` | Single relation | Object `id` |
| `objects` | Multiple relations | Object `id[]` |
| `json` | JSON data | `object` |
| `color` | Color picker | `"#hex"` |
| `repeater` | Repeatable group | `array` |
| `parent` | Nested group | `object` |

## Metafield Validation

Metafields support validation properties to enforce data quality:

| Property | Type | Description |
|----------|------|-------------|
| `required` | `boolean` | A value must be provided |
| `unique` | `boolean` | Value must be unique across all Objects of the same type. Applies to top-level text, textarea, number, date, select Metafields (not supported inside Parent or Repeater groups) |
| `show_when` | `object` | Conditional visibility: `{ key, op, value }`. Show field when sibling field matches condition. Ops: `eq`, `neq`, `exists`, `not_exists`. Hidden fields skip required validation. Top-level only. |
| `regex` | `string` | Restrict value to match a regular expression |
| `regex_message` | `string` | Message shown when regex validation fails |
| `minlength` | `number` | Minimum character length (text, textarea) |
| `maxlength` | `number` | Maximum character length (text, textarea) |

```typescript
// Object type with validation
await cosmic.objectTypes.insertOne({
  title: 'Contacts',
  slug: 'contacts',
  metafields: [
    { title: 'Email', key: 'email', type: 'text', required: true, unique: true },
    { title: 'Name', key: 'name', type: 'text', required: true, minlength: 2 }
  ]
})
```

## Queries

Filter content using MongoDB-style queries:

```typescript
// Basic filter
await cosmic.objects.find({
  type: 'products',
  'metadata.category': 'electronics'
})

// Comparison operators
await cosmic.objects.find({
  type: 'products',
  'metadata.price': { $lt: 100 }    // Less than
  // $gt, $gte, $lte, $eq, $ne also available
})

// Array matching
await cosmic.objects.find({
  type: 'products',
  'metadata.tags': { $in: ['sale', 'featured'] }  // Any match
  // $all for all must match, $nin for none match
})

// Logical operators
await cosmic.objects.find({
  type: 'products',
  $and: [
    { 'metadata.price': { $lte: 50 } },
    { 'metadata.in_stock': true }
  ]
  // $or, $not, $nor also available
})

// Text search
await cosmic.objects.find({
  type: 'posts',
  title: { $regex: 'hello', $options: 'i' }  // Case insensitive
})
```

### Query Options

```typescript
await cosmic.objects.find({ type: 'posts' })
  .props(['title', 'slug', 'metadata'])
  .sort('-created_at')  // Descending by created date
  .limit(10)
  .skip(20)             // Pagination
  .depth(2)             // Resolve nested Object relationships
  .status('any')        // 'published' | 'draft' | 'any'
```

## Media

```typescript
// List media
const media = await cosmic.media
  .find({ folder: 'images' })
  .props(['url', 'imgix_url', 'alt_text'])
  .limit(20)

// Upload media
const uploaded = await cosmic.media.insertOne({
  media: { originalname: 'photo.jpg', buffer: fileBuffer },
  folder: 'uploads',
  alt_text: 'Description of image',
  metadata: { caption: 'Photo caption' }
})

// Update media
await cosmic.media.updateOne('media-id', {
  alt_text: 'Updated alt text',
  folder: 'new-folder'
})

// Delete media
await cosmic.media.deleteOne('media-id')
```

### imgix Image Processing

All images have `imgix_url` for transformations:

```typescript
const optimized = `${media.imgix_url}?w=800&auto=format,compress`
const thumbnail = `${media.imgix_url}?w=100&h=100&fit=crop`
```

## AI Generation

Cosmic provides built-in AI capabilities for text, images, and video.

### Generate Text

```typescript
// Simple prompt
const text = await cosmic.ai.generateText({
  prompt: 'Write a product description for a coffee mug',
  model: 'claude-sonnet-4-5-20250929', // Optional
  max_tokens: 500
})
console.log(text.text)

// Chat messages
const chat = await cosmic.ai.generateText({
  messages: [
    { role: 'user', content: 'Tell me about coffee' },
    { role: 'assistant', content: 'Coffee is a beverage...' },
    { role: 'user', content: 'What about espresso?' }
  ]
})

// Analyze image or document
const analysis = await cosmic.ai.generateText({
  prompt: 'Describe this image',
  media_url: 'https://cdn.cosmicjs.com/image.jpg'
})

// Streaming
const stream = await cosmic.ai.stream({
  prompt: 'Write a blog post',
  max_tokens: 1000
})
for await (const chunk of stream) {
  process.stdout.write(chunk.text || '')
}
```

### Generate Image

```typescript
const image = await cosmic.ai.generateImage({
  prompt: 'Mountain landscape at sunset',
  model: 'gemini-3-pro-image-preview', // Default, supports up to 4K
  size: '1024x1024',
  folder: 'ai-generated',
  alt_text: 'AI mountain landscape'
})
console.log(image.media.url)

// With reference images (Gemini only)
const styled = await cosmic.ai.generateImage({
  prompt: 'Same style but with ocean',
  reference_images: ['https://cdn.cosmicjs.com/style-ref.jpg']
})
```

### Generate Video

```typescript
const video = await cosmic.ai.generateVideo({
  prompt: 'A kitten playing with yarn in sunlight',
  model: 'veo-3.1-fast-generate-preview', // Fast: 30-90s generation
  duration: 8,        // 4, 6, or 8 seconds
  resolution: '720p', // or '1080p'
  folder: 'videos'
})
console.log(video.media.url)

// With reference image as first frame
const productVideo = await cosmic.ai.generateVideo({
  prompt: 'Product rotates smoothly',
  reference_images: ['https://cdn.cosmicjs.com/product.jpg'],
  duration: 6
})

// Extend existing video
const extended = await cosmic.ai.extendVideo({
  media_id: video.media.id,
  prompt: 'The kitten walks away into the garden'
})
```

## Available AI Models

**Text:** `claude-sonnet-4-5-20250929` (recommended), `claude-opus-4-5-20251101`, `gpt-5`, `gemini-3-pro-preview`

**Image:** `gemini-3-pro-image-preview` (default, up to 4K), `dall-e-3`

**Video:** `veo-3.1-fast-generate-preview` (recommended), `veo-3.1-generate-preview` (premium)

## Framework Patterns

### Next.js App Router

```typescript
// app/posts/page.tsx
import { createBucketClient } from '@cosmicjs/sdk'

const cosmic = createBucketClient({
  bucketSlug: process.env.COSMIC_BUCKET_SLUG!,
  readKey: process.env.COSMIC_READ_KEY!
})

export default async function Posts() {
  const { objects: posts } = await cosmic.objects
    .find({ type: 'posts' })
    .props(['title', 'slug', 'metadata.excerpt'])
    .limit(10)

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### Server Actions

```typescript
'use server'

export async function createPost(formData: FormData) {
  const cosmic = createBucketClient({
    bucketSlug: process.env.COSMIC_BUCKET_SLUG!,
    readKey: process.env.COSMIC_READ_KEY!,
    writeKey: process.env.COSMIC_WRITE_KEY!
  })

  await cosmic.objects.insertOne({
    title: formData.get('title') as string,
    type: 'posts',
    metadata: {
      content: formData.get('content')
    }
  })
}
```

## Common Patterns

### Pagination

```typescript
const page = 1
const limit = 10

const { objects, total } = await cosmic.objects
  .find({ type: 'posts' })
  .skip((page - 1) * limit)
  .limit(limit)

const totalPages = Math.ceil(total / limit)
```

### Draft Preview

```typescript
const post = await cosmic.objects
  .findOne({ type: 'posts', slug })
  .status('any')  // Include drafts
```

### Localized Content

```typescript
const post = await cosmic.objects
  .findOne({ type: 'posts', slug, locale: 'es' })
```

## Select and Multi-Select Values

For new content models, prefer `select` (single selection) and `multi-select` (multiple selection) over the legacy `select-dropdown` type.

- **`select`** returns a plain `string` from the API. No helper needed.
- **`multi-select`** returns a `string[]` from the API. No helper needed.
- **`select-dropdown`** (deprecated) returns `{key: string, value: string}` objects, which can cause "Objects are not valid as a React child" errors in JSX.

```tsx
// select and multi-select: use values directly
<span>{product.metadata?.status}</span>
{product.metadata?.tags?.map(tag => <span key={tag}>{tag}</span>)}
```

If your project still uses legacy `select-dropdown` fields, include this helper in your project (e.g., `lib/cosmic.ts`):

```typescript
export function getMetafieldValue(field: unknown): string {
  if (field === null || field === undefined) return '';
  if (typeof field === 'string') return field;
  if (typeof field === 'number' || typeof field === 'boolean') return String(field);
  if (typeof field === 'object' && field !== null && 'value' in field) {
    return String((field as { value: unknown }).value);
  }
  if (typeof field === 'object' && field !== null && 'key' in field) {
    return String((field as { key: unknown }).key);
  }
  return '';
}
```

The function is safe for all types (strings, numbers, booleans pass through unchanged), so wrap legacy `select-dropdown` metadata values rendered in JSX:

```tsx
// Wrong: may crash if field is select-dropdown
<span>{product.metadata?.category}</span>

// Correct: always safe for legacy select-dropdown
<span>{getMetafieldValue(product.metadata?.category)}</span>
```

## Key Reminders

1. **No credentials? Use agent signup** - If the user has no `COSMIC_*` env vars and no account, call `POST /v3/agents/sign-up` (see "No Cosmic account?" section above). Never ask the user to manually go sign up.
2. **Object type is the SLUG** - Use `type: 'blog-posts'`, not `type: 'Blog Posts'`
3. **Media uses `name`** - Reference media by `name` property, not URL
4. **Relations use `id`** - Reference related Objects by `id`, not slug
5. **Never expose `writeKey`** - Keep it server-side only
6. **Use `props()`** - Always specify needed properties for performance
7. **imgix for images** - Use `imgix_url` with query params for optimizations
8. **Use `select` over `select-dropdown`** - New content models should use `select` (returns plain strings) instead of `select-dropdown` (returns objects)
9. **Wrap legacy metadata in JSX** - Use `getMetafieldValue()` for `select-dropdown` metadata values rendered in JSX

## Resources

- [Documentation](https://www.cosmicjs.com/docs)
- [API Reference](https://www.cosmicjs.com/docs/api)
- [Agent signup flow](https://www.cosmicjs.com/docs/agent-skills#agent-signup) - end-to-end onboarding with no prior Cosmic account
- [Agents API reference](https://www.cosmicjs.com/docs/api/agents) - `/v3/agents/sign-up`, `/verify`, `/status` details and error codes
- [SDK on npm](https://www.npmjs.com/package/@cosmicjs/sdk)
- [Discord Community](https://discord.gg/cosmicjs)
