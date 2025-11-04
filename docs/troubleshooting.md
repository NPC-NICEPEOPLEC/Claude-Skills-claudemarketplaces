# Troubleshooting Guide

**TL;DR**: Most issues are caused by data sync problems (outdated JSON files) or build cache. Solution: Re-run `bun run scripts/search.ts` and `bun run build --no-cache`.

## Common Issues

### 1. Plugins Not Showing for a Marketplace

**Symptoms**:
- Marketplace page shows "N plugins" in header
- But displays "0 plugins found" or "No plugins found matching your criteria"
- No filter is active

**Root Cause**:
Data sync issue between `marketplaces.json` and `plugins.json`. The marketplace metadata says it has plugins, but they weren't extracted into the plugin database.

**Solution**:

```bash
# Re-run the search script to extract plugins
bun run scripts/search.ts --verbose

# Verify plugins were extracted
grep -c "yamadashy-repomix" lib/data/plugins.json

# Rebuild the site
bun run build

# Test locally
bun dev
```

**Prevention**:
The search script should always be run to completion. If it errors partway through, marketplaces may be added without their plugins.

**User Experience Fix**:
The system now detects this scenario and displays:
```
"Plugins are currently being indexed.
This marketplace has N plugins that will appear shortly."
```

**Related Files**:
- `components/plugin-content.tsx:30` - Detection logic
- `scripts/search.ts:234` - Plugin extraction step
- `lib/search/plugin-extractor.ts` - Extraction implementation

---

### 2. Marketplace Not Appearing

**Symptoms**:
- You know a marketplace exists on GitHub
- It's not in the marketplace grid

**Root Cause**:
The marketplace either:
1. Hasn't been discovered yet
2. Failed validation
3. Has `pluginCount: 0` and filters are hiding it

**Solution**:

```bash
# Option 1: Re-run search (will discover new marketplaces)
bun run scripts/search.ts --verbose

# Option 2: Check if it exists but failed validation
cat lib/data/marketplaces.json | grep "owner/repo"

# Option 3: Check if it has 0 plugins (filtered out by default)
# Look in the JSON file
cat lib/data/marketplaces.json | grep -A 5 "owner/repo"
```

**Debugging**:

```bash
# Test with a specific repo
bun run scripts/search.ts --verbose 2>&1 | grep "owner/repo"

# Check validation errors
bun run scripts/search.ts --verbose 2>&1 | grep -A 10 "Failed:"
```

**Common Validation Failures**:
- Invalid JSON syntax in marketplace.json
- Missing required fields (name, plugins array)
- Invalid email addresses in author fields
- Repository is private or deleted
- `.claude-plugin/marketplace.json` file doesn't exist

---

### 3. Build Fails with "Module not found"

**Symptoms**:
```
Error: Cannot find module '@/components/some-component'
```

**Root Cause**:
- Import path is incorrect
- File doesn't exist
- TypeScript path alias not configured correctly

**Solution**:

```bash
# Check if file exists
ls components/some-component.tsx

# Verify path alias in tsconfig.json
cat tsconfig.json | grep "@/*"

# Should see:
# "paths": {
#   "@/*": ["./*"]
# }

# Clear Next.js cache
rm -rf .next

# Rebuild
bun run build
```

**Common Import Mistakes**:

```tsx
// ❌ Wrong
import { Component } from "components/component";
import { Component } from "/components/component";

// ✅ Correct
import { Component } from "@/components/component";
```

---

### 4. GitHub API Rate Limit Exceeded

**Symptoms**:
```
Error: API rate limit exceeded for user
```

**Root Cause**:
- Too many GitHub API requests
- Unauthenticated requests have lower limits (60/hour vs 5000/hour)
- Running search script multiple times quickly

**Solution**:

```bash
# Check current rate limit status
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/rate_limit

# Wait for rate limit to reset (shown in response)
# OR

# Add/update GitHub token with higher limits
echo "GITHUB_TOKEN=ghp_your_token_here" >> .env.local

# Use --limit flag to test with fewer requests
bun run scripts/search.ts --limit 10
```

**Prevention**:
- Always use a GitHub token
- Use `--dry-run` flag when testing
- Use `--limit` flag for quick tests

---

### 5. Build Takes Too Long

**Symptoms**:
- Build hangs or takes >10 minutes
- High memory usage during build

**Root Cause**:
- Generating static pages for 400+ marketplaces
- Large JSON files being processed
- Cache issues

**Solution**:

```bash
# Clear build cache
rm -rf .next

# Use Turbopack (faster)
bun run build --turbopack

# Check for memory issues
node --max-old-space-size=4096 $(which next) build

# Limit number of marketplaces (testing)
# Edit lib/data/marketplaces.json manually or filter in code
```

**Optimization**:
Consider implementing ISR (Incremental Static Regeneration) for frequently changing pages.

---

### 6. Filters Not Working

**Symptoms**:
- Typing in search doesn't filter results
- Category badges don't filter when clicked
- URL updates but results don't change

**Root Cause**:
- Client Component not hydrating properly
- JavaScript error preventing filtering
- useMemo dependencies incorrect

**Solution**:

```bash
# Check browser console for errors
# Open DevTools → Console

# Verify "use client" directive
cat components/marketplace-grid.tsx | head -1
# Should see: "use client";

# Check useMemo dependencies
# In lib/hooks/use-marketplace-filters.ts
# Dependencies should include all variables used in filter logic
```

**Debug Code**:

```tsx
// Add logging to filter hook
const filteredMarketplaces = useMemo(() => {
  console.log('Filtering with:', { searchQuery, selectedCategories });
  const result = marketplaces.filter(/* ... */);
  console.log('Filtered result:', result.length);
  return result;
}, [marketplaces, searchQuery, selectedCategories]);
```

---

### 7. Data Not Updating After Running Search Script

**Symptoms**:
- Ran `bun run scripts/search.ts`
- Script completed successfully
- But website still shows old data

**Root Cause**:
- Build cache not cleared
- Development server still running with old data
- JSON files not being read correctly

**Solution**:

```bash
# Stop development server (Ctrl+C)

# Clear Next.js cache
rm -rf .next

# Verify JSON files updated
ls -lh lib/data/*.json
# Check modification times

# Rebuild from scratch
bun run build

# Restart development server
bun dev
```

**Production**:
If deployed to Vercel:
- Data is stored in Vercel Blob
- Re-deploy to update data
- Or run search script from Vercel cron job

---

### 8. TypeScript Errors

**Symptoms**:
```
Type 'X' is not assignable to type 'Y'
Property 'foo' does not exist on type 'Bar'
```

**Root Cause**:
- Type definitions out of sync
- Missing type imports
- Incorrect type usage

**Solution**:

```bash
# Check TypeScript version
bun run --bun tsc --version

# Run type checker
bun run type-check

# Check types for specific file
bunx tsc --noEmit components/marketplace-grid.tsx

# Restart TypeScript server in IDE
# VS Code: Cmd+Shift+P → "TypeScript: Restart TS Server"
```

**Common Fixes**:

```typescript
// ❌ Wrong: Using wrong type
const marketplace: Plugin = { /* marketplace data */ };

// ✅ Correct
const marketplace: Marketplace = { /* marketplace data */ };

// ❌ Wrong: Optional chaining not handled
const count = marketplace.pluginCount.toString();

// ✅ Correct
const count = marketplace.pluginCount?.toString() ?? '0';
```

---

### 9. Styling Issues / Classes Not Applying

**Symptoms**:
- Tailwind classes don't work
- Custom CSS not loading
- Styles look broken

**Root Cause**:
- Tailwind config incorrect
- Classes not in safelist
- PostCSS not configured
- Browser cache

**Solution**:

```bash
# Check Tailwind config
cat tailwind.config.ts

# Check PostCSS config
cat postcss.config.js

# Clear browser cache
# Or open in incognito mode

# Verify globals.css is imported
cat app/layout.tsx | grep "globals.css"

# Rebuild
rm -rf .next
bun run build
```

**Debug Classes**:

```tsx
// Check if class is being applied
<div className="bg-red-500 p-4">
  {/* Should have red background */}
</div>

// Check computed styles in DevTools
```

---

### 10. 404 Page for Valid Marketplace

**Symptoms**:
- Visit `/plugins/anthropics-claude-code`
- Get 404 "Page not found"
- Marketplace exists in JSON

**Root Cause**:
- Static params not generated
- Slug mismatch
- Build didn't complete

**Solution**:

```bash
# Check if marketplace slug matches URL
cat lib/data/marketplaces.json | grep "anthropics/claude-code" -A 5
# Look for: "slug": "anthropics-claude-code"

# Verify slug generation
bun run -e "console.log(require('./lib/utils/slug').repoToSlug('anthropics/claude-code'))"
# Should output: anthropics-claude-code

# Check static params generation
# In app/plugins/[slug]/page.tsx
export async function generateStaticParams() {
  const marketplaces = await getAllMarketplaces();
  console.log('Generating static params:', marketplaces.map(m => m.slug));
  return marketplaces.map(m => ({ slug: m.slug }));
}

# Rebuild
bun run build
```

---

## Debugging Techniques

### 1. Trace Data Flow

```bash
# 1. Check if marketplace exists
grep "yamadashy/repomix" lib/data/marketplaces.json

# 2. Check if plugins extracted
grep "yamadashy-repomix" lib/data/plugins.json

# 3. Check if static page generated
ls .next/server/app/plugins/yamadashy-repomix/

# 4. Check build logs
bun run build | grep "yamadashy"
```

### 2. Add Strategic Logging

```typescript
// In page components
export default async function Page({ params }) {
  console.log('Page params:', params);

  const data = await fetchData();
  console.log('Fetched data:', data.length);

  return <Component data={data} />;
}

// In Client Components
export function Component({ data }) {
  console.log('Component rendered with:', data.length);
  // ...
}
```

### 3. Use React DevTools

Install React DevTools browser extension:
- Inspect component tree
- Check props and state
- Track renders
- Find performance issues

### 4. Network Tab

Check browser DevTools → Network:
- See what files are loading
- Check for 404s
- Verify API calls (if any)
- Check file sizes

---

## Error Messages Reference

### "Cannot read properties of undefined"

```
TypeError: Cannot read properties of undefined (reading 'length')
```

**Cause**: Accessing property on undefined/null value

**Fix**:
```typescript
// ❌ Wrong
const count = plugins.length;

// ✅ Correct
const count = plugins?.length ?? 0;
```

### "Hydration failed"

```
Error: Hydration failed because the initial UI does not match what was rendered on the server
```

**Cause**: Server HTML doesn't match client render

**Fix**:
- Ensure Server and Client Components render identically
- Avoid using `Date.now()`, `Math.random()` in render
- Check for browser-only APIs in Server Components

```typescript
// ❌ Wrong: Different on server vs client
<div>{new Date().toLocaleString()}</div>

// ✅ Correct: Same on server and client
<div>{isoDateString}</div>
```

### "Module not found"

```
Module not found: Can't resolve '@/components/foo'
```

**Cause**: Import path incorrect or file doesn't exist

**Fix**:
```bash
# Check file exists
ls components/foo.tsx

# Check import
# Should use @/ alias for root imports
import { Foo } from "@/components/foo";
```

---

## Performance Issues

### Slow Page Loads

**Symptoms**: Pages take >2s to load

**Check**:
1. Build output size: `ls -lh .next/`
2. Number of static pages: `find .next/server/app -name "*.html" | wc -l`
3. JSON file sizes: `ls -lh lib/data/`

**Solutions**:
- Enable compression in production
- Optimize images with next/image
- Lazy load heavy components
- Use ISR for frequently updated pages

### High Memory Usage

**Symptoms**: Node runs out of memory during build

**Fix**:
```bash
# Increase memory limit
NODE_OPTIONS="--max-old-space-size=4096" bun run build

# Or in package.json
{
  "scripts": {
    "build": "NODE_OPTIONS='--max-old-space-size=4096' next build"
  }
}
```

---

## Getting More Help

If issues persist:

1. **Check documentation**: Review all files in `/docs`
2. **Search codebase**: Look for similar patterns
3. **Check Git history**: See how similar issues were solved
4. **Read Next.js docs**: https://nextjs.org/docs
5. **Check shadcn/ui**: https://ui.shadcn.com
6. **GitHub Issues**: Search for similar problems

## Reporting Issues

When reporting issues, include:

1. **Environment**:
   ```bash
   bun --version
   node --version
   cat package.json | grep "next"
   ```

2. **Steps to reproduce**
3. **Expected vs actual behavior**
4. **Error messages** (full stack trace)
5. **Screenshots** (if UI issue)
6. **Data samples** (JSON snippets, if relevant)

## Prevention Checklist

Before making changes:
- [ ] Read existing documentation
- [ ] Understand data flow
- [ ] Follow existing patterns
- [ ] Test locally before deploying
- [ ] Clear cache when in doubt
- [ ] Use verbose logging during development
- [ ] Check both dev and production builds
