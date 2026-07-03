# React / Browser Setup

## When to use this reference

- User has React components (`.tsx`)
- App runs in the browser (client-side logging)
- Needs error boundary + event tracking

## Install

```bash
npm install @flarelog/sdk
```

## Setup

### Error Boundary

```tsx
import { flarelog } from "@flarelog/sdk";
import { FlareLogErrorBoundary } from "@flarelog/sdk/react";

const logger = flarelog({
  apiKey: import.meta.env.VITE_FLARELOG_API_KEY,
  // or process.env.REACT_APP_FLARELOG_API_KEY for Create React App
});

export default function App() {
  return (
    <FlareLogErrorBoundary logger={logger} fallback={<ErrorPage />}>
      <Router />
    </FlareLogErrorBoundary>
  );
}
```

The error boundary catches render errors, logs them with the component stack,
and shows a fallback UI.

### useFlareLog hook

```tsx
import { useFlareLog } from "@flarelog/sdk/react";

function CheckoutButton() {
  const { trackEvent, trackError, setUser, addBreadcrumb } = useFlareLog(logger);

  const handleCheckout = async () => {
    trackEvent("checkout_started", { items: 3 });
    try {
      await processPayment();
      trackEvent("checkout_completed");
    } catch (err) {
      trackError(err, { step: "payment" });
    }
  };

  return <button onClick={handleCheckout}>Checkout</button>;
}
```

### Page view tracking

```tsx
import { useFlareLogPageView } from "@flarelog/sdk/react";

function HomePage() {
  useFlareLogPageView(logger, "Home");
  return <div>Home</div>;
}
```

## API key for the browser

Use the **same** `FLARELOG_API_KEY` everywhere. The only difference is that
most frontend build tools require a prefix to expose an env var to the browser:

- **Vite**: `VITE_FLARELOG_API_KEY` → read with `import.meta.env.VITE_FLARELOG_API_KEY`
- **Next.js**: `NEXT_PUBLIC_FLARELOG_API_KEY` → read with `process.env.NEXT_PUBLIC_FLARELOG_API_KEY`
- **Create React App**: `REACT_APP_FLARELOG_API_KEY` → read with `process.env.REACT_APP_FLARELOG_API_KEY`

```bash
# .env (Vite example)
VITE_FLARELOG_API_KEY=fl_your_key
```

The prefix only tells the bundler to inject the value into the client bundle.
It is **not** a different or safer/less-safe key — it is the same FlareLog API
key used on the server.

## What the components do

`FlareLogErrorBoundary`:
- Catches errors during rendering, in lifecycle methods, and in constructors
- Calls `logger.logError(error, { message: "React error boundary caught error", metadata: { componentStack, reactVersion } })`
- Shows `fallback` prop (or default UI) when an error is caught

`useFlareLog(logger)`:
- Returns `{ trackEvent, trackError, setUser, addBreadcrumb }`
- `trackEvent(name, data)` → `logger.info(name, data)`
- `trackError(err, context)` → `logger.logError(err, { metadata: context })`
- `setUser(user)` → `logger.setUser(user)`
- `addBreadcrumb({ category, message, data })` → `logger.addBreadcrumb(...)`

`useFlareLogPageView(logger, pageName)`:
- Logs a page view on mount with `path` and `referrer`

## See also

- `references/nextjs.md` — for server-side logging in Next.js apps
