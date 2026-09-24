## 1. How Optimizely UI Maps to the GraphQL Query & Response

Every section of the GraphQL query and JSON response directly corresponds to content created by editors inside **Optimizely CMS Edit Mode**.

```
  Optimizely CMS Edit Mode                          GraphQL JSON Response
┌───────────────────────────────────────┐          ┌───────────────────────────────────┐
│ Page Tree: /benefit (ArticlePage)     │ ────────>│ "targetPage": { "headline": ... } │
├───────────────────────────────────────┤          ├───────────────────────────────────┤
│ Assets Pane: Menu Blocks (SubItems)   │ ────────>│ "headerMenu": [ { "subItems" } ]  │
├───────────────────────────────────────┤          ├───────────────────────────────────┤
│ Navigation Config Page (LOB: BANKING) │ ────────>│ "navigationData": { "lob" }       │
├───────────────────────────────────────┤          ├───────────────────────────────────┤
│ Blocks Pane: Redirect Rules           │ ────────>│ "redirectRules": []               │
└───────────────────────────────────────┘          └───────────────────────────────────┘

```

---

## 2. Step-by-Step UI Guide ("How To Create This in Optimizely")

### Step 1: Create Your Content Pages

1. In **Edit Mode**, go to the **Page Tree** (Left Navigation Pane).
2. Create three pages:
* **`ArticlePage`**: Title = `"Comprehensive Health & Wellness Benefits"`, URL = `/benefit`.
* **`LandingPage`**: Title = `"Welcome to Your Banking Portal"`, URL = `/dashboard`.
* **`ArticlePage`**: Title = `"404 - Page Not Found"`, URL = `/404`.


3. Fill in the `Headline`, `MainBody`, and `Author` fields, then click **Publish** on each page.

> **How this maps to GraphQL:**
> These pages populate `targetPage`, `fallbackPage`, and `notFoundPage` in the response payload.

---

### Step 2: Create Menu Blocks and Fallbacks

1. Open the **Assets Pane** (Right side) and select the **Blocks** tab.
2. Click **+** to create a new **Menu Item Block**:
* **Block 1 (Parent):** Name = `Benefits & Perks Menu Item`.
* **Target Content:** Drag and drop the `/benefit` page from the Page Tree into this reference field.


* **Block 2 (Child):** Name = `Medical Coverage Menu Item`.
* **Target Content:** Drag and drop `/benefit/medical`.


* **Block 3 (Child with Fallback):** Name = `Unpublished Sub-menu Item`.
* **Target Content:** Leave **EMPTY** (or link to an unpublished draft page).
* **Fallback Menu Item:** Create/link a block named `Default Wellness Stipend` pointing to `/benefit/wellness-default`.




3. Nest child blocks inside the parent block by dragging Block 2 and Block 3 into the **Sub Items** Content Area of Block 1.
4. Click **Publish** on all created blocks.

---

### Step 3: Assemble the Navigation Config Page

1. In the **Page Tree**, create a page of type **Navigation Config Page** under a global settings folder.
2. Set the properties:
* **LOB:** Enter `BANKING`.


3. Open the **Assets Pane $\rightarrow$ Blocks tab**:
* Drag `Benefits & Perks Menu Item` and `Find Care` into the **Header Menu** Content Area.
* Drag `Terms of Service` and `Help Center` into the **Footer Menu** Content Area.


4. Click **Publish**.

---

### Step 4: Create Redirect Rules (Optional)

1. In the **Assets Pane $\rightarrow$ Blocks tab**, click **+** and choose **Redirect Rule Block**.
2. Set **Old URL** = `/old-benefit-plan` and **New URL** = `/benefit`.
3. Set **Is Active** = `True` and click **Publish**.

---

## 3. Dissecting the Query & Response Line-by-Line

### Section A: `redirectRules`

#### GraphQL Request:

```graphql
redirectRules: RedirectRuleBlock(
  where: { OldUrl: { eq: "/benefit" }, IsActive: { eq: true } }
) {
  items { OldUrl NewUrl StatusCode }
}

```

#### JSON Response:

```json
"redirectRules": {
  "items": []
}

```

* **What it means:** Content Graph checks if editors defined a `RedirectRuleBlock` where `OldUrl` equals `"/benefit"`.
* **Why it's empty:** `/benefit` is an active target page, not a legacy URL, so no redirect rule was found.

---

### Section B: `navigationData` & Fallback Mechanism

#### GraphQL Request:

```graphql
navigationData: NavigationConfigPage(where: { lob: { eq: "BANKING" } }) {
  items {
    headerMenu {
      title
      targetContent { _metadata { url } headline }
      fallbackMenuItem { title targetContent { _metadata { url } } }
      subItems { ... }
    }
  }
}

```

#### JSON Response (Excerpt):

```json
{
  "title": "Unpublished Sub-menu Item",
  "url": null,
  "targetContent": null,
  "fallbackMenuItem": {
    "title": "Default Wellness Stipend",
    "url": "/benefit/wellness-default",
    "targetContent": {
      "_metadata": { "id": "1005", "url": "/benefit/wellness-default" },
      "headline": "Standard Wellness Allowance"
    }
  }
}

```

* **What it means:**
1. The CMS editor added a `MenuItemBlock` titled `"Unpublished Sub-menu Item"`.
2. The linked page was deleted or unpublished in the CMS, so `targetContent` returned `null`.
3. Content Graph resolves the `fallbackMenuItem` property linked inside that block, returning the default fallback item (`/benefit/wellness-default`).



---

### Section C: Page Content Resolution (`targetPage`, `fallbackPage`, `notFoundPage`)

#### JSON Response:

```json
"targetPage": {
  "items": [
    {
      "_metadata": { "id": "1050", "url": "/benefit", "types": ["ArticlePage"] },
      "headline": "Comprehensive Health & Wellness Benefits",
      "mainBody": "<p>Welcome to your 2026 employee benefits guide...</p>",
      "author": "HR Benefits Team"
    }
  ]
},
"fallbackPage": {
  "items": [
    {
      "_metadata": { "id": "1001", "url": "/dashboard", "types": ["LandingPage"] },
      "heroTitle": "Welcome to Your Banking Portal"
    }
  ]
},
"notFoundPage": {
  "items": [
    {
      "_metadata": { "id": "4040", "url": "/404", "types": ["ArticlePage"] },
      "headline": "404 - Page Not Found"
    }
  ]
}

```

* **How your app processes this:**
1. Your frontend checks `targetPage.items[0]`. Since an `ArticlePage` exists for `/benefit`, it renders this content immediately.
2. The payload for `fallbackPage` (`/dashboard`) and `notFoundPage` (`/404`) are ignored by your app for this route, but were sent in the same request to guarantee zero round-trips if `/benefit` had been missing.
