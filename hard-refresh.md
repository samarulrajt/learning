When a user performs a hard refresh (or enters a URL directly) while on `/benefit` or `/find-care`, the browser resets the in-memory React state and Context.

To handle this seamlessly without breaking the layout or making multiple network calls, your application dynamically passes the **current browser route** (`window.location.pathname`) as the `$pageUrl` parameter to the initial boot GraphQL query.

---

### How the Hard Refresh Execution Flow Works

1. **Browser Request:** User hits hard refresh on `[https://your-app.com/benefit](https://your-app.com/benefit)`.
2. **React App Mounts:** In-memory `NavigationContext` is empty. The initial boot state is `true`.
3. **Dynamic URL Extraction:** The app inspects `location.pathname` (which returns `"/benefit"`).
4. **Single GraphQL Request:** Executed with `$pageUrl: "/benefit"` and `$defaultPageUrl: "/dashboard"`.
5. **Simultaneous Payload Delivery:**
* **Global Navigation:** Populates `headerMenu`, `footerMenu`, and sub-item trees into `NavigationContext`.
* **Page Content:** Returns the content for `/benefit`. If `/benefit` does not exist or is unpublished, it falls back to `/dashboard`.



---

### React Implementation for Route-Aware Booting

Update the boot logic in your main layout or router shell so that `$pageUrl` is never hardcoded:

```tsx
import React, { useEffect, useState } from 'react';
import { useLocation } from 'react-router-dom';
import { useNavigation } from './NavigationContext';

export const AppShell: React.FC = () => {
  const location = useLocation(); // Extracts current path: "/benefit", "/find-care", etc.
  const { isNavLoaded, setNavigation } = useNavigation();
  const [pageData, setPageData] = useState<any>(null);
  const [isLoading, setIsLoading] = useState(true);

  const LOB = "BANKING";
  const FALLBACK_PAGE = "/dashboard";

  useEffect(() => {
    async function handleNavigation() {
      setIsLoading(true);
      
      // Determine target page URL dynamically from browser location
      const currentPath = location.pathname === "/" ? FALLBACK_PAGE : location.pathname;

      if (!isNavLoaded) {
        // --- HARD REFRESH / INITIAL ENTRY POINT ---
        // Single query fetches Navigation + Page Content for current path (/benefit)
        const response = await fetchOptimizelyGraph(GET_INITIAL_APP_DATA, {
          lob: LOB,
          pageUrl: currentPath,          // Dynamic: "/benefit" or "/find-care"
          defaultPageUrl: FALLBACK_PAGE   // Fallback if targeted page doesn't exist
        });

        // 1. Hydrate Global Navigation Context
        const nav = response.data.navigationData?.items[0];
        if (nav) {
          setNavigation(nav.headerMenu, nav.footerMenu);
        }

        // 2. Set Page Content (Primary requested page or fallback)
        const page = response.data.targetPage?.items[0] || response.data.fallbackPage?.items[0];
        setPageData(page);

      } else {
        // --- CLIENT-SIDE ROUTE TRANSITION ---
        // Navigation is already cached in Context. Fetch ONLY the page content.
        const response = await fetchOptimizelyGraph(GET_PAGE_BY_URL, {
          pageUrl: currentPath,
          defaultPageUrl: FALLBACK_PAGE
        });

        const page = response.data.targetPage?.items[0] || response.data.fallbackPage?.items[0];
        setPageData(page);
      }

      setIsLoading(false);
    }

    handleNavigation();
  }, [location.pathname]);

  if (isLoading) return <AppSkeletonLoader />;

  return (
    <Layout>
      <MainContent content={pageData} />
    </Layout>
  );
};

```

---

### Important Web Server Setup (Single Page App Requirement)

If you are running a purely client-side React app (SPA using Vite, Create React App, etc.), ensure your hosting server (Nginx, AWS S3/CloudFront, Azure Static Web Apps, or Netlify) has **URL rewrite rules** configured.

Without rewrites, a hard refresh on `/benefit` will result in a server **404 Not Found** before React even loads.

* **Nginx Configuration Example:**
```nginx
location / {
  try_files $uri $uri/ /index.html;
}

```


* **Netlify / Vercel (`_redirects` or `vercel.json`):**
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}

```



If you are using **Next.js** or **Remix**, SSR/SSG naturally routes requests to the server layout component, rendering both the navigation and page server-side before sending HTML to the client.
