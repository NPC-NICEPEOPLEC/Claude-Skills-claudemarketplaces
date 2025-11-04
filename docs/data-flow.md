# Data Flow

**TL;DR**: GitHub → Search → Validate → Extract → Store → Build → Deploy. Data flows one direction: from GitHub repositories through validation and extraction into JSON files, which are then read at build time to generate static pages.

## Complete Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                          STEP 1: DISCOVERY                        │
│                                                                    │
│  User runs: bun run scripts/search.ts                            │
│                                                                    │
│  ┌──────────────┐                                                │
│  │ GitHub API   │ Search for .claude-plugin/marketplace.json     │
│  │ Search       │ Query: "filename:marketplace.json              │
│  │              │         path:.claude-plugin"                    │
│  └──────┬───────┘                                                │
│         │                                                         │
│         ↓                                                         │
│  Returns ~500 repositories with marketplace files                │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                          STEP 2: FETCH                            │
│                                                                    │
│  For each repository:                                             │
│                                                                    │
│  ┌──────────────┐                                                │
│  │ GitHub API   │ GET /repos/:owner/:repo/contents/             │
│  │ Content      │     .claude-plugin/marketplace.json            │
│  │              │ Try: main branch → master branch              │
│  └──────┬───────┘                                                │
│         │                                                         │
│         ↓                                                         │
│  Base64-encoded file content                                      │
│         │                                                         │
│         ↓                                                         │
│  Decode to JSON string                                            │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                       STEP 3: VALIDATION                          │
│                                                                    │
│  ┌─────────────────────────────────────────────────┐            │
│  │ validateMarketplace(repo, jsonContent)          │            │
│  │                                                  │            │
│  │  1. Parse JSON                                   │            │
│  │     ├─ ✓ Valid → Continue                       │            │
│  │     └─ ✗ Invalid → REJECT                       │            │
│  │                                                  │            │
│  │  2. Validate against Zod schema                 │            │
│  │     ├─ ✓ Valid → Continue                       │            │
│  │     └─ ✗ Invalid → REJECT                       │            │
│  │                                                  │            │
│  │  3. Check repository accessibility               │            │
│  │     ├─ ✓ Public → Continue                      │            │
│  │     └─ ✗ Private/404 → REJECT                   │            │
│  │                                                  │            │
│  │  4. Validate plugins have required fields        │            │
│  │     ├─ ✓ All valid → ACCEPT                     │            │
│  │     └─ ✗ Missing fields → REJECT                │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                    │
│  Result: ValidationResult { valid: true/false, errors: [] }       │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                    STEP 4: PLUGIN EXTRACTION                      │
│                                                                    │
│  ┌─────────────────────────────────────────────────┐            │
│  │ extractPluginsFromMarketplace(marketplace, json) │            │
│  │                                                  │            │
│  │  For each plugin in marketplace.plugins:         │            │
│  │                                                  │            │
│  │  1. Generate unique ID                           │            │
│  │     marketplace-slug/plugin-name                 │            │
│  │                                                  │            │
│  │  2. Extract metadata                             │            │
│  │     - name, description, version                 │            │
│  │     - author, category, keywords                 │            │
│  │     - source path                                │            │
│  │                                                  │            │
│  │  3. Generate install command                     │            │
│  │     /plugin install <name>@<marketplace-slug>    │            │
│  │                                                  │            │
│  │  4. Create Plugin object                         │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                    │
│  Result: Plugin[] (array of plugin objects)                       │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                       STEP 5: ENRICHMENT                          │
│                                                                    │
│  Fetch GitHub star counts:                                        │
│                                                                    │
│  ┌──────────────┐                                                │
│  │ GitHub API   │ GET /repos/:owner/:repo                        │
│  │ Repository   │ Extract: stargazers_count                      │
│  └──────┬───────┘                                                │
│         │                                                         │
│         ↓                                                         │
│  Add stars to marketplace metadata                                │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                        STEP 6: STORAGE                            │
│                                                                    │
│  ┌────────────────────────────────────────┐                     │
│  │ mergeMarketplaces(discovered, existing) │                     │
│  │                                          │                     │
│  │  1. Load existing marketplaces.json      │                     │
│  │  2. For each discovered marketplace:     │                     │
│  │     ├─ Exists → UPDATE                   │                     │
│  │     └─ New → ADD                         │                     │
│  │  3. Remove invalid entries               │                     │
│  └────────────────────────────────────────┘                     │
│                    ↓                                              │
│  ┌────────────────────────────────────────┐                     │
│  │ writeMarketplaces(marketplaces)         │                     │
│  │                                          │                     │
│  │  If BLOB_TOKEN exists:                   │                     │
│  │    Write to Vercel Blob (production)     │                     │
│  │  Else:                                   │                     │
│  │    Write to lib/data/marketplaces.json   │                     │
│  └────────────────────────────────────────┘                     │
│                                                                    │
│  ┌────────────────────────────────────────┐                     │
│  │ writePlugins(plugins)                   │                     │
│  │                                          │                     │
│  │  If BLOB_TOKEN exists:                   │                     │
│  │    Write to Vercel Blob (production)     │                     │
│  │  Else:                                   │                     │
│  │    Write to lib/data/plugins.json        │                     │
│  └────────────────────────────────────────┘                     │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                       STEP 7: BUILD TIME                          │
│                                                                    │
│  User runs: bun run build                                         │
│                                                                    │
│  ┌────────────────────────────────────────┐                     │
│  │ Next.js Build Process                   │                     │
│  │                                          │                     │
│  │  1. Read marketplaces.json               │                     │
│  │     └─ getAllMarketplaces()              │                     │
│  │                                          │                     │
│  │  2. Generate static params               │                     │
│  │     └─ generateStaticParams()            │                     │
│  │        For each marketplace:             │                     │
│  │          Create /plugins/[slug] path     │                     │
│  │                                          │                     │
│  │  3. For each marketplace page:           │                     │
│  │     a. Read marketplace data             │                     │
│  │     b. Read plugins for marketplace      │                     │
│  │     c. Extract categories                │                     │
│  │     d. Generate HTML                     │                     │
│  │     e. Generate metadata (SEO)           │                     │
│  │                                          │                     │
│  │  4. Generate home page                   │                     │
│  │     a. Load all marketplaces             │                     │
│  │     b. Sort by stars                     │                     │
│  │     c. Render grid                       │                     │
│  └────────────────────────────────────────┘                     │
│                                                                    │
│  Output: Static HTML files for all routes                         │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                      STEP 8: RUNTIME (USER)                       │
│                                                                    │
│  User visits: claudemarketplaces.com/plugins/anthropics-claude-code│
│                                                                    │
│  1. CDN serves pre-rendered HTML                                  │
│  2. React hydrates on client                                      │
│  3. User interacts with filters                                   │
│     ├─ Search query → Filter plugins in memory                    │
│     ├─ Category filter → Filter plugins in memory                 │
│     └─ Update URL params                                          │
│  4. No API calls needed (all data in static HTML)                 │
└──────────────────────────────────────────────────────────────────┘
```

## Detailed Step Breakdowns

### Step 1: Discovery

**File**: `lib/search/github-search.ts`

```typescript
export async function searchMarketplaceFiles() {
  const query = "filename:marketplace.json path:.claude-plugin";

  // Pagination loop
  for (let page = 1; page <= maxPages; page++) {
    const response = await octokit.rest.search.code({
      q: query,
      per_page: 100,
      page: page,
    });

    results.push(...response.data.items);
  }

  return results.map(item => ({
    repo: item.repository.full_name, // "owner/repo"
    path: item.path,                  // ".claude-plugin/marketplace.json"
  }));
}
```

**Output**: Array of ~500 repository references

### Step 2: Fetch

**File**: `lib/search/github-search.ts`

```typescript
export async function fetchMarketplaceFile(repo: string) {
  const [owner, repoName] = repo.split('/');

  try {
    // Try main branch
    const response = await octokit.rest.repos.getContent({
      owner,
      repo: repoName,
      path: ".claude-plugin/marketplace.json",
      ref: "main",
    });

    // Decode base64 content
    if ('content' in response.data) {
      return Buffer.from(response.data.content, 'base64').toString('utf-8');
    }
  } catch {
    // Fallback to master branch
    // ... retry logic
  }
}
```

**Output**: JSON string content of marketplace file

### Step 3: Validation

**File**: `lib/search/validator.ts`

```typescript
export async function validateMarketplace(repo: string, jsonContent: string) {
  const errors: string[] = [];

  // Parse JSON
  let parsedData;
  try {
    parsedData = JSON.parse(jsonContent);
  } catch {
    return { valid: false, errors: ['Invalid JSON format'] };
  }

  // Validate schema
  const schemaValidation = ClaudeMarketplaceFileSchema.safeParse(parsedData);
  if (!schemaValidation.success) {
    return {
      valid: false,
      errors: schemaValidation.error.issues.map(e => e.message)
    };
  }

  // Check repo accessibility
  const accessible = await isRepoAccessible(repo);
  if (!accessible) {
    return { valid: false, errors: ['Repository not accessible'] };
  }

  // Create Marketplace object
  const marketplace: Marketplace = {
    repo,
    slug: repoToSlug(repo),
    description: parsedData.description || parsedData.metadata?.description,
    pluginCount: parsedData.plugins.length,
    categories: extractCategories(parsedData.plugins),
    discoveredAt: new Date().toISOString(),
    source: 'auto',
  };

  return { valid: true, marketplace, errors: [] };
}
```

**Output**: `{ valid: true/false, marketplace?, errors[] }`

### Step 4: Plugin Extraction

**File**: `lib/search/plugin-extractor.ts`

```typescript
export function extractPluginsFromMarketplace(
  marketplace: Marketplace,
  jsonContent: string
): Plugin[] {
  const parsedData = JSON.parse(jsonContent);
  const marketplaceSlug = repoToSlug(marketplace.repo);

  return parsedData.plugins.map(plugin => {
    const pluginId = `${marketplaceSlug}/${normalizePluginName(plugin.name)}`;

    return {
      id: pluginId,
      name: plugin.name,
      description: plugin.description || '',
      source: plugin.source,
      marketplace: marketplaceSlug,
      marketplaceUrl: `https://github.com/${marketplace.repo}`,
      category: plugin.category || 'community',
      installCommand: `/plugin install ${plugin.name}@${marketplaceSlug}`,
      // ... other fields
    };
  });
}
```

**Output**: Array of Plugin objects

### Step 5: Enrichment

**File**: `lib/search/github-stars.ts`

```typescript
export async function batchFetchStars(repos: string[]): Promise<Map<string, number>> {
  const starMap = new Map();

  // Batch into groups of 30 (rate limit consideration)
  for (const batch of batchArray(repos, 30)) {
    const results = await Promise.allSettled(
      batch.map(async repo => {
        const [owner, name] = repo.split('/');
        const { data } = await octokit.rest.repos.get({ owner, repo: name });
        return { repo, stars: data.stargazers_count };
      })
    );

    results.forEach(result => {
      if (result.status === 'fulfilled') {
        starMap.set(result.value.repo, result.value.stars);
      }
    });
  }

  return starMap;
}
```

**Output**: Map of repo → star count

### Step 6: Storage

**File**: `lib/search/storage.ts`

```typescript
export async function mergeMarketplaces(
  discovered: Marketplace[],
  allDiscoveredRepos: Set<string>
) {
  const existing = await readMarketplaces();
  const existingMap = new Map(existing.map(m => [m.repo, m]));

  // Update or add
  for (const marketplace of discovered) {
    const existingMarketplace = existingMap.get(marketplace.repo);

    if (existingMarketplace) {
      // UPDATE
      existingMarketplace.description = marketplace.description;
      existingMarketplace.pluginCount = marketplace.pluginCount;
      existingMarketplace.lastUpdated = new Date().toISOString();
      updated++;
    } else {
      // ADD
      existingMap.set(marketplace.repo, marketplace);
      added++;
    }
  }

  // Remove invalid entries that were discovered but failed validation
  for (const [repo] of existingMap) {
    if (allDiscoveredRepos.has(repo) && !discovered.find(m => m.repo === repo)) {
      existingMap.delete(repo);
      removed++;
    }
  }

  const merged = Array.from(existingMap.values());
  await writeMarketplaces(merged);

  return { added, updated, removed, total: merged.length };
}
```

**File Operations**:
- Read: `lib/data/marketplaces.json` or Vercel Blob
- Write: Both locations (with appropriate checks)

### Step 7: Build Time

**File**: `app/plugins/[slug]/page.tsx`

```typescript
// 1. Generate static paths
export async function generateStaticParams() {
  const marketplaces = await getAllMarketplaces();
  return marketplaces.map(marketplace => ({
    slug: marketplace.slug,
  }));
}

// 2. Generate metadata for SEO
export async function generateMetadata({ params }) {
  const { slug } = await params;
  const marketplace = await getMarketplaceBySlug(slug);

  return {
    title: `${marketplace.repo} Plugins`,
    description: marketplace.description,
    // OpenGraph, Twitter cards, etc.
  };
}

// 3. Render page
export default async function PluginsPage({ params }) {
  const { slug } = await params;

  const [marketplace, plugins, categories] = await Promise.all([
    getMarketplaceBySlug(slug),
    getPluginsByMarketplace(slug),
    getPluginCategories(slug),
  ]);

  return <PluginContent plugins={plugins} categories={categories} />;
}
```

**Process**:
1. Next.js calls `generateStaticParams()` → gets all marketplace slugs
2. For each slug, calls page component → fetches data
3. Renders to static HTML
4. Generates metadata and structured data
5. Outputs to `.next/` build directory

### Step 8: Runtime

**User Flow**:
1. User navigates to `/plugins/anthropics-claude-code`
2. CDN serves pre-rendered HTML (instant load)
3. React hydrates with static data
4. User types in search → Client-side filtering
5. User clicks category → Update URL params → Re-filter
6. No network requests needed (all data embedded)

**File**: `lib/hooks/use-plugin-filters.ts`

```typescript
export function usePluginFilters(plugins: Plugin[]) {
  const searchParams = useSearchParams();
  const searchQuery = searchParams.get('q') || '';
  const selectedCategories = searchParams.get('categories')?.split(',') || [];

  // Filter in memory (very fast)
  const filteredPlugins = useMemo(() => {
    let filtered = plugins;

    if (searchQuery) {
      filtered = filtered.filter(p =>
        p.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
        p.description.toLowerCase().includes(searchQuery.toLowerCase())
      );
    }

    if (selectedCategories.length > 0) {
      filtered = filtered.filter(p =>
        selectedCategories.includes(p.category)
      );
    }

    return filtered;
  }, [plugins, searchQuery, selectedCategories]);

  return { filteredPlugins, /* ... */ };
}
```

## Data Consistency

### When Data Gets Out of Sync

**Scenario**: Marketplace has plugins, but plugins.json doesn't include them

**Detection**:
```typescript
const hasDataSyncIssue =
  plugins.length === 0 &&
  !hasActiveFilters &&
  expectedPluginCount > 0;
```

**User Experience**:
```
Instead of: "No plugins found"
Display: "Plugins are currently being indexed.
         This marketplace has 3 plugins that will appear shortly."
```

**Fix**: Re-run `bun run scripts/search.ts`

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| GitHub Search | O(n) | n = repositories on GitHub |
| Fetch Files | O(m) | m = marketplaces found |
| Validation | O(m × p) | p = avg plugins per marketplace |
| Build | O(m) | Generate m static pages |
| Runtime Filter | O(p) | Filter plugins in memory |

### Space Complexity

| Component | Size | Notes |
|-----------|------|-------|
| marketplaces.json | ~2MB | 442 marketplaces |
| plugins.json | ~5MB | ~3000 plugins |
| Build output | ~100MB | All static pages |
| Runtime memory | <1MB | Only filtered subset |

## Error Handling

Errors are handled at each step:

1. **Discovery**: Log failed searches, continue
2. **Fetch**: Skip inaccessible repos, log warning
3. **Validation**: Mark as invalid, log errors
4. **Extraction**: Skip malformed plugins, log errors
5. **Storage**: Rollback on write failure
6. **Build**: Fail build on missing data
7. **Runtime**: Show user-friendly message

## Debugging Data Flow

To trace data flow for a specific marketplace:

```bash
# 1. Search for marketplace
bun run scripts/search.ts --verbose | grep "yamadashy/repomix"

# 2. Check if validated
cat lib/data/marketplaces.json | grep "yamadashy/repomix"

# 3. Check plugins extracted
cat lib/data/plugins.json | grep "yamadashy-repomix"

# 4. Verify build includes it
bun run build | grep "yamadashy-repomix"

# 5. Check generated page exists
ls .next/server/app/plugins/yamadashy-repomix/
```
