# Architecture Overview

**TL;DR**: Static-first Next.js app that periodically scrapes GitHub for Claude Code marketplaces, validates and indexes them, stores data in JSON files (with Vercel Blob backup), and generates static pages for browsing.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Repositories                       │
│         (Contains .claude-plugin/marketplace.json)           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ GitHub API (Search + Content)
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                   Search Script                              │
│              (scripts/search.ts)                             │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐          │
│  │  Discover  │→ │  Validate  │→ │   Extract    │          │
│  │Marketplaces│  │   Schema   │  │   Plugins    │          │
│  └────────────┘  └────────────┘  └──────────────┘          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Write JSON
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                  Data Storage Layer                          │
│                                                              │
│  ┌──────────────────────┐    ┌──────────────────────┐      │
│  │  Vercel Blob (Prod)  │    │ Local Files (Dev)    │      │
│  │  - marketplaces.json │    │ - marketplaces.json  │      │
│  │  - plugins.json      │    │ - plugins.json       │      │
│  └──────────────────────┘    └──────────────────────┘      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Read data at build time
                     ↓
┌─────────────────────────────────────────────────────────────┐
│              Next.js Build Process                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │Generate Home │  │Generate 400+ │  │  Optimize    │     │
│  │    Page      │  │Plugin Pages  │  │   Assets     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Deploy
                     ↓
┌─────────────────────────────────────────────────────────────┐
│              Static Website (Vercel)                         │
│                                                              │
│  User → Browse → Filter → View Plugin Details               │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

### Core Framework

- **Next.js 15** with App Router
  - File-based routing
  - React Server Components by default
  - Static site generation (SSG)
  - Turbopack for fast builds

### Frontend

- **React 19** - Latest React with concurrent features
- **TypeScript 5** - Strict typing throughout
- **Tailwind CSS v4** - Utility-first styling with custom theme
- **shadcn/ui** - Accessible component library (Radix UI primitives)

### Backend/Build

- **Bun** - Fast package manager and runtime
- **Zod** - Schema validation for marketplace files
- **Octokit** - GitHub API client
- **Vercel Blob** - Production data storage

### Data Flow

- **Static Generation**: All pages pre-rendered at build time
- **No Runtime Database**: Data read from JSON files
- **Periodic Updates**: Search script run manually or via cron

## Key Design Decisions

### 1. Static Site Generation (SSG)

**Why**:

- Fast page loads (pre-rendered HTML)
- No database queries at runtime
- Excellent SEO
- Low hosting costs

**How**:

- `generateStaticParams()` generates paths for all marketplaces
- Data fetched at build time, not request time
- Pages cached on CDN

### 2. JSON File Storage

**Why**:

- Simple to version control
- No database setup required
- Easy to inspect and debug
- Fast read access at build time

**Trade-offs**:

- Not suitable for frequently changing data
- Requires rebuild to update content
- File size can grow (current: ~35k lines for plugins.json)

### 3. Dual Storage Strategy

**Development**:

```typescript
// Read from lib/data/*.json
const data = await fs.readFile(MARKETPLACES_FILE, "utf-8");
```

**Production**:

```typescript
// Try Vercel Blob first, fallback to local
const { blobs } = await list({ prefix: "marketplaces.json" });
const response = await fetch(blobUrl);
```

**Why**: Vercel has a read-only filesystem, but we want persistent storage for the search script output.

### 4. URL-based Filtering

Filters stored in URL query parameters:

```
/plugins/anthropics-claude-code?q=security&categories=productivity
```

**Benefits**:

- Shareable URLs
- Browser back/forward works
- No client state management needed

### 5. Slug-based Routing

Marketplaces identified by slug (e.g., `anthropics-claude-code`):

```typescript
// Conversion function
function repoToSlug(repo: string): string {
  return repo.replace(/\//g, "-").toLowerCase();
}

// "anthropics/claude-code" → "anthropics-claude-code"
```

**Why**: URL-safe, predictable, unique

## Data Model

### Marketplace Schema

```typescript
interface Marketplace {
  repo: string; // "owner/repo-name"
  slug: string; // "owner-repo-name" (URL-safe)
  description: string; // Marketplace description
  pluginCount: number; // Number of plugins
  categories: string[]; // Plugin categories found
  discoveredAt?: string; // ISO timestamp
  lastUpdated?: string; // ISO timestamp
  source?: "manual" | "auto";
  stars?: number; // GitHub star count
  starsFetchedAt?: string; // ISO timestamp
}
```

### Plugin Schema

```typescript
interface Plugin {
  id: string; // "marketplace-slug/plugin-name"
  name: string; // Plugin name
  description: string; // Plugin description
  version?: string;
  author?: {
    name: string;
    email?: string;
    url?: string;
  };
  homepage?: string;
  repository?: string;
  source: string; // Path to plugin directory
  marketplace: string; // Marketplace slug
  marketplaceUrl: string; // Full GitHub URL
  category: string; // Primary category
  license?: string;
  keywords?: string[];
  commands?: string[]; // Command files
  agents?: string[]; // Agent files
  hooks?: string[]; // Hook files
  mcpServers?: string[]; // MCP server configs
  installCommand: string; // "/plugin install name@marketplace"
}
```

## Validation Layer

### Schema Validation (Zod)

The system validates marketplace.json files against a strict schema:

```typescript
const ClaudeMarketplaceFileSchema = z.object({
  name: z.string(),
  version: z.string().optional(),
  description: z.string().optional(),
  owner: z
    .object({
      name: z.string(),
      email: z.string().email().optional(),
    })
    .optional(),
  metadata: z
    .object({
      description: z.string().optional(),
      // ...
    })
    .optional(),
  plugins: z.array(PluginSchema),
});
```

### Multi-step Validation Process

1. **Parse JSON** - Ensure valid JSON syntax
2. **Schema Validation** - Zod schema check
3. **Repository Access** - Verify repo is public
4. **Plugin Health Check** - Verify required fields
5. **Data Extraction** - Extract categories, descriptions

Failures at any step mark the marketplace as invalid.

## Search and Discovery

### GitHub Search Query

```typescript
const query = "filename:marketplace.json path:.claude-plugin";
```

Finds all repositories with `.claude-plugin/marketplace.json`

### Rate Limiting

- GitHub API: 5000 requests/hour (authenticated)
- Search API: 30 requests/minute
- Handled with exponential backoff and retry logic

### Pagination

- Fetches up to 1000 results (10 pages × 100 per page)
- Currently finds ~500 marketplaces

## Performance Characteristics

### Build Time

- ~442 valid marketplaces
- ~3000+ plugins indexed
- Build completes in ~2-3 minutes
- Static pages generated for all routes

### Runtime Performance

- No database queries
- Static HTML served from CDN
- Client-side filtering (instant)
- Lazy loading for large lists

### Data Size

- marketplaces.json: ~2MB (442 marketplaces)
- plugins.json: ~5MB (~3000 plugins)
- Loaded once at build time, not sent to client

## Deployment

### Vercel Configuration

```typescript
// next.config.mjs
export default {
  experimental: {
    turbo: {
      // Turbopack optimization
    },
  },
};
```

### Environment Variables

```bash
# GitHub API (for search script)
GITHUB_TOKEN=ghp_...

# Vercel Blob (for data storage)
BLOB_READ_WRITE_TOKEN=vercel_blob_...
```

### Build Process

1. Install dependencies: `bun install`
2. Run search script: `bun run scripts/search.ts`
3. Build site: `bun run build`
4. Deploy: Automatic via Vercel GitHub integration

## Caching Strategy

### Build-time Caching

- Static pages cached indefinitely
- Revalidate on new deployment

### Client-side Caching

- URL parameters preserve filter state
- Browser caches static assets

### Data Freshness

- Manual: Run `bun run scripts/search.ts`
- Automated: Vercel cron job (if configured)

## Scalability Considerations

### Current Limits

- ~500 marketplaces discoverable
- ~5000 plugins indexable
- Build time scales linearly with marketplace count

### Potential Bottlenecks

1. **GitHub API Rate Limits**: 5000 req/hour
2. **Build Time**: Grows with marketplace count
3. **File Size**: plugins.json approaching manageable limits

### Future Optimizations

- Incremental static regeneration (ISR)
- Database for runtime queries
- Pagination for marketplace list
- Search index (Algolia, etc.)
