## Complete Architecture & Production Specification

This blueprint details the complete end-to-end technical architecture for migrating from Ektron to **Optimizely CMS**, delivering content via **Optimizely Content Graph (GraphQL)**, and consuming it inside a high-performance **React** application.

---

## 1. End-to-End System Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   OPTIMIZELY CMS                                       │
│  ┌────────────────────────┐  ┌─────────────────────────┐  ┌──────────────────────────┐ │
│  │ NavigationConfigPage   │  │     MenuItemBlock       │  │    RedirectRuleBlock     │ │
│  │ (LOB, Header, Footer)  │  │ (Title, Url, Fallback)  │  │ (OldUrl, NewUrl, Status) │ │
│  └───────────┬────────────┘  └────────────┬────────────┘  └────────────┬─────────────┘ │
└──────────────┼────────────────────────────┼────────────────────────────┼───────────────┘
               │                            │                            │
               └────────────────────────────┼────────────────────────────┘
                                            ▼
                       ┌────────────────────────────────────────┐
                       │ Optimizely Content Graph Indexing Job  │
                       └────────────────────┬───────────────────┘
                                            │
                                            ▼
                       ┌────────────────────────────────────────┐
                       │  Optimizely Content Graph (Edge CDN)   │
                       │     https://cg.optimizely.com/v2       │
                       └────────────────────┬───────────────────┘
                                            │
                                            │ Single Unified GraphQL Query
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                    REACT SPA / SSR                                     │
│                                                                                        │
│  Hard Refresh / Boot (/benefit)                    Client Route Switch (/find-care)    │
│  ┌───────────────────────────┐                     ┌────────────────────────────────┐  │
│  │ Fetch Nav + Page + 301    │                     │ Fetch Page + 301 ONLY          │  │
│  │ (1 Request)               │                     │ (Re-use NavigationContext)     │  │
│  └─────────────┬─────────────┘                     └───────────────┬────────────────┘  │
│                │                                                   │                   │
│                ▼                                                   ▼                   │
│  ┌───────────────────────────┐                     ┌────────────────────────────────┐  │
│  │ Hydrate NavigationContext │                     │ Resolve Page / 301 / Fallback   │  │
│  │ & Render Header/Footer    │                     │ & Render Main View             │  │
│  └───────────────────────────┘                     └────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────────────┘

```

---

## 2. Optimizely CMS Content Model Design

Configure these models either via C# code or directly through **Optimizely CMS Admin Mode**:

### Model 1: `MenuItemBlock` (Block Type)

* **`Title`** *(String)*: Optional display title override.
* **`Url`** *(String)*: Optional custom URL override.
* **`TargetContent`** *(ContentReference)*: Link to primary target CMS page.
* **`FallbackMenuItem`** *(ContentReference)*: Restricted to `MenuItemBlock`. Fallback if `TargetContent` is missing/unpublished.
* **`SubItems`** *(ContentArea)*: Restricted to `MenuItemBlock` for multi-level nested sub-menus.

### Model 2: `NavigationConfigPage` (Page Type)

* **`LOB`** *(String)*: Line of Business identifier (e.g., `"BANKING"`, `"INSURANCE"`).
* **`HeaderMenu`** *(ContentArea)*: List of `MenuItemBlock` root items.
* **`FooterMenu`** *(ContentArea)*: List of `MenuItemBlock` root items.

### Model 3: `RedirectRuleBlock` (Block Type)

* **`OldUrl`** *(String)*: Source legacy path (e.g., `"/old-benefit-plan"`).
* **`NewUrl`** *(String)*: Target redirect path (e.g., `"/benefit"`).
* **`StatusCode`** *(Number)*: HTTP redirect status (`301` or `302`).
* **`IsActive`** *(Boolean)*: Enable/disable redirect rule.

---

## 3. Unified Master GraphQL Query

This query fetches **Redirect Rules**, **Global Navigation Trees (Header/Footer + Multi-level Sub-items + Fallbacks)**, the **Requested Target Page**, the **Default Fallback Page**, and the **CMS 404 Error Page** in **one single round-trip**.

```graphql
query GetInitialAppData(
  $lob: String!
  $pageUrl: String!
  $defaultPageUrl: String!
) {
  # --- 1. 301/302 REDIRECT CHECK ---
  redirectRules: RedirectRuleBlock(
    where: { OldUrl: { eq: $pageUrl }, IsActive: { eq: true } }
  ) {
    items {
      OldUrl
      NewUrl
      StatusCode
    }
  }

  # --- 2. GLOBAL NAVIGATION & NESTED MENU TREES ---
  navigationData: NavigationConfigPage(where: { lob: { eq: $lob } }) {
    items {
      lob
      headerMenu {
        ...MenuItemFields
        subItems {
          ...MenuItemFields
          subItems {
            ...MenuItemFields
          }
        }
      }
      footerMenu {
        ...MenuItemFields
        subItems {
          ...MenuItemFields
        }
      }
    }
  }

  # --- 3. TARGET REQUESTED PAGE CONTENT ---
  targetPage: _Page(where: { _metadata: { url: { eq: $pageUrl } } }) {
    items {
      _metadata { id url types }
      ...PageFields
    }
  }

  # --- 4. DEFAULT FALLBACK PAGE CONTENT ---
  fallbackPage: _Page(where: { _metadata: { url: { eq: $defaultPageUrl } } }) {
    items {
      _metadata { id url types }
      ...PageFields
    }
  }

  # --- 5. CMS 404 PAGE CONTENT ---
  notFoundPage: _Page(where: { _metadata: { url: { eq: "/404" } } }) {
    items {
      _metadata { id url types }
      ...PageFields
    }
  }
}

# --- REUSABLE FRAGMENTS ---
fragment MenuItemFields on MenuItemBlock {
  title
  url
  targetContent {
    _metadata { id url }
    ... on ArticlePage { headline }
    ... on LandingPage { heroTitle }
  }
  fallbackMenuItem {
    title
    url
    targetContent {
      _metadata { id url }
      ... on ArticlePage { headline }
    }
  }
}

fragment PageFields on _Page {
  ... on ArticlePage {
    headline
    mainBody
    author
  }
  ... on LandingPage {
    heroTitle
    bannerImage { url }
  }
}

```

---

## 4. React Application Architecture & Navigation Context

### Step A: Recursive Menu Tree & Fallback Resolver

`resolveMenuTree` flattens fallback items and processes nested sub-menus recursively down the tree:

```typescript
// utils/menuResolver.ts
export interface ResolvedMenuItem {
  title: string;
  url: string;
  children: ResolvedMenuItem[];
}

export function resolveMenuTree(rawItems: any[] = []): ResolvedMenuItem[] {
  return rawItems
    .map((item) => {
      // 1. Check primary target content or URL; if missing, resolve fallback item
      const active = (item.targetContent || item.url) ? item : item.fallbackMenuItem;
      if (!active) return null;

      const title = active.title || active.targetContent?.headline || active.targetContent?.heroTitle || "Menu Link";
      const url = active.url || active.targetContent?._metadata?.url || "#";
      
      // 2. Resolve sub-menu items recursively
      const children = item.subItems ? resolveMenuTree(item.subItems) : [];

      return { title, url, children };
    })
    .filter((node): node is ResolvedMenuItem => node !== null);
}

```

### Step B: React Navigation Context Provider

```tsx
// context/NavigationContext.tsx
import React, { createContext, useContext, useState } from 'react';
import { resolveMenuTree, ResolvedMenuItem } from '../utils/menuResolver';

interface NavContextType {
  headerMenu: ResolvedMenuItem[];
  footerMenu: ResolvedMenuItem[];
  isNavLoaded: boolean;
  setNavigation: (rawHeader: any[], rawFooter: any[]) => void;
}

const NavigationContext = createContext<NavContextType>({
  headerMenu: [],
  footerMenu: [],
  isNavLoaded: false,
  setNavigation: () => {}
});

export const NavigationProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [headerMenu, setHeaderMenu] = useState<ResolvedMenuItem[]>([]);
  const [footerMenu, setFooterMenu] = useState<ResolvedMenuItem[]>([]);
  const [isNavLoaded, setIsNavLoaded] = useState(false);

  const setNavigation = (rawHeader: any[] = [], rawFooter: any[] = []) => {
    setHeaderMenu(resolveMenuTree(rawHeader));
    setFooterMenu(resolveMenuTree(rawFooter));
    setIsNavLoaded(true);
  };

  return (
    <NavigationContext.Provider value={{ headerMenu, footerMenu, isNavLoaded, setNavigation }}>
      {children}
    </NavigationContext.Provider>
  );
};

export const useNavigation = () => useContext(NavigationContext);

```

### Step C: Route-Aware Router Shell (`AppShell.tsx`)

This wrapper handles hard refreshes on `/benefit` or `/find-care`, intercepts redirects, loads page content, and falls back gracefully:

```tsx
// components/AppShell.tsx
import React, { useEffect, useState } from 'react';
import { useLocation, useNavigate } from 'react-router-dom';
import { useNavigation } from '../context/NavigationContext';

export const AppShell: React.FC = () => {
  const location = useLocation();
  const navigate = useNavigate();
  const { isNavLoaded, setNavigation, headerMenu, footerMenu } = useNavigation();
  
  const [pageData, setPageData] = useState<any>(null);
  const [isLoading, setIsLoading] = useState(true);

  const LOB = "BANKING";
  const DEFAULT_PAGE = "/dashboard";

  useEffect(() => {
    async function loadRoute() {
      setIsLoading(true);
      
      // Get current path dynamically (handles hard refresh on /benefit, /find-care)
      const currentPath = location.pathname === "/" ? DEFAULT_PAGE : location.pathname;

      if (!isNavLoaded) {
        // --- 1. INITIAL BOOT / HARD REFRESH ---
        const response = await fetchOptimizelyGraph(GET_INITIAL_APP_DATA, {
          lob: LOB,
          pageUrl: currentPath,
          defaultPageUrl: DEFAULT_PAGE
        });

        const data = response.data;

        // A. Handle 301/302 Redirect
        const redirect = data?.redirectRules?.items?.[0];
        if (redirect) {
          navigate(redirect.NewUrl, { replace: true });
          return;
        }

        // B. Populate Global Navigation Context
        const nav = data?.navigationData?.items?.[0];
        if (nav) {
          setNavigation(nav.headerMenu, nav.footerMenu);
        }

        // C. Resolve Target Page / Fallback Page / 404 Page
        const page = data?.targetPage?.items?.[0] 
          || data?.fallbackPage?.items?.[0] 
          || data?.notFoundPage?.items?.[0];
          
        setPageData(page);

      } else {
        // --- 2. CLIENT-SIDE ROUTE TRANSITION ---
        // Navigation is already cached. Query page content + redirect only.
        const response = await fetchOptimizelyGraph(GET_PAGE_BY_URL, {
          pageUrl: currentPath,
          defaultPageUrl: DEFAULT_PAGE
        });

        const data = response.data;

        // A. Handle Redirect
        const redirect = data?.redirectRules?.items?.[0];
        if (redirect) {
          navigate(redirect.NewUrl, { replace: true });
          return;
        }

        // B. Resolve Page Content
        const page = data?.targetPage?.items?.[0] 
          || data?.fallbackPage?.items?.[0] 
          || data?.notFoundPage?.items?.[0];
          
        setPageData(page);
      }

      setIsLoading(false);
    }

    loadRoute();
  }, [location.pathname]);

  if (isLoading) return <div className="spinner">Loading...</div>;

  return (
    <div className="app-layout">
      <Header menu={headerMenu} />
      <main className="main-content">
        {pageData ? <RenderCMSPage data={pageData} /> : <Static404View />}
      </main>
      <Footer menu={footerMenu} />
    </div>
  );
};

```

---

## 5. Execution Pipeline & Fallback Resolution Logic

| Trigger Condition | Request Type | Resolution Priority Pipeline | Resulting Output |
| --- | --- | --- | --- |
| **Initial Boot / Hard Refresh** (`/benefit`) | `GetInitialAppData` *(1 Query)* | 1. Check `redirectRules`<br>

<br>2. Hydrate `NavigationContext`<br>

<br>3. Check `targetPage` (`/benefit`) | Header/Footer rendered from Context + `/benefit` page view. |
| **Route Switch** (`/find-care`) | `GetPageByUrl` *(1 Query)* | 1. Check `redirectRules`<br>

<br>2. Re-use cached `NavigationContext`<br>

<br>3. Check `targetPage` (`/find-care`) | Header/Footer preserved in layout + `/find-care` view. |
| **Missing Target Page** (`/invalid-path`) | `GetPageByUrl` or `GetInitialAppData` | 1. Check `targetPage` $\rightarrow$ `null`<br>

<br>2. Check `fallbackPage` $\rightarrow$ `Found` | Header/Footer preserved + `/dashboard` content view. |
| **Missing Target & Fallback** | `GetPageByUrl` or `GetInitialAppData` | 1. `targetPage` $\rightarrow$ `null`<br>

<br>2. `fallbackPage` $\rightarrow$ `null`<br>

<br>3. Check `notFoundPage` $\rightarrow$ `Found` | Header/Footer preserved + CMS `/404` content view + Telemetry logged. |
| **Legacy URL Hit** (`/old-benefit`) | `GetPageByUrl` or `GetInitialAppData` | 1. Check `redirectRules` $\rightarrow$ Match (`/benefit`) | Instant browser redirect to `/benefit` (No 404 flash). |
