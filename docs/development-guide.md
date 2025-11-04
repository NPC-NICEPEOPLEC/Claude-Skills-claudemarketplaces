# Development Guide

**TL;DR**: Before adding features, understand the data flow (GitHub → JSON → Static Pages). Follow existing patterns (Server Components for data, Client Components for interaction). Run the search script to update data. Test locally with `bun dev`.

## Before You Start

### Required Reading
1. [Architecture Overview](./architecture.md) - Understand the system design
2. [Data Flow](./data-flow.md) - Know how data moves through the system
3. [Components](./components.md) - Understand component patterns

### Prerequisites
- **Bun** installed (package manager)
- **Node.js** 18+ (for compatibility)
- **Git** for version control
- **GitHub token** (for running search script)

### Environment Setup

```bash
# Clone the repository
git clone <repo-url>
cd claude-plugins

# Install dependencies
bun install

# Set up environment variables
cp .env.example .env.local

# Add your GitHub token (for search script)
echo "GITHUB_TOKEN=ghp_your_token_here" >> .env.local

# (Optional) Add Vercel Blob token for production data storage
echo "BLOB_READ_WRITE_TOKEN=vercel_blob_..." >> .env.local
```

## Development Workflow

### 1. Start Development Server

```bash
bun dev
```

Access at http://localhost:3000

**Features**:
- Hot module replacement
- Turbopack for fast builds
- Error overlay for debugging

### 2. Make Changes

Follow the patterns below based on what you're changing.

### 3. Test Locally

```bash
# Type checking
bun run type-check

# Linting
bun run lint

# Build test
bun run build

# Run development server
bun dev
```

### 4. Update Data (if needed)

```bash
# Re-run search script to update marketplace/plugin data
bun run scripts/search.ts

# Or with verbose logging
bun run scripts/search.ts --verbose

# Or test with limited results
bun run scripts/search.ts --limit 10
```

### 5. Commit Changes

Follow conventional commits:
```bash
git commit -m "feat: add plugin detail page"
git commit -m "fix: resolve data sync issue"
git commit -m "docs: update component documentation"
```

## Common Development Tasks

### Adding a New Page

**Example**: Add a plugin detail page

1. **Create the page file**:
   ```typescript
   // app/plugin-details/[marketplace]/[plugin]/page.tsx
   export default async function PluginDetailPage({ params }) {
     const { marketplace, plugin } = await params;

     // Fetch plugin data
     const pluginData = await getPluginById(`${marketplace}/${plugin}`);

     return <div>{/* Render plugin details */}</div>;
   }
   ```

2. **Generate static params** (for SSG):
   ```typescript
   export async function generateStaticParams() {
     const plugins = await getAllPlugins();

     return plugins.map(plugin => {
       const [marketplace, name] = plugin.id.split('/');
       return { marketplace, plugin: name };
     });
   }
   ```

3. **Add metadata for SEO**:
   ```typescript
   export async function generateMetadata({ params }): Promise<Metadata> {
     const pluginData = await getPluginById(/* ... */);

     return {
       title: `${pluginData.name} - Claude Code Plugin`,
       description: pluginData.description,
     };
   }
   ```

4. **Test the route**:
   ```bash
   # Visit the new page
   open http://localhost:3000/plugin-details/anthropics-claude-code/agent-sdk-dev
   ```

### Adding a New Component

**Example**: Add a plugin comparison component

1. **Decide**: Server or Client Component?
   - Needs user interaction? → Client (`"use client"`)
   - Static rendering? → Server (default)

2. **Create the component file**:
   ```typescript
   // components/plugin-comparison.tsx
   "use client"; // If interactive

   import { Plugin } from "@/lib/types";
   import { Card } from "@/components/ui/card";

   interface PluginComparisonProps {
     plugins: Plugin[];
   }

   export function PluginComparison({ plugins }: PluginComparisonProps) {
     return (
       <div className="grid grid-cols-2 gap-4">
         {plugins.map(plugin => (
           <Card key={plugin.id}>
             <h3>{plugin.name}</h3>
             <p>{plugin.description}</p>
           </Card>
         ))}
       </div>
     );
   }
   ```

3. **Use the component**:
   ```typescript
   // In a page
   import { PluginComparison } from "@/components/plugin-comparison";

   export default async function Page() {
     const plugins = await getPlugins();
     return <PluginComparison plugins={plugins} />;
   }
   ```

### Adding a Filter or Search Feature

**Pattern**: Use URL query parameters + client-side filtering

1. **Create a custom hook**:
   ```typescript
   // lib/hooks/use-custom-filter.ts
   "use client";

   export function useCustomFilter(data: any[]) {
     const searchParams = useSearchParams();
     const router = useRouter();
     const pathname = usePathname();

     const filterValue = searchParams.get('filter') || '';

     const filteredData = useMemo(() => {
       return data.filter(/* filter logic */);
     }, [data, filterValue]);

     const setFilter = useCallback((value: string) => {
       const params = new URLSearchParams(searchParams);
       if (value) {
         params.set('filter', value);
       } else {
         params.delete('filter');
       }
       router.replace(`${pathname}?${params.toString()}`, { scroll: false });
     }, [searchParams, router, pathname]);

     return { filterValue, filteredData, setFilter };
   }
   ```

2. **Use in a Client Component**:
   ```typescript
   "use client";

   export function FilteredList({ data }) {
     const { filterValue, filteredData, setFilter } = useCustomFilter(data);

     return (
       <div>
         <input value={filterValue} onChange={e => setFilter(e.target.value)} />
         {filteredData.map(/* render */)}
       </div>
     );
   }
   ```

### Adding a New Data Field

**Example**: Add `downloads` count to plugins

1. **Update TypeScript interface**:
   ```typescript
   // lib/types.ts
   export interface Plugin {
     // ... existing fields
     downloads?: number; // Add new field
   }
   ```

2. **Update data extraction**:
   ```typescript
   // lib/search/plugin-extractor.ts
   const transformedPlugin: Plugin = {
     // ... existing fields
     downloads: plugin.downloads || 0, // Extract new field
   };
   ```

3. **Update validation schema** (if needed):
   ```typescript
   // lib/schemas/marketplace.schema.ts
   const PluginSchema = z.object({
     // ... existing fields
     downloads: z.number().optional(),
   });
   ```

4. **Update UI to display**:
   ```typescript
   // components/plugin-card.tsx
   {plugin.downloads && (
     <Badge>{plugin.downloads} downloads</Badge>
   )}
   ```

5. **Re-run search script**:
   ```bash
   bun run scripts/search.ts
   ```

### Adding a New UI Component (shadcn/ui)

```bash
# Install a new shadcn/ui component
bunx shadcn@latest add dialog

# Use it in your code
import { Dialog, DialogContent, DialogHeader } from "@/components/ui/dialog";
```

**Available components**: button, card, badge, input, select, dialog, sheet, dropdown-menu, popover, tooltip, alert, tabs, etc.

### Modifying the Search Script

**Example**: Add filtering by language

1. **Update search query**:
   ```typescript
   // lib/search/github-search.ts
   const query = "filename:marketplace.json path:.claude-plugin language:typescript";
   ```

2. **Add command-line option**:
   ```typescript
   // scripts/search.ts
   interface CliArgs {
     // ... existing args
     language?: string;
   }

   function parseArgs(): CliArgs {
     // ... parse --language flag
   }
   ```

3. **Test the change**:
   ```bash
   bun run scripts/search.ts --language typescript --limit 10
   ```

## Testing Strategies

### Manual Testing

1. **Test with real data**:
   ```bash
   # Update to latest data
   bun run scripts/search.ts

   # Start dev server
   bun dev

   # Test various scenarios
   # - Empty search results
   # - Large result sets
   # - Edge cases (0 plugins, very long descriptions, etc.)
   ```

2. **Test builds**:
   ```bash
   bun run build

   # Check build output
   ls -lh .next/

   # Test production build locally
   bun start
   ```

3. **Test different viewports**:
   - Mobile (< 640px)
   - Tablet (640px - 1024px)
   - Desktop (> 1024px)

### Debugging

**TypeScript errors**:
```bash
# Check types
bun run type-check
```

**Build errors**:
```bash
# Verbose build
bun run build --debug

# Check Next.js configuration
cat next.config.mjs
```

**Runtime errors**:
- Check browser console
- Check terminal output
- Use React DevTools
- Add console.log strategically

**Data issues**:
```bash
# Inspect JSON files
cat lib/data/marketplaces.json | grep "yamadashy"
cat lib/data/plugins.json | grep "yamadashy-repomix"

# Validate JSON syntax
bun run -e "JSON.parse(require('fs').readFileSync('lib/data/plugins.json', 'utf-8'))"
```

## Code Style Guide

### TypeScript

```typescript
// Use interfaces for data shapes
interface Marketplace {
  repo: string;
  slug: string;
}

// Use async/await (not .then())
const data = await fetchData();

// Use optional chaining
const stars = marketplace?.stars ?? 0;

// Destructure props
function Component({ title, description }: Props) {
  // ...
}
```

### React

```tsx
// Prefer named exports for components
export function MyComponent() {
  // ...
}

// Use semantic HTML
<main>
  <article>
    <header>
      <h1>{title}</h1>
    </header>
  </article>
</main>

// Keep components focused (single responsibility)
// Extract complex logic to hooks
const { filtered, setFilter } = useCustomFilter(data);
```

### Tailwind CSS

```tsx
// Use utility classes
<div className="container mx-auto px-4 py-8">

// Use cn() for conditional classes
<div className={cn(
  "base-class",
  isActive && "active-class",
  className
)} />

// Follow responsive design patterns
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
```

## Performance Best Practices

### 1. Minimize Client Components

```tsx
// ❌ Avoid: Entire page as Client Component
"use client";
export default function Page() {
  const [filter, setFilter] = useState('');
  return <div>{/* Everything is client-rendered */}</div>;
}

// ✅ Better: Server Component wrapping Client Component
export default async function Page() {
  const data = await fetchData(); // Server-side
  return <FilteredList data={data} />; // Only this is client-side
}
```

### 2. Use useMemo for Expensive Operations

```tsx
const filteredData = useMemo(() => {
  return data.filter(/* expensive operation */);
}, [data, filters]);
```

### 3. Optimize Images

```tsx
import Image from "next/image";

<Image
  src="/marketplace-icon.png"
  alt="Marketplace icon"
  width={48}
  height={48}
  priority={isAboveTheFold}
/>
```

### 4. Lazy Load Heavy Components

```tsx
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./heavy-component'), {
  loading: () => <LoadingSkeleton />,
});
```

## Git Workflow

### Branch Naming

```bash
git checkout -b feature/add-plugin-detail-page
git checkout -b fix/data-sync-issue
git checkout -b docs/update-architecture
```

### Commit Messages

Follow Conventional Commits:

```bash
feat: add plugin detail page
fix: resolve data sync for yamadashy/repomix
docs: update component documentation
refactor: extract filter logic to hook
style: format code with prettier
test: add tests for plugin extraction
chore: update dependencies
```

### Pull Request Checklist

- [ ] Code builds without errors (`bun run build`)
- [ ] Linting passes (`bun run lint`)
- [ ] TypeScript types are correct
- [ ] Tested locally with real data
- [ ] Responsive design verified
- [ ] Accessibility considered
- [ ] Documentation updated (if needed)
- [ ] Commit messages follow conventions

## Deployment

### Vercel Deployment

1. **Push to GitHub**:
   ```bash
   git push origin main
   ```

2. **Automatic deployment**:
   - Vercel watches the repository
   - Runs build on push
   - Deploys if successful

3. **Environment variables** (Vercel dashboard):
   ```
   GITHUB_TOKEN=ghp_...
   BLOB_READ_WRITE_TOKEN=vercel_blob_...
   ```

### Manual Deployment

```bash
# Install Vercel CLI
bun add -g vercel

# Deploy
vercel --prod
```

## Troubleshooting Development Issues

See [Troubleshooting Guide](./troubleshooting.md) for common issues and solutions.

## Getting Help

1. Check existing documentation in `/docs`
2. Search for similar code patterns in the codebase
3. Read Next.js 15 documentation
4. Check shadcn/ui component documentation
5. Review GitHub issues for similar problems

## Next Steps

After reading this guide:
1. Set up your development environment
2. Run `bun dev` and explore the app
3. Make a small change and test it
4. Read [Troubleshooting](./troubleshooting.md) for common issues
5. Consult [API Reference](./api-reference.md) for function signatures
