# Sentry for React Native: Performance & Debugging Guide

> Complete guide to performance monitoring, error tracking, and debugging for React Native apps using Sentry.

Based on workshop by **Lazar Nicolov** (Senior DevX Engineer at Sentry)

---

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Performance Monitoring](#performance-monitoring)
  - [Mobile Vitals](#mobile-vitals)
  - [Trace View](#trace-view)
  - [Custom Instrumentation](#custom-instrumentation)
- [Error Debugging](#error-debugging)
  - [Issue Details](#issue-details)
  - [Breadcrumbs](#breadcrumbs)
  - [Session Replay](#session-replay)
  - [Logs](#logs)
- [Seer AI Debugger](#seer-ai-debugger)
- [Size Analysis](#size-analysis)
- [Quick Reference](#quick-reference)
- [Additional Resources](#additional-resources)

---

## Installation

Run the Sentry wizard at your project root:

```bash
npx sentry wizard -i react-native
```

### What the wizard configures:

| Task | Description |
|------|-------------|
| Package installation | Installs `@sentry/react-native` |
| Metro config | Wraps Metro bundler configuration |
| Expo config | Adds Sentry Expo plugin |
| Android (Gradle) | Configures source map uploads |
| iOS (Xcode) | Adds build phase script for debug symbols |
| CocoaPods | Runs `pod install` automatically |
| DSN setup | Configures your project DSN in `layout.tsx` |

> ⚠️ **Important:** Requires Expo dev build (EAS Build), not Expo Go, due to native modules.

---

## Configuration

### Basic Setup (`app/_layout.tsx`)

```typescript
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  
  // Performance
  tracesSampleRate: 0.2, // 20% in production, 1.0 for dev
  enableUserInteractionTracing: true,
  enableNativeFramesTracking: true,
  
  // Privacy
  sendDefaultPii: true, // Set false if needed
  
  // Integrations
  integrations: [
    new Sentry.ReactNavigationInstrumentation({
      enableTimeToInitialDisplay: true,
    }),
    new Sentry.MobileReplay({
      maskAllText: true,      // PII protection
      maskAllImages: true,
      maskAllVectors: true,
    }),
    new Sentry.ConsoleLoggingIntegration(), // console.log → Sentry logs
  ],
});
```

### Wrap Root Component

```typescript
function RootLayout() {
  return (
    <Sentry.ErrorBoundary fallback={<ErrorFallback />}>
      {/* Your app */}
    </Sentry.ErrorBoundary>
  );
}

export default Sentry.wrap(RootLayout);
```

### Error Boundary Fallback

```typescript
const ErrorFallback = () => (
  <View style={styles.container}>
    <Text>Oops! Something went wrong.</Text>
    <Text>We've been notified and are working on a fix.</Text>
  </View>
);
```

---

## Performance Monitoring

### Mobile Vitals

**Location:** Sentry Dashboard → Insights → Mobile

#### Available Metrics

| Metric | Description |
|--------|-------------|
| Cold Start | App launch from terminated state |
| Warm Start | App launch from background |
| TTID | Time to Initial Display |
| TTFD | Time to Full Display |
| Slow Frames | Frames taking >16ms |
| Frozen Frames | Frames taking >700ms |
| Screen Loads | Number of times screen rendered |

#### Device Distribution

Sentry categorizes devices into tiers:
- **High** — Flagship devices
- **Medium** — Mid-range devices  
- **Low** — Budget devices
- **Unknown**

> 💡 **Tip:** Always filter by platform (iOS/Android) in the top-right corner.

---

### Trace View

Traces visualize **where time is spent** as a waterfall diagram.

#### Automatically Captured Spans

- `UI.load` — Screen mounting
- Navigation processing
- HTTP requests (fetch/axios)
- JS bundle execution
- Native module loading
- Database queries (if backend instrumented)

#### Example Cold Start Trace

```
UI.load.index (6.13s)
├── Cold Start (6.06s)
│   ├── UIKit init → JSX start (4.78s)
│   ├── Native modules loading
│   └── JS bundle execution
├── Navigation processing (36ms)
├── Root layout mount
└── GET /api/profile (62ms)
    └── [Backend] Express handler
        └── SQL SELECT (3.39ms)
```

#### Cross-Service Tracing

If your backend also uses Sentry, traces connect automatically:

```
[React Native] HTTP Client request
    └── [Express.js] API endpoint
        └── [Database] SQL query
```

---

### Custom Instrumentation

Use custom spans for critical flows not automatically captured.

#### Basic Custom Span

```typescript
import * as Sentry from '@sentry/react-native';

const handleSearch = async (query: string, page: number) => {
  return Sentry.startSpan(
    {
      name: 'search products',
      op: 'function',
      attributes: {
        'attr.query': query,
        'attr.page': page,
      },
    },
    async (span) => {
      try {
        const results = await searchAPI(query, page);
        
        // Add attributes after response
        span.setAttributes({
          'attr.totalCount': results.total,
          'attr.hasMore': results.products.length > 0,
        });
        
        return results;
      } catch (error) {
        Sentry.captureException(error);
        span.setStatus({ code: 2, message: error.message });
        throw error;
      }
    }
  );
};
```

#### Querying Custom Spans

Navigate to **Explore → Traces** and filter:

```
span.description = "search products"
```

#### When to Add Custom Spans

| ✅ Add spans for | ❌ Skip spans for |
|------------------|-------------------|
| Checkout flow | Simple UI toggles |
| Search/filter operations | Navigation (auto-captured) |
| File uploads | HTTP requests (auto-captured) |
| Complex calculations | Database queries (auto on backend) |
| Third-party API calls | |

---

## Error Debugging

### Issue Details

**Location:** Sentry Dashboard → Issues

#### Issue Page Components

1. **Timeline** — Frequency and occurrence pattern
2. **Tags** — Device, OS, release, environment, custom tags
3. **Stack Trace** — Component hierarchy + actual error location
4. **Suspect Commits** — Git commits that touched affected code

#### Stack Trace Example

```
TypeError: Cannot read property 'toUpperCase' of undefined

React Error Boundary hierarchy:
  RootLayout → ProfileProvider → ErrorBoundary → MainLayout → AnalysisScreen

Actual error:
  at analysis.tsx:238
  → diet!.toUpperCase()  // diet is undefined
```

#### Custom Tags

```typescript
// Add tags for filtering/prioritization
Sentry.setTag('user_tier', 'premium');
Sentry.setTag('feature_flag', 'new_checkout');
```

---

### Breadcrumbs

Automatic timeline of user actions leading to a crash.

#### Example Breadcrumb Trail

```
[touch]      User tapped search
[console]    "searching for Frank"
[http]       GET /api/search → 200
[navigation] search → product-details
[console]    "changing diet from keto to none"  ← 🔍 Hint!
[http]       PUT /api/profile → 200
[navigation] → analysis
[error]      Cannot read property toUpperCase of undefined
```

#### Manual Breadcrumbs

```typescript
Sentry.addBreadcrumb({
  category: 'user-action',
  message: 'User completed onboarding',
  level: 'info',
  data: {
    step: 'profile-setup',
    duration: 45000,
  },
});
```

#### Filtering Breadcrumbs

Filter by level in the UI:
- `debug`
- `info`
- `warning`
- `error`

---

### Session Replay

Video-like recording of user sessions (periodic screenshots).

#### Features

| Feature | Description |
|---------|-------------|
| Timeline sync | Aligned with breadcrumbs |
| Trace sync | Shows backend operations |
| Log sync | Console output overlay |
| PII masking | Text/images masked by default |

#### Configuration

```typescript
new Sentry.MobileReplay({
  maskAllText: true,      // Mask all text (PII protection)
  maskAllImages: true,    // Mask all images
  maskAllVectors: true,   // Mask vector graphics
})
```

> 💡 **Use case:** See exactly what the user did before crash without asking them to reproduce.

---

### Logs

Console logs are auto-captured via `ConsoleLoggingIntegration`.

#### Power Queries

Find patterns across all users:

```sql
-- How many users changed diet to "none"?
message matches "changing diet from * to none"

-- Result: 7 matches in 11,000 logs
-- Insight: Potential crash wave incoming!
```

#### Log Levels

```typescript
console.log('info message');    // level: info
console.warn('warning message'); // level: warning
console.error('error message');  // level: error
```

---

## Seer AI Debugger

Sentry's AI-powered debugging assistant.

### Capabilities

1. **Ingests** — Stack traces, breadcrumbs, logs, traces
2. **Analyzes** — Identifies root cause
3. **Proposes** — Solution with explanation
4. **Generates** — Code diff
5. **Opens PR** — Directly on GitHub

### Example Root Cause Analysis

```
Root Cause: Silent null handling in backend

Timeline:
1. User selects "none" for diet
2. Backend stores NULL in database
3. Backend returns null (not undefined)
4. Frontend receives { diet: null }
5. Analysis screen calls diet!.toUpperCase()
6. TypeError thrown

Solution:
- Backend: Return undefined instead of null
- Frontend: Add defensive check before toUpperCase()
```

### Generated Fix

**Backend (`profile-repository.ts`):**
```diff
- diet: row.diet ?? null
+ diet: row.diet ?? undefined
```

**Frontend (`analysis.tsx`):**
```diff
- {diet!.toUpperCase()}
+ {diet && diet.toUpperCase()}
```

### Automation Levels

| Level | Description |
|-------|-------------|
| Root cause only | Just identify the problem |
| Find solution | + Propose how to fix |
| Generate changes | + Create code diff |
| Open PR | + Push to GitHub automatically |

### Trigger Options

- Manual — Click "Start root cause analysis"
- Auto on all issues
- Auto on actionable issues only (top 2%)

---

## Size Analysis

> 🚧 Coming soon in beta

### Features

| Feature | Description |
|---------|-------------|
| Bundle breakdown | JS, native code, assets |
| Actionable insights | Specific optimization suggestions |
| Build comparison | Diff between versions |
| CI integration | Size checks on every commit |

### Example Insights

| Issue | Potential Savings |
|-------|-------------------|
| Remove Hermes debug info | ~12MB |
| Strip binary symbols | ~5MB |
| Optimize images → HEIC | ~2MB |
| Remove unused assets | Varies |

### Supported Formats

- **iOS:** XCArchive
- **Android:** AAB (Android App Bundle)

---

## Quick Reference

### Common Operations

```typescript
import * as Sentry from '@sentry/react-native';

// Custom span
Sentry.startSpan(
  { name: 'operation', op: 'function' },
  async (span) => {
    // your code
  }
);

// Capture exception
Sentry.captureException(error);

// Capture message
Sentry.captureMessage('Something happened', 'warning');

// Set user context
Sentry.setUser({
  id: 'user-123',
  email: 'user@example.com',
  subscription: 'premium',
});

// Add tag
Sentry.setTag('feature', 'checkout-v2');

// Add breadcrumb
Sentry.addBreadcrumb({
  category: 'action',
  message: 'User completed step',
  level: 'info',
});

// Set context
Sentry.setContext('order', {
  id: 'order-456',
  total: 99.99,
});
```

### Offline Support

| Platform | Behavior |
|----------|----------|
| Android | Events cached, sent on app restart |
| iOS | Events cached, sent when back online |
| Expo Go | ❌ Not supported (use EAS Build) |

---

## Additional Resources

| Resource | Link |
|----------|------|
| Sentry Docs | [docs.sentry.io](https://docs.sentry.io) |
| React Native SDK | [docs.sentry.io/platforms/react-native](https://docs.sentry.io/platforms/react-native) |
| MCP Server | [mcp.sentry.dev](https://mcp.sentry.dev) |
| Pricing | [sentry.io/pricing](https://sentry.io/pricing) |

### MCP Server Tools

Use Sentry directly in your IDE:
- Analyze issues
- Create projects
- Query traces
- Trigger Seer

---

## License

This guide is based on publicly available Sentry workshop content.

---

**Last updated:** December 2024
