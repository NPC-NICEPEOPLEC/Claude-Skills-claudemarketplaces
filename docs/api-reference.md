# API Reference

**TL;DR**: This document lists key functions, their signatures, and purposes. For usage examples, see the [Development Guide](./development-guide.md).

## Data Access Functions

### lib/data/marketplaces.ts

#### `getAllMarketplaces(options?)`

Fetches all marketplaces from storage.

**Signature**:
```typescript
async function getAllMarketplaces(options?: {
  includeEmpty?: boolean;
}): Promise<Marketplace[]>
```

**Parameters**:
- `options.includeEmpty` (optional): If `false`, filters out marketplaces with `pluginCount === 0`. Default: `true`

**Returns**: Array of Marketplace objects with computed slugs

**Example**:
```typescript
// Get all marketplaces
const all = await getAllMarketplaces();

// Get only marketplaces with plugins
const nonEmpty = await getAllMarketplaces({ includeEmpty: false });
```

**Implementation**:
- Reads from `lib/data/marketplaces.json` (or Vercel Blob)
- Adds `slug` field to each marketplace
- Optionally filters by plugin count

---

#### `getMarketplaceBySlug(slug)`

Finds a single marketplace by its URL slug.

**Signature**:
```typescript
async function getMarketplaceBySlug(slug: string): Promise<Marketplace | null>
```

**Parameters**:
- `slug`: URL-safe marketplace identifier (e.g., `"anthropics-claude-code"`)

**Returns**: Marketplace object or `null` if not found

**Example**:
```typescript
const marketplace = await getMarketplaceBySlug('anthropics-claude-code');

if (!marketplace) {
  notFound(); // Next.js 404
}
```

**Usage**: Primary function for marketplace detail pages

---

#### `getMarketplacesByCategory(category)`

Filters marketplaces by category.

**Signature**:
```typescript
async function getMarketplacesByCategory(category: string): Promise<Marketplace[]>
```

**Parameters**:
- `category`: Category name (e.g., `"productivity"`, `"development"`)

**Returns**: Array of marketplaces containing that category

**Example**:
```typescript
const productivityMarketplaces = await getMarketplacesByCategory('productivity');
```

---

#### `getCategories()`

Gets all unique categories across all marketplaces.

**Signature**:
```typescript
async function getCategories(): Promise<string[]>
```

**Returns**: Sorted array of category names

**Example**:
```typescript
const categories = await getCategories();
// ["community", "development", "integration", "productivity"]
```

---

### lib/data/plugins.ts

#### `getAllPlugins()`

Fetches all plugins across all marketplaces.

**Signature**:
```typescript
async function getAllPlugins(): Promise<Plugin[]>
```

**Returns**: Array of all Plugin objects

**Example**:
```typescript
const plugins = await getAllPlugins();
console.log(`Total plugins: ${plugins.length}`);
```

**Implementation**: Reads from `lib/data/plugins.json` (or Vercel Blob)

---

#### `getPluginsByMarketplace(slug)`

Gets plugins for a specific marketplace.

**Signature**:
```typescript
async function getPluginsByMarketplace(slug: string): Promise<Plugin[]>
```

**Parameters**:
- `slug`: Marketplace slug (e.g., `"anthropics-claude-code"`)

**Returns**: Array of plugins from that marketplace

**Example**:
```typescript
const plugins = await getPluginsByMarketplace('anthropics-claude-code');
```

**Usage**: Primary function for plugin list pages

---

#### `getPluginCategories(marketplaceSlug)`

Gets unique categories for plugins in a marketplace.

**Signature**:
```typescript
async function getPluginCategories(marketplaceSlug: string): Promise<string[]>
```

**Parameters**:
- `marketplaceSlug`: Marketplace slug

**Returns**: Sorted array of category names found in that marketplace's plugins

**Example**:
```typescript
const categories = await getPluginCategories('anthropics-claude-code');
// ["development", "productivity"]
```

**Usage**: For category filter badges on plugin pages

---

## Search and Discovery

### lib/search/github-search.ts

#### `searchMarketplaceFiles(verbose?)`

Searches GitHub for repositories with `.claude-plugin/marketplace.json`.

**Signature**:
```typescript
async function searchMarketplaceFiles(verbose?: boolean): Promise<GitHubSearchResult[]>
```

**Parameters**:
- `verbose` (optional): Enable detailed logging. Default: `false`

**Returns**: Array of search results with repo and path info

**Example**:
```typescript
const results = await searchMarketplaceFiles(true);
console.log(`Found ${results.length} marketplaces`);
```

**Implementation**:
- Uses GitHub Code Search API
- Query: `filename:marketplace.json path:.claude-plugin`
- Fetches up to 1000 results (10 pages × 100)

---

#### `fetchMarketplaceFile(repo, branch?, verbose?)`

Fetches the raw content of a marketplace.json file.

**Signature**:
```typescript
async function fetchMarketplaceFile(
  repo: string,
  branch?: string,
  verbose?: boolean
): Promise<string>
```

**Parameters**:
- `repo`: Full repository name (e.g., `"anthropics/claude-code"`)
- `branch` (optional): Branch name. Default: `"main"` (falls back to `"master"`)
- `verbose` (optional): Enable logging

**Returns**: JSON string content of marketplace.json

**Throws**: Error if file not accessible

**Example**:
```typescript
try {
  const content = await fetchMarketplaceFile('anthropics/claude-code', 'main');
  const parsed = JSON.parse(content);
} catch (error) {
  console.error('Failed to fetch marketplace file');
}
```

---

#### `isRepoAccessible(repo, verbose?)`

Checks if a GitHub repository is publicly accessible.

**Signature**:
```typescript
async function isRepoAccessible(
  repo: string,
  verbose?: boolean
): Promise<boolean>
```

**Parameters**:
- `repo`: Full repository name
- `verbose` (optional): Enable logging

**Returns**: `true` if repo is accessible, `false` otherwise

**Example**:
```typescript
if (await isRepoAccessible('anthropics/claude-code')) {
  // Proceed with fetching
}
```

---

#### `getRepoDescription(repo, verbose?)`

Fetches repository description from GitHub.

**Signature**:
```typescript
async function getRepoDescription(
  repo: string,
  verbose?: boolean
): Promise<string>
```

**Parameters**:
- `repo`: Full repository name
- `verbose` (optional): Enable logging

**Returns**: Repository description or empty string

**Example**:
```typescript
const description = await getRepoDescription('anthropics/claude-code');
```

---

### lib/search/validator.ts

#### `validateMarketplace(repo, jsonContent, verbose?)`

Validates a marketplace.json file and converts to Marketplace format.

**Signature**:
```typescript
async function validateMarketplace(
  repo: string,
  jsonContent: string,
  verbose?: boolean
): Promise<ValidationResult>
```

**Parameters**:
- `repo`: Repository name
- `jsonContent`: Raw JSON string from marketplace.json
- `verbose` (optional): Enable logging

**Returns**: ValidationResult object:
```typescript
interface ValidationResult {
  valid: boolean;
  marketplace?: Marketplace;
  errors: string[];
}
```

**Example**:
```typescript
const result = await validateMarketplace(
  'anthropics/claude-code',
  jsonContent
);

if (result.valid) {
  console.log('Valid marketplace:', result.marketplace);
} else {
  console.error('Validation errors:', result.errors);
}
```

**Validation Steps**:
1. Parse JSON
2. Validate against Zod schema
3. Check repo accessibility
4. Verify plugin required fields

---

#### `validateMarketplaces(marketplaceFiles, verbose?)`

Validates multiple marketplaces in parallel.

**Signature**:
```typescript
async function validateMarketplaces(
  marketplaceFiles: Array<{ repo: string; content: string }>,
  verbose?: boolean
): Promise<ValidationResult[]>
```

**Parameters**:
- `marketplaceFiles`: Array of repo/content pairs
- `verbose` (optional): Enable logging

**Returns**: Array of ValidationResult objects

**Example**:
```typescript
const results = await validateMarketplaces([
  { repo: 'owner/repo1', content: '{ ... }' },
  { repo: 'owner/repo2', content: '{ ... }' },
]);

const valid = results.filter(r => r.valid);
const invalid = results.filter(r => !r.valid);
```

---

### lib/search/plugin-extractor.ts

#### `extractPluginsFromMarketplace(marketplace, jsonContent)`

Extracts plugins from a validated marketplace.

**Signature**:
```typescript
function extractPluginsFromMarketplace(
  marketplace: Marketplace,
  jsonContent: string
): Plugin[]
```

**Parameters**:
- `marketplace`: Validated Marketplace object
- `jsonContent`: Raw JSON string from marketplace.json

**Returns**: Array of Plugin objects

**Example**:
```typescript
const plugins = extractPluginsFromMarketplace(marketplace, jsonContent);
console.log(`Extracted ${plugins.length} plugins`);
```

**Implementation**:
- Parses JSON
- Transforms each plugin to Plugin interface
- Generates unique plugin IDs
- Creates install commands

---

#### `extractPluginsFromMarketplaces(marketplaces, marketplaceFiles)`

Extracts plugins from multiple marketplaces.

**Signature**:
```typescript
function extractPluginsFromMarketplaces(
  marketplaces: Marketplace[],
  marketplaceFiles: Array<{ repo: string; content: string }>
): Plugin[]
```

**Parameters**:
- `marketplaces`: Array of validated Marketplace objects
- `marketplaceFiles`: Array of repo/content pairs

**Returns**: Combined array of all plugins

**Example**:
```typescript
const allPlugins = extractPluginsFromMarketplaces(
  validMarketplaces,
  fetchedFiles
);
```

---

## Storage Functions

### lib/search/storage.ts

#### `readMarketplaces()`

Reads marketplaces from storage.

**Signature**:
```typescript
async function readMarketplaces(): Promise<Marketplace[]>
```

**Returns**: Array of marketplaces

**Implementation**:
- Tries Vercel Blob first (if `BLOB_READ_WRITE_TOKEN` set)
- Falls back to `lib/data/marketplaces.json`
- Logs source used

---

#### `writeMarketplaces(marketplaces)`

Writes marketplaces to storage.

**Signature**:
```typescript
async function writeMarketplaces(marketplaces: Marketplace[]): Promise<void>
```

**Parameters**:
- `marketplaces`: Array of marketplaces to save

**Throws**: Error on write failure

**Implementation**:
- Writes to Vercel Blob (production)
- Writes to local file (development)
- JSON formatted with 2-space indent

---

#### `mergeMarketplaces(discovered, allDiscoveredRepos)`

Merges newly discovered marketplaces with existing data.

**Signature**:
```typescript
async function mergeMarketplaces(
  discovered: Marketplace[],
  allDiscoveredRepos: Set<string>
): Promise<{
  added: number;
  updated: number;
  removed: number;
  total: number;
}>
```

**Parameters**:
- `discovered`: Newly validated marketplaces
- `allDiscoveredRepos`: Set of all repo names found in search

**Returns**: Statistics about merge operation

**Example**:
```typescript
const result = await mergeMarketplaces(newMarketplaces, allRepos);
console.log(`Added: ${result.added}, Updated: ${result.updated}`);
```

**Logic**:
- Loads existing marketplaces
- Updates existing entries with new data
- Adds new marketplaces
- Removes marketplaces that were discovered but failed validation
- Preserves manual entries

---

#### `readPlugins()`

Reads plugins from storage.

**Signature**:
```typescript
async function readPlugins(): Promise<Plugin[]>
```

**Returns**: Array of plugins

**Implementation**: Similar to `readMarketplaces()`

---

#### `writePlugins(plugins)`

Writes plugins to storage.

**Signature**:
```typescript
async function writePlugins(plugins: Plugin[]): Promise<void>
```

**Parameters**:
- `plugins`: Array of plugins to save

**Throws**: Error on write failure

**Implementation**: Similar to `writeMarketplaces()`

---

## Utility Functions

### lib/utils/slug.ts

#### `repoToSlug(repo)`

Converts GitHub repo path to URL-safe slug.

**Signature**:
```typescript
function repoToSlug(repo: string): string
```

**Parameters**:
- `repo`: Repository name (e.g., `"anthropics/claude-code"`)

**Returns**: URL-safe slug (e.g., `"anthropics-claude-code"`)

**Example**:
```typescript
const slug = repoToSlug('anthropics/claude-code');
// "anthropics-claude-code"

const slug2 = repoToSlug('Owner/Repo-Name');
// "owner-repo-name"
```

**Implementation**: Replace `/` with `-`, convert to lowercase

---

### lib/utils/cn.ts

#### `cn(...inputs)`

Merges className strings with Tailwind conflict resolution.

**Signature**:
```typescript
function cn(...inputs: ClassValue[]): string
```

**Parameters**:
- `...inputs`: Any number of className strings, objects, or arrays

**Returns**: Merged className string

**Example**:
```typescript
cn('px-2 py-1', 'px-3'); // "py-1 px-3" (px-3 wins)
cn('text-red-500', isActive && 'text-blue-500'); // Conditional class
cn('base-class', { 'active': isActive }, className); // Object syntax
```

**Implementation**: Uses `clsx` + `tailwind-merge`

---

## React Hooks

### lib/hooks/use-marketplace-filters.ts

#### `useMarketplaceFilters(marketplaces)`

Manages marketplace filtering state.

**Signature**:
```typescript
function useMarketplaceFilters(marketplaces: Marketplace[]): {
  searchQuery: string;
  selectedCategories: string[];
  filteredMarketplaces: Marketplace[];
  filteredCount: number;
  setSearchQuery: (query: string) => void;
  toggleCategory: (category: string) => void;
  clearFilters: () => void;
}
```

**Parameters**:
- `marketplaces`: Array of marketplaces to filter

**Returns**: Filter state and control functions

**Example**:
```typescript
const {
  filteredMarketplaces,
  setSearchQuery,
  toggleCategory,
} = useMarketplaceFilters(marketplaces);

<input onChange={e => setSearchQuery(e.target.value)} />
<Badge onClick={() => toggleCategory('productivity')} />
```

**Features**:
- URL-based state (shareable links)
- Search by name, description, category
- Multi-category filtering
- Sorted by stars (descending)

---

### lib/hooks/use-plugin-filters.ts

#### `usePluginFilters(plugins)`

Manages plugin filtering state (similar to marketplace filters).

**Signature**:
```typescript
function usePluginFilters(plugins: Plugin[]): {
  searchQuery: string;
  selectedCategories: string[];
  filteredPlugins: Plugin[];
  filteredCount: number;
  setSearchQuery: (query: string) => void;
  toggleCategory: (category: string) => void;
  clearFilters: () => void;
}
```

**Parameters**:
- `plugins`: Array of plugins to filter

**Returns**: Filter state and control functions

**Example**:
```typescript
const {
  filteredPlugins,
  setSearchQuery,
  toggleCategory,
} = usePluginFilters(plugins);
```

**Features**: Same as marketplace filters, but for plugins

---

## Type Definitions

### lib/types.ts

#### `Marketplace`

```typescript
interface Marketplace {
  repo: string;              // "owner/repo-name"
  slug: string;              // "owner-repo-name" (computed)
  description: string;       // Marketplace description
  pluginCount: number;       // Number of plugins
  categories: string[];      // Unique categories from plugins
  discoveredAt?: string;     // ISO timestamp
  lastUpdated?: string;      // ISO timestamp
  source?: 'manual' | 'auto'; // Discovery method
  stars?: number;            // GitHub star count
  starsFetchedAt?: string;   // When stars were fetched
}
```

---

#### `Plugin`

```typescript
interface Plugin {
  id: string;                // "marketplace-slug/plugin-name"
  name: string;              // Plugin name
  description: string;       // Plugin description
  version?: string;          // Semantic version
  author?: {
    name: string;
    email?: string;
    url?: string;
  };
  homepage?: string;         // Plugin homepage URL
  repository?: string;       // Plugin repository URL
  source: string;            // Path to plugin directory
  marketplace: string;       // Marketplace slug
  marketplaceUrl: string;    // Full marketplace GitHub URL
  category: string;          // Primary category
  license?: string;          // License identifier (e.g., "MIT")
  keywords?: string[];       // Search keywords
  commands?: string[];       // Slash command files
  agents?: string[];         // Agent files
  hooks?: string[];          // Hook files
  mcpServers?: string[];     // MCP server configs
  installCommand: string;    // "/plugin install name@marketplace"
}
```

---

## Environment Variables

### `GITHUB_TOKEN`

**Purpose**: GitHub API authentication

**Required for**: Running search script with higher rate limits

**Format**: `ghp_...` (GitHub personal access token)

**Permissions needed**:
- `public_repo` (read public repositories)
- `read:user` (optional, for rate limit)

**Usage**:
```bash
export GITHUB_TOKEN=ghp_your_token_here
bun run scripts/search.ts
```

---

### `BLOB_READ_WRITE_TOKEN`

**Purpose**: Vercel Blob storage access

**Required for**: Production data persistence

**Format**: `vercel_blob_...`

**Usage**: Automatically used by storage functions if set

---

## Command-Line Scripts

### `bun run scripts/search.ts`

**Purpose**: Discover, validate, and index marketplaces

**Options**:
- `--limit N`: Limit to first N repositories
- `--dry-run`: Run without saving results
- `--verbose` or `-v`: Enable detailed logging
- `--help` or `-h`: Show help message

**Example**:
```bash
# Full search
bun run scripts/search.ts

# Test with 10 repos
bun run scripts/search.ts --limit 10 --verbose

# Dry run (no writes)
bun run scripts/search.ts --dry-run
```

**Output**:
- Discovered marketplaces count
- Validation results
- Plugin extraction stats
- Merge statistics (added/updated/removed)

---

## Next.js API Conventions

### `generateStaticParams()`

**Purpose**: Generate static paths for dynamic routes

**Returns**: Array of param objects

**Example**:
```typescript
export async function generateStaticParams() {
  const marketplaces = await getAllMarketplaces();
  return marketplaces.map(m => ({ slug: m.slug }));
}
```

---

### `generateMetadata(params)`

**Purpose**: Generate page metadata for SEO

**Returns**: Metadata object

**Example**:
```typescript
export async function generateMetadata({ params }): Promise<Metadata> {
  const marketplace = await getMarketplaceBySlug(params.slug);

  return {
    title: `${marketplace.repo} Plugins`,
    description: marketplace.description,
    openGraph: { /* ... */ },
  };
}
```

---

## Common Patterns

### Server Component Data Fetching

```typescript
export default async function Page() {
  const data = await fetchData(); // At build time (SSG)
  return <ClientComponent data={data} />;
}
```

### Client Component with URL State

```typescript
"use client";

export function Component({ data }) {
  const searchParams = useSearchParams();
  const router = useRouter();

  const query = searchParams.get('q') || '';

  const updateQuery = (newQuery: string) => {
    const params = new URLSearchParams(searchParams);
    params.set('q', newQuery);
    router.replace(`${pathname}?${params}`, { scroll: false });
  };

  // ...
}
```

### Error Handling

```typescript
try {
  const data = await fetchData();
  return data;
} catch (error) {
  console.error('Error:', error);
  return [];
}
```

---

## Related Documentation

- [Architecture](./architecture.md) - System design overview
- [Data Flow](./data-flow.md) - How data moves through the system
- [Development Guide](./development-guide.md) - How to add features
- [Troubleshooting](./troubleshooting.md) - Common issues and solutions
