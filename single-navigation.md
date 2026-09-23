To fetch both a **specific page's content** and the **global application navigation (Header & Footer menus)** in a **single GraphQL query**, you can take advantage of GraphQL's ability to execute multiple root-level queries in a single request.

In Optimizely Content Graph, you can combine a polymorphic page query (using `_Page` or `_Content`) with your `NavigationConfigPage` query using GraphQL aliases.

---

### 1. Combined GraphQL Query (Single Page + Global Menus)

Pass the target page URL (or ID) along with the `LOB` into one query:

```graphql
query GetPageAndNavigation($pageUrl: String!,$lob: String!) {
  
  # --- ROOT QUERY 1: Fetch Specific Page Content ---
  pageData: _Page(
    where: {
      _metadata: { url: { eq: $pageUrl } }
    }
  ) {
    items {
      _metadata {
        id
        displayName
        url
        types
      }
      # Dynamic inline fragments based on your CMS Page Types
      ... on ArticlePage {
        headline
        author
        mainBody
        heroImage {
          url
        }
      }
      ... on LandingPage {
        heroTitle
        bannerImage {
          url
        }
      }
    }
  }

  # --- ROOT QUERY 2: Fetch Application Navigation (Header/Footer) ---
  navigationData: NavigationConfigPage(
    where: {
      lob: { eq: $lob }
    }
  ) {
    items {
      lob
      
      # Header Menu Items & Linked Target Content
      headerMenu {
        title
        url
        targetContent {
          _metadata {
            id
            url
          }
          ... on ArticlePage {
            headline
          }
        }
        fallbackMenuItem {
          title
          url
          targetContent {
            _metadata {
              url
            }
          }
        }
      }

      # Footer Menu Items & Linked Target Content
      footerMenu {
        title
        url
        targetContent {
          _metadata {
            id
            url
          }
        }
        fallbackMenuItem {
          title
          url
        }
      }
    }
  }
}

```

---

### 2. Combined Single Response Payload

The response delivers both the specific page data and the layout navigation objects in one JSON object:

```json
{
  "data": {
    "pageData": {
      "items": [
        {
          "_metadata": {
            "id": "2048",
            "displayName": "Enterprise Security Overview",
            "url": "/en/security/overview",
            "types": ["ArticlePage"]
          },
          "headline": "Enterprise Security Overview",
          "author": "Security Team",
          "mainBody": "<p>Detailed security protocols for enterprise customers...</p>",
          "heroImage": {
            "url": "/globalassets/images/security-hero.jpg"
          }
        }
      ]
    },
    "navigationData": {
      "items": [
        {
          "lob": "BANKING",
          "headerMenu": [
            {
              "title": "Products",
              "url": "/products",
              "targetContent": null,
              "fallbackMenuItem": {
                "title": "Default Services",
                "url": "/en/default-services",
                "targetContent": {
                  "_metadata": {
                    "url": "/en/default-services"
                  }
                }
              }
            }
          ],
          "footerMenu": [
            {
              "title": "Terms & Conditions",
              "url": "/terms",
              "targetContent": {
                "_metadata": {
                  "id": "901",
                  "url": "/en/terms"
                }
              },
              "fallbackMenuItem": null
            }
          ]
        }
      ]
    }
  }
}

```

---

### 3. How to Consume This in Next.js / React / Frontend App

By querying both root fields together, your application wrapper renders the entire page (Layout + Body) in one render pass:

```typescript
export async function getStaticProps({ params }) {
  const pageUrl = `/en/${params.slug}`;
  const lob = "BANKING"; // Determined by domain, path, or environment

  const { data } = await optimizelyGraphClient.query({
    query: GET_PAGE_AND_NAVIGATION,
    variables: { pageUrl, lob }
  });

  return {
    props: {
      currentPage: data.pageData.items[0] || null,
      navigation: data.navigationData.items[0] || null
    }
  };
}

export default function Page({ currentPage, navigation }) {
  if (!currentPage) return <NotFound />;

  return (
    <Layout navigation={navigation}>
      {/* Page Specific Body Content */}
      <h1>{currentPage.headline || currentPage.heroTitle}</h1>
      <div dangerouslySetInnerHTML={{ __html: currentPage.mainBody }} />
    </Layout>
  );
}

```


To support multi-level nested sub-menus with fallbacks, update your Optimizely CMS `MenuItemBlock` content model to include a self-referential children property, then leverage GraphQL fragments nested to your application's maximum menu depth.

---

### 1. Updated Optimizely CMS Content Model

Add a recursive `subItems` property to your `MenuItemBlock` definition:

```csharp
[ContentType(GUID = "11111111-2222-3333-4444-555555555555", DisplayName = "Menu Item Block")]
public class MenuItemBlock : BlockData
{
    public virtual string Title { get; set; }
    public virtual string Url { get; set; }
    
    public virtual ContentReference TargetContent { get; set; }
    
    // Fallback if TargetContent is unlinked/unpublished
    public virtual ContentReference FallbackMenuItem { get; set; }

    // Multi-level nesting link (ContentArea or List of MenuItemBlock)
    public virtual ContentArea SubItems { get; set; }
}

```

---

### 2. Nested GraphQL Query Using Fragments

Because standard GraphQL requires explicit field definitions for nested levels, use a reusable fragment expanded to your target maximum depth (e.g., 3 levels):

```graphql
# --- Base Fragment for Single Menu Item ---
fragment MenuItemFields on MenuItemBlock {
  title
  url
  targetContent {
    _metadata {
      id
      url
      types
    }
    ... on ArticlePage {
      headline
    }
    ... on LandingPage {
      heroTitle
    }
  }
  fallbackMenuItem {
    title
    url
    targetContent {
      _metadata {
        url
      }
      ... on ArticlePage {
        headline
      }
    }
  }
}

# --- Combined Query with Multi-Level Sub-menus ---
query GetPageAndNestedNavigation($pageUrl: String!, $lob: String!) {
  pageData: _Page(where: { _metadata: { url: { eq: $pageUrl } } }) {
    items {
      _metadata { id url }
      ... on ArticlePage { headline mainBody }
    }
  }

  navigationData: NavigationConfigPage(where: { lob: { eq: $lob } }) {
    items {
      lob
      headerMenu {
        ...MenuItemFields
        
        # Level 2 Sub-items
        subItems {
          ...MenuItemFields
          
          # Level 3 Sub-items
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
}

```

---

### 3. Sample GraphQL JSON Response

```json
{
  "data": {
    "navigationData": {
      "items": [
        {
          "lob": "BANKING",
          "headerMenu": [
            {
              "title": "Loans",
              "url": null,
              "targetContent": null,
              "fallbackMenuItem": null,
              "subItems": [
                {
                  "title": "Home Loans (Unpublished)",
                  "url": null,
                  "targetContent": null,
                  "fallbackMenuItem": {
                    "title": "General Mortgages",
                    "url": "/mortgages/default",
                    "targetContent": {
                      "_metadata": { "url": "/mortgages/default" },
                      "headline": "Standard Mortgage Plans"
                    }
                  },
                  "subItems": [
                    {
                      "title": "Fixed Rate Calculator",
                      "url": "/calculators/fixed-rate",
                      "targetContent": {
                        "_metadata": { "url": "/calculators/fixed-rate" }
                      },
                      "fallbackMenuItem": null,
                      "subItems": []
                    }
                  ]
                }
              ]
            }
          ]
        }
      ]
    }
  }
}

```

---

### 4. Recursive Client-Side Tree Resolver

In your frontend application (e.g., React/Next.js), parse the multi-level structure recursively to flatten fallbacks and build the navigation tree:

```typescript
interface MenuItemNode {
  title?: string;
  url?: string;
  targetContent?: Record<string, any> | null;
  fallbackMenuItem?: MenuItemNode | null;
  subItems?: MenuItemNode[];
}

interface ResolvedMenuItem {
  title: string;
  url: string;
  children: ResolvedMenuItem[];
}

function resolveMenuTree(nodes: MenuItemNode[] = []): ResolvedMenuItem[] {
  return nodes
    .map((item) => {
      // 1. Resolve Primary Content/URL or Fallback Item
      const activeItem = item.targetContent || item.url ? item : item.fallbackMenuItem;
      if (!activeItem) return null;

      const title = activeItem.title || activeItem.targetContent?.headline || "Untitled";
      const url = activeItem.url || activeItem.targetContent?._metadata?.url || "#";

      // 2. Recursively resolve child sub-menus
      const children = item.subItems && item.subItems.length > 0 
        ? resolveMenuTree(item.subItems) 
        : [];

      return { title, url, children };
    })
    .filter((node): node is ResolvedMenuItem => node !== null);
}

```
