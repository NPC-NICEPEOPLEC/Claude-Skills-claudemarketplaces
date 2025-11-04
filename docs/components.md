# Component Architecture

**TL;DR**: Server-first React components using Next.js App Router. Pages are Server Components that fetch data at build time. Client Components handle interactivity (search, filters). shadcn/ui provides accessible UI primitives.

## Component Tree

```
app/
├── layout.tsx (RootLayout)                    [Server Component]
│   ├── <html>
│   ├── <body>
│   │   └── {children}
│   └── fonts, metadata
│
├── page.tsx (HomePage)                        [Server Component]
│   ├── <Header />
│   ├── <MarketplaceGrid />                   [Client Component]
│   │   ├── <MarketplaceSearch />             [Client Component]
│   │   ├── Category Filter Badges
│   │   └── <MarketplaceCard /> × N           [Server Component]
│   └── <Footer />
│
└── plugins/[slug]/page.tsx (PluginPage)      [Server Component]
    ├── <Header />
    ├── <Breadcrumb />                         [shadcn/ui]
    ├── Marketplace Header (title, description, stats)
    ├── <PluginContent />                      [Client Component]
    │   ├── <MarketplaceSearch />
    │   ├── Category Filter Badges
    │   └── <PluginCard /> × N                 [Server Component]
    └── <Footer />
```

## Core Pages

### app/layout.tsx

**Type**: Server Component
**Purpose**: Root layout defining document structure

```typescript
export default function RootLayout({ children }) {
  return (
    <html lang="en" className={`${geistSans.variable} ${geistMono.variable}`}>
      <body className="antialiased">
        {children}
      </body>
    </html>
  );
}

export const metadata: Metadata = {
  title: "Claude Code Marketplaces",
  description: "...",
};
```

**Responsibilities**:
- Load fonts (Geist Sans, Geist Mono)
- Set global metadata
- Apply global CSS
- Wrap all pages

### app/page.tsx (Home Page)

**Type**: Server Component
**Purpose**: Display marketplace grid with search/filter

```typescript
export default async function Home() {
  // Fetch data at build time
  const marketplaces = await getAllMarketplaces({
    includeEmpty: false, // Filter out marketplaces with 0 plugins
  });
  const categories = await getCategories();

  return (
    <div>
      <Header />
      <main>
        <h1>Claude Code Marketplaces</h1>
        <MarketplaceGrid
          marketplaces={marketplaces}
          categories={categories}
        />
      </main>
      <Footer />
    </div>
  );
}
```

**Data Fetching**:
- `getAllMarketplaces()` - Reads from marketplaces.json
- `getCategories()` - Extracts unique categories
- Happens at build time (SSG)
- No runtime data fetching

### app/plugins/[slug]/page.tsx (Plugin List Page)

**Type**: Server Component
**Purpose**: Display plugins for a specific marketplace

```typescript
// Generate static paths for all marketplaces
export async function generateStaticParams() {
  const marketplaces = await getAllMarketplaces();
  return marketplaces.map(marketplace => ({
    slug: marketplace.slug,
  }));
}

// Generate SEO metadata
export async function generateMetadata({ params }): Promise<Metadata> {
  const { slug } = await params;
  const marketplace = await getMarketplaceBySlug(slug);

  return {
    title: `${marketplace.repo} Plugins`,
    description: marketplace.description,
    // OpenGraph, Twitter, structured data
  };
}

// Render page
export default async function PluginsPage({ params }) {
  const { slug } = await params;

  const [marketplace, plugins, categories] = await Promise.all([
    getMarketplaceBySlug(slug),
    getPluginsByMarketplace(slug),
    getPluginCategories(slug),
  ]);

  if (!marketplace) {
    notFound();
  }

  return (
    <div>
      <Header />
      <main>
        <Breadcrumb />
        <MarketplaceHeader marketplace={marketplace} />
        <PluginContent
          plugins={plugins}
          categories={categories}
          expectedPluginCount={marketplace.pluginCount}
        />
      </main>
      <Footer />
    </div>
  );
}
```

**Key Features**:
- Static generation for all marketplace pages
- SEO-optimized with metadata
- Structured data for search engines
- Handles not-found cases

## Client Components

### components/marketplace-grid.tsx

**Type**: Client Component (`"use client"`)
**Purpose**: Interactive marketplace browsing with filters

```typescript
"use client";

export function MarketplaceGrid({ marketplaces, categories }) {
  const {
    searchQuery,
    selectedCategories,
    filteredMarketplaces,
    setSearchQuery,
    toggleCategory,
    clearFilters,
  } = useMarketplaceFilters(marketplaces);

  return (
    <div>
      {/* Search */}
      <MarketplaceSearch value={searchQuery} onChange={setSearchQuery} />

      {/* Category Filters */}
      <div>
        {categories.map(category => (
          <Badge
            key={category}
            variant={selectedCategories.includes(category) ? "default" : "outline"}
            onClick={() => toggleCategory(category)}
          >
            {category}
          </Badge>
        ))}
      </div>

      {/* Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {filteredMarketplaces.map(marketplace => (
          <MarketplaceCard key={marketplace.repo} marketplace={marketplace} />
        ))}
      </div>
    </div>
  );
}
```

**Why Client Component**:
- Uses React hooks (`useMarketplaceFilters`)
- Handles user interactions (search, filter clicks)
- Updates URL parameters
- Manages filter state

**Props**:
- `marketplaces: Marketplace[]` - All marketplaces (from server)
- `categories: string[]` - All unique categories

### components/plugin-content.tsx

**Type**: Client Component
**Purpose**: Plugin list with search and filters

```typescript
"use client";

export function PluginContent({ plugins, categories, expectedPluginCount }) {
  const {
    searchQuery,
    selectedCategories,
    filteredPlugins,
    filteredCount,
    setSearchQuery,
    toggleCategory,
    clearFilters,
  } = usePluginFilters(plugins);

  const hasActiveFilters = searchQuery || selectedCategories.length > 0;

  // Data sync detection
  const hasDataSyncIssue =
    plugins.length === 0 &&
    !hasActiveFilters &&
    expectedPluginCount &&
    expectedPluginCount > 0;

  return (
    <div>
      <MarketplaceSearch value={searchQuery} onChange={setSearchQuery} />

      {/* Category badges */}
      {categories.map(category => (
        <Badge
          key={category}
          variant={selectedCategories.includes(category) ? "default" : "outline"}
          onClick={() => toggleCategory(category)}
        >
          {category}
        </Badge>
      ))}

      {/* Results count */}
      <p>{filteredCount} plugins</p>

      {/* Grid or empty state */}
      {filteredPlugins.length > 0 ? (
        <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
          {filteredPlugins.map(plugin => (
            <PluginCard key={plugin.id} plugin={plugin} />
          ))}
        </div>
      ) : (
        <EmptyState
          hasDataSyncIssue={hasDataSyncIssue}
          expectedPluginCount={expectedPluginCount}
          hasActiveFilters={hasActiveFilters}
          onClearFilters={clearFilters}
        />
      )}
    </div>
  );
}
```

**Props**:
- `plugins: Plugin[]` - Plugins for this marketplace
- `categories: string[]` - Unique categories
- `expectedPluginCount?: number` - For data sync detection

## Server Components

### components/marketplace-card.tsx

**Type**: Server Component (default)
**Purpose**: Display marketplace summary card

```typescript
export function MarketplaceCard({ marketplace }: { marketplace: Marketplace }) {
  return (
    <Link href={`/plugins/${marketplace.slug}`}>
      <Card>
        <CardHeader>
          <div className="flex items-start justify-between">
            <CardTitle>{marketplace.repo}</CardTitle>
            {marketplace.stars > 0 && (
              <Badge variant="secondary">
                ⭐ {marketplace.stars}
              </Badge>
            )}
          </div>
          <CardDescription>{marketplace.description}</CardDescription>
        </CardHeader>
        <CardContent>
          <div className="flex items-center gap-4">
            <span>{marketplace.pluginCount} plugins</span>
          </div>
          <div className="flex flex-wrap gap-2 mt-4">
            {marketplace.categories.map(category => (
              <Badge key={category} variant="outline">
                {category}
              </Badge>
            ))}
          </div>
        </CardContent>
      </Card>
    </Link>
  );
}
```

**Why Server Component**:
- No interactivity needed
- Pure rendering from props
- Can be cached effectively

### components/plugin-card.tsx

**Type**: Server Component
**Purpose**: Display plugin details card

```typescript
export function PluginCard({ plugin }: { plugin: Plugin }) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-start justify-between">
          <div>
            <CardTitle className="text-lg">{plugin.name}</CardTitle>
            {plugin.version && (
              <Badge variant="secondary">v{plugin.version}</Badge>
            )}
          </div>
        </div>
        <CardDescription>{plugin.description}</CardDescription>
      </CardHeader>
      <CardContent>
        <div className="space-y-3">
          {/* Category badge */}
          <Badge variant="outline">{plugin.category}</Badge>

          {/* Install command */}
          <InstallCommand command={plugin.installCommand} />

          {/* Metadata */}
          {plugin.author && <p>by {plugin.author.name}</p>}
          {plugin.commands && <p>{plugin.commands.length} commands</p>}
          {plugin.agents && <p>{plugin.agents.length} agents</p>}
        </div>
      </CardContent>
    </Card>
  );
}
```

## Shared Components

### components/header.tsx

**Type**: Server Component
**Purpose**: Site header with branding and navigation

```typescript
export function Header() {
  return (
    <header className="border-b">
      <div className="container mx-auto px-4 py-6">
        <Link href="/">
          <h1 className="text-2xl font-bold font-serif">
            CLAUDE CODE MARKETPLACES
          </h1>
        </Link>
        <p className="text-muted-foreground">
          A comprehensive directory for discovering plugin marketplaces
        </p>
      </div>
    </header>
  );
}
```

### components/footer.tsx

**Type**: Server Component
**Purpose**: Site footer with links

```typescript
export function Footer() {
  return (
    <footer className="border-t mt-auto">
      <div className="container mx-auto px-4 py-8">
        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          <div>
            <h3>Claude Code Marketplaces</h3>
            <p>Discover Claude Code plugins, extensions, and tools.</p>
          </div>
          <div>
            <h4>Resources</h4>
            <ul>
              <li><Link href="#">Plugin Marketplaces</Link></li>
            </ul>
          </div>
          <div>
            <h4>Community</h4>
            <ul>
              <li><Link href="#">About</Link></li>
            </ul>
          </div>
        </div>
      </div>
    </footer>
  );
}
```

### components/marketplace-search.tsx

**Type**: Client Component
**Purpose**: Search input with live filtering

```typescript
"use client";

export function MarketplaceSearch({ value, onChange }) {
  return (
    <div className="relative">
      <Search className="absolute left-3 top-3 h-4 w-4" />
      <Input
        type="search"
        placeholder="Search by name, description, or category..."
        value={value}
        onChange={(e) => onChange(e.target.value)}
        className="pl-9"
      />
    </div>
  );
}
```

## shadcn/ui Components

The project uses shadcn/ui for accessible, customizable primitives:

### ui/card.tsx

```typescript
// Compound component pattern
export function Card({ className, ...props }) {
  return <div className={cn("rounded-lg border bg-card", className)} {...props} />;
}

export function CardHeader({ className, ...props }) {
  return <div className={cn("flex flex-col space-y-1.5 p-6", className)} {...props} />;
}

export function CardTitle({ className, ...props }) {
  return <h3 className={cn("text-2xl font-semibold", className)} {...props} />;
}

// ... CardDescription, CardContent, CardFooter
```

**Usage**:
```tsx
<Card>
  <CardHeader>
    <CardTitle>Title</CardTitle>
    <CardDescription>Description</CardDescription>
  </CardHeader>
  <CardContent>Content</CardContent>
</Card>
```

### ui/badge.tsx

```typescript
const badgeVariants = cva(
  "inline-flex items-center rounded-full border px-2.5 py-0.5 text-xs",
  {
    variants: {
      variant: {
        default: "border-transparent bg-primary text-primary-foreground",
        secondary: "border-transparent bg-secondary text-secondary-foreground",
        outline: "text-foreground",
      },
    },
    defaultVariants: {
      variant: "default",
    },
  }
);

export function Badge({ className, variant, ...props }) {
  return <div className={cn(badgeVariants({ variant }), className)} {...props} />;
}
```

### ui/input.tsx

```typescript
export const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({ className, type, ...props }, ref) => {
    return (
      <input
        type={type}
        className={cn(
          "flex h-10 w-full rounded-md border bg-background px-3 py-2",
          "focus-visible:outline-none focus-visible:ring-2",
          className
        )}
        ref={ref}
        {...props}
      />
    );
  }
);
```

### ui/breadcrumb.tsx

```typescript
// Used for navigation on plugin pages
export function Breadcrumb({ children }) {
  return <nav aria-label="breadcrumb">{children}</nav>;
}

export function BreadcrumbList({ children }) {
  return <ol className="flex items-center gap-2">{children}</ol>;
}

export function BreadcrumbItem({ children }) {
  return <li>{children}</li>;
}

// ... BreadcrumbLink, BreadcrumbPage, BreadcrumbSeparator
```

## Custom Hooks

### lib/hooks/use-marketplace-filters.ts

**Purpose**: Manage marketplace search and filter state

```typescript
export function useMarketplaceFilters(marketplaces: Marketplace[]) {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  // Read from URL
  const searchQuery = searchParams.get('q') || '';
  const selectedCategories = searchParams.get('categories')?.split(',') || [];

  // Update URL function
  const updateURL = useCallback((params: Record<string, string | null>) => {
    const newParams = new URLSearchParams(searchParams);
    Object.entries(params).forEach(([key, value]) => {
      if (value) {
        newParams.set(key, value);
      } else {
        newParams.delete(key);
      }
    });
    router.replace(`${pathname}?${newParams.toString()}`, { scroll: false });
  }, [searchParams, router, pathname]);

  // Filter marketplaces
  const filteredMarketplaces = useMemo(() => {
    let filtered = marketplaces;

    if (searchQuery) {
      const query = searchQuery.toLowerCase();
      filtered = filtered.filter(m =>
        m.repo.toLowerCase().includes(query) ||
        m.description.toLowerCase().includes(query) ||
        m.categories.some(c => c.toLowerCase().includes(query))
      );
    }

    if (selectedCategories.length > 0) {
      filtered = filtered.filter(m =>
        m.categories.some(c => selectedCategories.includes(c))
      );
    }

    return filtered.sort((a, b) => (b.stars || 0) - (a.stars || 0));
  }, [marketplaces, searchQuery, selectedCategories]);

  return {
    searchQuery,
    selectedCategories,
    filteredMarketplaces,
    filteredCount: filteredMarketplaces.length,
    setSearchQuery: (q: string) => updateURL({ q: q || null }),
    toggleCategory: (cat: string) => {
      const newCats = selectedCategories.includes(cat)
        ? selectedCategories.filter(c => c !== cat)
        : [...selectedCategories, cat];
      updateURL({ categories: newCats.length ? newCats.join(',') : null });
    },
    clearFilters: () => updateURL({ q: null, categories: null }),
  };
}
```

**Features**:
- URL-based state management
- Debounced search
- Multi-select category filtering
- Browser back/forward support

### lib/hooks/use-plugin-filters.ts

**Purpose**: Same as above but for plugins
- Nearly identical implementation
- Filters Plugin[] instead of Marketplace[]

## Component Patterns

### Server vs Client Component Decision Tree

```
Is the component interactive?
├─ Yes → Client Component ("use client")
│  Examples:
│  - Search inputs
│  - Filter toggles
│  - Forms
│  - Components using hooks
│
└─ No → Server Component (default)
   Examples:
   - Static headers/footers
   - Display cards
   - Layouts
   - Data fetching components
```

### Data Fetching Pattern

```typescript
// Page (Server Component) fetches data
export default async function Page() {
  const data = await fetchData(); // At build time (SSG)

  return <ClientComponent data={data} />;
}

// Client Component receives data as props
"use client";
export function ClientComponent({ data }) {
  // Use data for interactions
  const filtered = useMemo(() => filterData(data), [data]);

  return <div>...</div>;
}
```

**Benefits**:
- Server fetches at build time (fast)
- Client gets static data (no loading states)
- Client handles interactions only

### URL State Management

```typescript
// Read from URL
const searchParams = useSearchParams();
const query = searchParams.get('q');

// Write to URL
const router = useRouter();
const pathname = usePathname();

router.replace(`${pathname}?q=${newQuery}`, { scroll: false });
```

**Benefits**:
- Shareable URLs
- Browser history works
- No complex state management

## Styling Approach

### Tailwind Utility Classes

```tsx
<div className="container mx-auto px-4 py-8">
  <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
    <Card className="hover:shadow-lg transition-shadow">
      {/* ... */}
    </Card>
  </div>
</div>
```

### CSS Variables for Theming

```css
/* app/globals.css */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --card: 0 0% 100%;
  --card-foreground: 222.2 84% 4.9%;
  --primary: 221.2 83.2% 53.3%;
  /* ... */
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... */
  }
}
```

### cn() Utility for Conditional Classes

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Usage
<div className={cn(
  "base-class",
  isActive && "active-class",
  className // Allow prop overrides
)} />
```

## Performance Optimizations

### React.memo for Expensive Renders

```typescript
export const PluginCard = React.memo(function PluginCard({ plugin }) {
  // Only re-renders if plugin prop changes
  return <Card>...</Card>;
});
```

### useMemo for Expensive Computations

```typescript
const filteredPlugins = useMemo(() => {
  // Only recompute when dependencies change
  return plugins.filter(/* ... */);
}, [plugins, searchQuery, selectedCategories]);
```

### Suspense Boundaries

```tsx
<Suspense fallback={<LoadingSkeleton />}>
  <PluginData slug={slug} />
</Suspense>
```

## Accessibility

shadcn/ui components are built on Radix UI primitives, ensuring:
- ARIA attributes
- Keyboard navigation
- Focus management
- Screen reader support

Example:
```tsx
<Badge variant="outline">
  {/* Automatically includes appropriate ARIA roles */}
</Badge>
```

## Component Checklist

When creating a new component:

- [ ] Decide: Server or Client Component?
- [ ] Add TypeScript interface for props
- [ ] Use `cn()` for className merging
- [ ] Follow existing naming conventions
- [ ] Add JSDoc comments for complex logic
- [ ] Consider accessibility (ARIA, keyboard)
- [ ] Test with different data scenarios
- [ ] Verify responsive behavior
