# Food Shop Price Comparison App

## Context

Build a Next.js full-stack app where a user types a food item into a search box and the backend scrapes Tesco, Sainsbury's, Asda, Morrisons, and Aldi simultaneously using Playwright, returning results sorted cheapest-first. CI via GitHub Actions (lint, format-check, typecheck, test, build); deployed to Vercel using `@sparticuz/chromium`.

## Stack

- **Next.js 15** (App Router, TypeScript)
- **Tailwind CSS v4**
- **playwright-core** + **@sparticuz/chromium** (Vercel-compatible headless browser)
- **Redux Toolkit** + **RTK Query** for state management and API calls
- **Vitest** + **@testing-library/react** + **fishery** for integration-style unit tests
- **ESLint** (Next.js, React, React Hooks, JSX a11y, TypeScript, Prettier plugins)
- **Prettier** for formatting
- **GitHub Actions** for CI
- **Vercel** (GitHub App) for deployment

---

## Project Structure

```
food-shop-scrapper/
├── package.json
├── next.config.ts                  ← serverExternalPackages: ['playwright-core', '@sparticuz/chromium']
├── tsconfig.json
├── vitest.config.ts                ← Vitest config with jsdom + path aliases
├── vitest.setup.ts                 ← @testing-library/jest-dom setup
├── eslint.config.mjs               ← flat config: next, react, react-hooks, a11y, ts, prettier
├── .prettierrc                     ← formatting rules
├── .prettierignore
├── vercel.json                     ← maxDuration: 60, memory: 3008
├── .env.local                      ← PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=true
├── .gitignore
├── .github/
│   └── workflows/
│       └── ci.yml                  ← lint → format:check → typecheck → test → build
├── app/
│   ├── layout.tsx
│   ├── page.tsx                    ← Client UI: search form + results
│   ├── globals.css
│   └── api/search/route.ts         ← POST handler, maxDuration = 60
├── lib/
│   ├── scrapers/
│   │   ├── types.ts                ← ProductResult, ScraperOutcome, SearchResponse
│   │   ├── base.ts                 ← withBrowserPage() — prod vs dev Chromium path
│   │   ├── index.ts                ← runAllScrapers() with Promise.all
│   │   ├── tesco.ts
│   │   ├── sainsburys.ts
│   │   ├── asda.ts
│   │   ├── morrisons.ts
│   │   └── aldi.ts
│   ├── testing/
│   │   └── factories.ts            ← fishery factories for ProductResult, ScraperOutcome, SearchResponse
│   └── utils/
│       └── parsePrice.ts           ← "£1.50" → 1.50
├── store/
│   ├── index.ts                    ← configureStore, export RootState + AppDispatch
│   ├── provider.tsx                ← 'use client' Redux Provider wrapper
│   └── searchApi.ts                ← RTK Query createApi — POST /api/search endpoint
├── components/
│   ├── SearchForm.tsx
│   ├── ResultsTable.tsx            ← sorted table, cheapest highlighted green
│   ├── LoadingSpinner.tsx
│   └── ErrorBanner.tsx
└── __tests__/
    ├── utils/
    │   └── parsePrice.test.ts      ← pure function, no factories needed
    ├── scrapers/
    │   └── index.test.ts           ← runAllScrapers error isolation (spies on scrapers)
    ├── api/
    │   └── search.test.ts          ← API route input validation + response shape
    └── components/
        ├── SearchForm.test.tsx
        └── ResultsTable.test.tsx   ← renders factory-built results correctly
```

---

## Key Dependencies

```json
"dependencies": {
  "next": "15.2.2",
  "react": "^19.0.0",
  "react-dom": "^19.0.0",
  "playwright-core": "^1.51.0",
  "@sparticuz/chromium": "^141.0.0",
  "@reduxjs/toolkit": "^2",
  "react-redux": "^9"
},
"devDependencies": {
  "typescript": "^5",
  "tailwindcss": "^4",
  "@tailwindcss/postcss": "^4",

  "vitest": "^3",
  "@vitejs/plugin-react": "^4",
  "@testing-library/react": "^16",
  "@testing-library/user-event": "^14",
  "@testing-library/jest-dom": "^6",
  "jsdom": "^26",
  "fishery": "^2",

  "eslint": "^9",
  "eslint-config-next": "15.2.2",
  "eslint-plugin-react": "^7",
  "eslint-plugin-react-hooks": "^5",
  "eslint-plugin-jsx-a11y": "^6",
  "@typescript-eslint/eslint-plugin": "^8",
  "@typescript-eslint/parser": "^8",
  "eslint-config-prettier": "^10",
  "eslint-plugin-prettier": "^5",
  "prettier": "^3"
},
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "lint": "next lint",
  "lint:fix": "next lint --fix",
  "format": "prettier --write .",
  "format:check": "prettier --check .",
  "typecheck": "tsc --noEmit",
  "test": "vitest run",
  "test:watch": "vitest"
}
```

> **No `postinstall` browser download** — `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=true` prevents playwright from downloading browsers at install time. Dev uses local Chromium (`npx playwright install chromium` once); prod uses `@sparticuz/chromium`.

---

## Scraper Selectors (confirmed from live DOM)

| Supermarket | Search URL                                                    | Card                                    | Name                                           | Price                                                                |
| ----------- | ------------------------------------------------------------- | --------------------------------------- | ---------------------------------------------- | -------------------------------------------------------------------- |
| Tesco       | `/groceries/en-GB/search?query=`                              | `li[data-testid][data-auto-available]`  | `a.gyT8MW_titleLink`                           | `p.gyT8MW_priceText`                                                 |
| Sainsbury's | Navigate to `/gol-ui/SearchResults/`, fill input, press Enter | `article[data-testid^="product-tile-"]` | `h2[data-testid="product-tile-description"] a` | `span.pt__cost--price`                                               |
| Asda        | `/groceries/search/{query}`                                   | `div.product-module`                    | `a[data-locator="txt-product-name"]`           | first element matching `/^£\d/`                                      |
| Morrisons   | `/search?entry=` (may redirect — fall back to search input)   | `div.product-card-container`            | `a[data-test="fop-product-link"] span.salt-vc` | regex `£[\d.]+` from `.price-pack-size-container`                    |
| Aldi        | `/results?q=` (not `/search`)                                 | `div[data-mn^="product-category-item"]` | `div[data-test="product-tile__name"] p`        | `div[data-test="product-tile__price"] span.base-price__regular span` |

---

## Critical Implementation Notes

1. **`next.config.ts`** must include `serverExternalPackages: ['playwright-core', '@sparticuz/chromium']` — prevents Next.js from bundling native binaries.

2. **`base.ts`** — prod vs dev Chromium detection:

   ```typescript
   import chromium from '@sparticuz/chromium'
   import { chromium as playwright } from 'playwright-core'

   const isProduction = process.env.NODE_ENV === 'production'

   export async function withBrowserPage<T>(fn: (page: Page) => Promise<T>): Promise<T> {
     const browser = await playwright.launch({
       args: isProduction
         ? chromium.args
         : [
             '--no-sandbox',
             '--disable-setuid-sandbox',
             '--disable-blink-features=AutomationControlled',
           ],
       executablePath: isProduction ? await chromium.executablePath() : undefined,
       headless: true,
     })
     // ... try/finally close
   }
   ```

3. **Sainsbury's** cannot use a direct URL — must navigate to base search page and use the search input (SPA routing).

4. **Morrisons** may redirect anon users — scraper detects URL change and falls back to filling the search input.

5. **Aldi** search URL is `/results?q=`, not `/search?q=`.

6. **Each scraper** launches its own browser instance, wrapped in try/catch in `runAllScrapers` so one failure doesn't block others.

7. **`maxDuration = 60`** set on the API route file AND in `vercel.json`.

---

## Redux / RTK Query

### Store (`store/index.ts`)

```typescript
import { configureStore } from '@reduxjs/toolkit'
import { searchApi } from './searchApi'

export const store = configureStore({
  reducer: {
    [searchApi.reducerPath]: searchApi.reducer,
  },
  middleware: (getDefaultMiddleware) => getDefaultMiddleware().concat(searchApi.middleware),
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
```

### RTK Query API slice (`store/searchApi.ts`)

```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react'
import type { SearchResponse } from '@/lib/scrapers/types'

export const searchApi = createApi({
  reducerPath: 'searchApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    searchProducts: builder.mutation<SearchResponse, string>({
      query: (query) => ({
        url: '/search',
        method: 'POST',
        body: { query },
      }),
    }),
  }),
})

export const { useSearchProductsMutation } = searchApi
```

### Provider (`store/provider.tsx`)

```typescript
'use client'
import { Provider } from 'react-redux'
import { store } from './index'

export function ReduxProvider({ children }: { children: React.ReactNode }) {
  return <Provider store={store}>{children}</Provider>
}
```

Wrap `app/layout.tsx` body with `<ReduxProvider>`.

### Usage in `page.tsx`

Replace manual `fetch` + `useState` with:

```typescript
const [searchProducts, { data, isLoading, error }] = useSearchProductsMutation()
// call searchProducts(query) on form submit
// data is SearchResponse | undefined
// isLoading / error come from RTK Query — no manual state needed
```

### What RTK Query manages automatically

- Loading state (`isLoading`)
- Error state (`isError`, `error`)
- Result caching per query string
- Request deduplication

### Test factories addition

Add `store/searchApi` factory helpers in `lib/testing/factories.ts` for mocking RTK Query state in component tests using `setupStore()` + a pre-loaded state.

---

## Testing Strategy

### Philosophy

- **Integration-style** — tests exercise real logic paths end-to-end within a layer, not isolated stubs
- **Factories over mocks** — use fishery to build realistic `ProductResult` / `ScraperOutcome` objects; avoid `vi.mock()` where the real function can be used or spied upon
- **Scrapers are NOT unit tested** (they require a live browser) — the scraping layer is covered by manual/e2e testing

### Factories (`lib/testing/factories.ts`)

```typescript
import { Factory } from 'fishery'
import type { ProductResult, ScraperOutcome, SearchResponse } from '../scrapers/types'

export const productResultFactory = Factory.define<ProductResult>(({ sequence }) => ({
  supermarket: 'Tesco',
  name: `Product ${sequence}`,
  price: 1.0 + sequence * 0.5,
  priceDisplay: `£${(1.0 + sequence * 0.5).toFixed(2)}`,
  url: `https://www.tesco.com/groceries/en-GB/products/${sequence}`,
}))

export const scraperOutcomeFactory = Factory.define<ScraperOutcome>(({ sequence }) => ({
  supermarket: 'Tesco',
  results: productResultFactory.buildList(3),
  durationMs: 1000 + sequence * 100,
}))

export const searchResponseFactory = Factory.define<SearchResponse>(() => ({
  query: 'milk',
  outcomes: [
    scraperOutcomeFactory.build({ supermarket: 'Tesco' }),
    scraperOutcomeFactory.build({ supermarket: 'Asda' }),
  ],
  sortedResults: productResultFactory.buildList(6),
  totalDurationMs: 3500,
}))
```

### Test Coverage

| File                    | What it tests                                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------------- |
| `parsePrice.test.ts`    | All price string formats: `£1.50`, `£1`, `1.50`, null, garbage → correct number or null             |
| `index.test.ts`         | `runAllScrapers` isolates failures — if one scraper throws, other 4 still return results            |
| `search.test.ts`        | API route: rejects short queries, rejects too-long queries, calls `runAllScrapers` and returns JSON |
| `SearchForm.test.tsx`   | Renders input, disables button when loading, calls `onSearch` with trimmed value on submit          |
| `ResultsTable.test.tsx` | Renders factory results sorted correctly, marks first row as "Cheapest", links open to correct URLs |
| `searchApi.test.ts`     | RTK Query endpoint — correct URL, method, body; maps response to `SearchResponse` type              |

### `vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import { resolve } from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
    globals: true,
  },
  resolve: {
    alias: { '@': resolve(__dirname, '.') },
  },
})
```

---

## Linting & Formatting

### ESLint (`eslint.config.mjs` — flat config)

Extends `eslint-config-next` (which bundles react, react-hooks, jsx-a11y, @next/next) plus:

- `@typescript-eslint` recommended rules
- `eslint-plugin-prettier` — runs Prettier as an ESLint rule, so `lint` catches format issues too
- `eslint-config-prettier` — disables ESLint rules that conflict with Prettier (must be last)

### Prettier (`.prettierrc`)

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2
}
```

---

## Config Files

### `vercel.json`

```json
{
  "functions": {
    "app/api/search/route.ts": {
      "maxDuration": 60,
      "memory": 3008
    }
  }
}
```

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
        env:
          PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD: true
      - run: npm run lint
      - run: npm run format:check
      - run: npm run typecheck
      - run: npm run test
      - run: npm run build
```

### `.env.local`

```
PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=true
```

### Vercel Deployment

Use **Vercel's GitHub App** (connect repo via Vercel dashboard) for automatic production deploys on `main` and PR preview deployments. Set `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=true` in Vercel project environment variables.

---

## Setup Commands (Bootstrap)

```bash
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir --eslint
npm install playwright-core @sparticuz/chromium @reduxjs/toolkit react-redux
npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom fishery
npm install -D eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-jsx-a11y @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint-config-prettier eslint-plugin-prettier prettier
npx playwright install chromium   # local dev only, one-time
```

---

## Verification

1. **Local:** `npm run dev` → `http://localhost:3000` → search "semi-skimmed milk" → results within ~30s
2. **Lint:** `npm run lint` — zero errors
3. **Format:** `npm run format:check` — passes
4. **Types:** `npm run typecheck` — zero errors
5. **Tests:** `npm run test` — all tests green
6. **Build:** `npm run build` — succeeds
7. **CI:** Push to GitHub → Actions shows all 5 steps passing
8. **Vercel:** Deploy triggered automatically; scraping works on live URL
