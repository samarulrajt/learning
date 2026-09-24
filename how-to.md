Optimizely generates its GraphQL model structure dynamically through an automated schema mapping engine, and you can build the entire content model directly from the Optimizely CMS **Admin UI** without writing C# code.

---

## Part 1: How Optimizely Generates the GraphQL Model

Optimizely Content Graph continuously inspects the CMS Content Model Repository and automatically maps CMS primitives into GraphQL schema types:

```
┌─────────────────────────┐       ┌──────────────────────────────┐       ┌──────────────────────────────┐
│  Optimizely CMS Model   │       │  Content Graph Sync Provider │       │      GraphQL Endpoint        │
│  (C# or Admin UI)       │  ───> │  (Reflects & Serializes)     │  ───> │  (https://cg.optimizely.com) │
└─────────────────────────┘       └──────────────────────────────┘       └──────────────────────────────┘
  • PageType                       • Creates `_Page` & `_Content`         • Generates GraphQL Types:
  • BlockType                      • Flattens or nests properties         • `NavigationConfigPage`
  • ContentArea / References       • Builds relational references         • `MenuItemBlock`

```

### Automatic Mapping Rules

| Optimizely CMS Type | Generated GraphQL Type | How it Behaves in GraphQL |
| --- | --- | --- |
| **Page Type** (`NavigationConfigPage`) | Queryable root type (`NavigationConfigPage`) | Inherits standard metadata fields (`id`, `displayName`, `url`) and exposes custom page properties. |
| **Block Type** (`MenuItemBlock`) | Complex Object Type | Rendered as a structured nested object or list when linked. |
| **Content Area** (`SubItems`, `HeaderMenu`) | Array of Objects (`[MenuItemBlock]`) | Resolves child blocks sequentially, expanding nested block trees. |
| **Content Reference** (`TargetContent`) | Polymorphic Relation Object | Resolves to the linked page/media item metadata and specific content fields using inline fragments (`... on ArticlePage`). |

---

## Part 2: How to Build This Model in Optimizely CMS UI

If you prefer building content types visually without code, follow these steps in **Optimizely CMS Admin Mode**:

### Step 1: Create the `MenuItemBlock` (Block Type)

1. Open Optimizely CMS and switch to **Admin Mode** (top navigation bar).
2. Go to **Content Type $\rightarrow$ Block Types** and click **New Block Type**.
3. Enter details:
* **Name**: `MenuItemBlock`
* **Display Name**: `Menu Item Block`


4. Save, then click **Add Property** to add the following fields:

| Property Name | Type | Settings / Restrictions |
| --- | --- | --- |
| `Title` | **String** | Display label override |
| `Url` | **String** | Custom URL path |
| `TargetContent` | **Page / Content Reference** | Link to primary target content |
| `FallbackMenuItem` | **Block / Content Reference** | Restrict allowed block types to `MenuItemBlock` |
| `SubItems` | **Content Area** | Restrict allowed types under "Display Settings" to `MenuItemBlock` |

---

### Step 2: Create the `NavigationConfigPage` (Page Type)

1. Under **Admin Mode**, navigate to **Content Type $\rightarrow$ Page Types**.
2. Click **New Page Type**.
3. Enter details:
* **Name**: `NavigationConfigPage`
* **Display Name**: `Navigation Config Page`


4. Click **Add Property** to create the navigation container fields:

| Property Name | Type | Settings / Restrictions |
| --- | --- | --- |
| `LOB` | **String** | Line of Business identifier (e.g., `BANKING`) |
| `HeaderMenu` | **Content Area** | Restrict allowed types to `MenuItemBlock` |
| `FooterMenu` | **Content Area** | Restrict allowed types to `MenuItemBlock` |

---

### Step 3: Create Menu Data in Edit Mode

1. Switch to **Edit Mode**.
2. Open the **For This Page / Assets Pane** on the right side $\rightarrow$ **Blocks tab**.
3. Create new `Menu Item Block` instances for your header, footer, and nested sub-items. Set target links or fallback links.
4. Create a new page of type **Navigation Config Page** (e.g., in a "Global Settings" folder).
5. Set `LOB = "BANKING"`.
6. Drag and drop your `Menu Item Block` instances into the **HeaderMenu** and **FooterMenu** Content Areas.
7. Click **Publish**.

---

### Step 4: Sync UI-Created Models to Content Graph

When you create content types in the UI, trigger a re-index so Optimizely Content Graph updates its GraphQL schema:

1. Return to **Admin Mode**.
2. Go to **Scheduled Jobs**.
3. Select **Optimizely Content Graph Content Indexing Job**.
4. Click **Start Manually**.

Once complete, open `[https://cg.optimizely.com/graphiql](https://cg.optimizely.com/graphiql)`. Your UI-defined page (`NavigationConfigPage`) and block (`MenuItemBlock`) will appear in the GraphQL documentation explorer automatically.
