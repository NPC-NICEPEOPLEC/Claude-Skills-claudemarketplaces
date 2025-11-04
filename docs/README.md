# Claude Code Marketplaces Documentation

**TL;DR**: This is a Next.js 15 application that discovers, indexes, and displays Claude Code plugin marketplaces from GitHub. It uses automated GitHub search to find marketplaces, validates them, extracts plugins, and presents them in a searchable web interface.

## Quick Links

- [Architecture Overview](./architecture.md) - System design and technology stack
- [Data Flow](./data-flow.md) - How data moves through the system
- [Components](./components.md) - Frontend component structure
- [Development Guide](./development-guide.md) - How to add features
- [Troubleshooting](./troubleshooting.md) - Common issues and solutions
- [API Reference](./api-reference.md) - Key functions and types

## Project Overview

### What It Does

1. **Discovers** Claude Code marketplaces by searching GitHub for `.claude-plugin/marketplace.json` files
2. **Validates** marketplace files against a Zod schema
3. **Extracts** plugin metadata from each marketplace
4. **Stores** marketplace and plugin data in JSON files (with Vercel Blob backup)
5. **Displays** searchable, filterable marketplace and plugin listings

### Key Features

- Automated GitHub marketplace discovery
- Schema validation using Zod
- Plugin extraction and indexing
- Static site generation for all marketplace pages
- Search and filter functionality
- Responsive UI with shadcn/ui components
- SEO optimization with metadata and structured data

## Quick Start

### Running the Development Server

```bash
bun dev
```

Access at http://localhost:3000

### Updating Marketplace Data

```bash
bun run scripts/search.ts
```

This discovers new marketplaces, extracts plugins, and updates the data files.

### Building for Production

```bash
bun run build
```

## Directory Structure

```
.
├── app/                    # Next.js 15 App Router pages
│   ├── page.tsx           # Home page (marketplace grid)
│   └── plugins/[slug]/    # Dynamic plugin pages
├── components/            # React components
│   ├── ui/               # shadcn/ui primitives
│   ├── marketplace-*.tsx # Marketplace-specific components
│   └── plugin-*.tsx      # Plugin-specific components
├── lib/
│   ├── data/             # Data access layer
│   │   ├── marketplaces.json  # Marketplace database
│   │   ├── plugins.json       # Plugin database
│   │   ├── marketplaces.ts    # Marketplace queries
│   │   └── plugins.ts         # Plugin queries
│   ├── search/           # GitHub search and validation
│   │   ├── github-search.ts   # GitHub API integration
│   │   ├── validator.ts       # Marketplace validation
│   │   ├── plugin-extractor.ts # Plugin extraction logic
│   │   └── storage.ts         # Data persistence
│   ├── schemas/          # Zod validation schemas
│   ├── hooks/            # React hooks
│   └── utils/            # Utility functions
├── scripts/
│   └── search.ts         # Marketplace discovery script
└── docs/                 # Documentation (you are here)
```

## Data Storage

### Development
- Data stored in `lib/data/marketplaces.json` and `lib/data/plugins.json`
- Files are read directly from disk

### Production
- Primary: Vercel Blob storage (publicly accessible)
- Fallback: Local JSON files (read-only filesystem)
- Both are kept in sync by the search script

## Technology Stack

- **Framework**: Next.js 15 with App Router
- **Runtime**: React 19
- **Language**: TypeScript 5
- **Package Manager**: Bun
- **Styling**: Tailwind CSS v4
- **UI Components**: shadcn/ui (Radix primitives)
- **Validation**: Zod
- **Storage**: Vercel Blob + Local JSON
- **Build Tool**: Turbopack

## Common Tasks

### Adding a New Feature
See [Development Guide](./development-guide.md)

### Debugging Plugin Issues
See [Troubleshooting](./troubleshooting.md)

### Understanding Data Flow
See [Data Flow](./data-flow.md)

### Component Hierarchy
See [Components](./components.md)

## Getting Help

1. Check [Troubleshooting](./troubleshooting.md) for common issues
2. Review [Architecture](./architecture.md) to understand system design
3. Consult [API Reference](./api-reference.md) for function signatures
4. Read relevant section in [Development Guide](./development-guide.md)
