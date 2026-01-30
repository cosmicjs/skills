# Cosmic Agent Skills

AI coding assistant skills for building applications with [Cosmic](https://www.cosmicjs.com) headless CMS.

## Installation

Install skills for your AI coding assistant using [skills](https://skills.sh):

```bash
npx skills add cosmicjs/skills
```

Works with: Cursor, Claude Code, GitHub Copilot, Windsurf, Gemini, and [16+ other agents](https://skills.sh).

## What This Skill Covers

- **SDK-first development** with `@cosmicjs/sdk`
- **Objects API** - Create, read, update, delete content
- **Object Types** - Content modeling with Metafields
- **Media API** - Upload and manage files/images
- **Queries** - MongoDB-style filtering and search
- **AI Generation** - Text, images, and video generation
- **Framework patterns** - Next.js, React, and more

## Quick Example

```typescript
import { createBucketClient } from '@cosmicjs/sdk'

const cosmic = createBucketClient({
  bucketSlug: 'your-bucket',
  readKey: 'your-read-key'
})

// Get blog posts
const { objects: posts } = await cosmic.objects
  .find({ type: 'posts' })
  .props(['title', 'slug', 'metadata'])
  .limit(10)
```

## Resources

- [Cosmic Documentation](https://www.cosmicjs.com/docs)
- [SDK on npm](https://www.npmjs.com/package/@cosmicjs/sdk)
- [Discord Community](https://discord.com/invite/MSCwQ7D6Mg)

## License

MIT
