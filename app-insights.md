To log missing page queries (GraphQL 404s or soft 404s) in Application Insights or Datadog, hook into your page resolution pipeline or Apollo/GraphQL fetch layer. When `targetPage.items` returns empty, fire a custom telemetry event with context like `requestedUrl`, `lob`, and `referrer`.

---

### 1. Unified Telemetry Service (App Insights + Datadog)

Create a telemetry helper module that routes custom events to either Application Insights, Datadog RUM/Logs, or both:

```typescript
// telemetry.ts
import { ApplicationInsights } from '@microsoft/applicationinsights-web';
import { datadogRum } from '@datadog/browser-rum';

// Initialize App Insights (Client-side)
export const appInsights = new ApplicationInsights({
  config: {
    connectionString: process.env.NEXT_PUBLIC_APPINSIGHTS_CONNECTION_STRING,
  }
});
appInsights.loadAppInsights();

// Track Custom Missing Page Event
export interface MissingPageTelemetry {
  requestedUrl: string;
  lob: string;
  fallbackUsed: boolean;
  fallbackUrl?: string;
  referrer?: string;
}

export function logMissingPageEvent(data: MissingPageTelemetry) {
  const eventName = 'GraphQL_PageNotFound';
  const properties = {
    ...data,
    referrer: data.referrer || (typeof document !== 'undefined' ? document.referrer : 'direct'),
    environment: process.env.NODE_ENV,
  };

  // 1. Azure Application Insights
  appInsights.trackEvent({ name: eventName }, properties);
  appInsights.trackMetric({ name: 'MissingPage_Count', average: 1 }, properties);

  // 2. Datadog RUM & Custom Action
  datadogRum.addAction(eventName, properties);
}

```

---

### 2. Intercepting Missing Pages in GraphQL Fetch Layer

Integrate the logger into your `resolvePageData` resolution helper or GraphQL client middleware so every failed lookup automatically dispatches telemetry:

```typescript
import { logMissingPageEvent } from './telemetry';

export function processPageResponse(responseData: any, requestedUrl: string, lob: string) {
  const targetPage = responseData?.targetPage?.items?.[0];
  const fallbackPage = responseData?.fallbackPage?.items?.[0];
  const notFoundPage = responseData?.notFoundPage?.items?.[0];

  // Target Page Found -> Normal Render
  if (targetPage) {
    return { content: targetPage, status: 200 };
  }

  // --- TARGET PAGE MISSING: LOG TELEMETRY ---
  logMissingPageEvent({
    requestedUrl,
    lob,
    fallbackUsed: !!fallbackPage,
    fallbackUrl: fallbackPage?._metadata?.url || 'none'
  });

  // Return fallback or 404 content
  if (fallbackPage) {
    return { content: fallbackPage, status: 200, isFallback: true };
  }

  if (notFoundPage) {
    return { content: notFoundPage, status: 404 };
  }

  return { content: null, status: 404 };
}

```

---

### 3. Server-Side Telemetry (Next.js SSR / Node.js)

If performing server-side rendering (SSR), use server SDKs (`applicationinsights` or `@datadog/browser-logs` / Winston logger) inside your API or Server Component handler:

```typescript
// serverTelemetry.ts (Node.js / Next.js SSR)
import * as appInsights from 'applicationinsights';
import statsd from 'hot-shots'; // Datadog DogStatsD client

const dogstatsd = new statsd();

export function logServerMissingPage(requestedUrl: string, lob: string) {
  // Azure Application Insights Server SDK
  if (appInsights.defaultClient) {
    appInsights.defaultClient.trackEvent({
      name: 'GraphQL_PageNotFound_Server',
      properties: { requestedUrl, lob }
    });
  }

  // Datadog Metric via DogStatsD
  dogstatsd.increment('optimizely.graphql.missing_page', 1, [`lob:${lob}`, `url:${requestedUrl}`]);
}

```

---

### 4. Datadog & App Insights Dashboard Queries

Once telemetry is flowing, set up monitors and alerts using these query structures:

#### Application Insights (Kusto Query Language - KQL)

```kusto
customEvents
| where name == "GraphQL_PageNotFound"
| extend requestedUrl = tostring(customDimensions.requestedUrl), lob = tostring(customDimensions.lob)
| summarize MissingPageCount = count() by requestedUrl, lob
| order by MissingPageCount desc

```

#### Datadog RUM / Log Query

```text
@type:action @action.name:GraphQL_PageNotFound
| stats count() by @requestedUrl, @lob

```

---

### Recommended Telemetry Properties

When logging missing pages, include these properties to detect broken links or SEO crawl issues:

| Attribute | Type | Example | Purpose |
| --- | --- | --- | --- |
| `requestedUrl` | `string` | `/benefits/2026-plans` | Identifies the missing path. |
| `lob` | `string` | `BANKING` | Tracks which business unit is affected. |
| `fallbackUsed` | `boolean` | `true` | Indicates if the user was saved by a fallback page. |
| `referrer` | `string` | `[https://google.com](https://google.com)` | Helps identify broken incoming external links or old indexed URLs. |
| `userLanguage` | `string` | `en-US` | Useful for multi-locale Optimizely sites. |
