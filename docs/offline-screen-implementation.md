# Offline Screen — React Implementation

Internal reference for the team. Covers how the application detects internet connectivity issues and displays an offline fallback screen, preventing users from interacting with the app when disconnected.

---

## Overview

When a user loses internet connectivity, the application immediately replaces the main routing view with an **Offline Screen**. This ensures that users do not attempt to fetch data or submit forms, which would fail or lead to a bad state.

Features:

- **Real-time detection**: Uses browser `online` and `offline` events to react instantly.
- **Auto-recovery**: When the connection is restored, the app notifies the user and automatically refreshes.
- **Manual retry**: Users can manually click a "Retry Connection" button.

---

## Architecture at a Glance

```text
Browser Network State Changes
       |
       v
useOnlineStatus (Hook)
  ├─ Listens to "online" & "offline" events
  └─ Returns `isOnline` boolean
       |
       v
App.jsx (Root Component)
  ├─ if (!isOnline) → renders <OfflineScreen />
  └─ if (isOnline)  → renders <RenderRouter /> (normal app flow)
       |
       v
OfflineScreen.jsx
  ├─ Displays Ant Design <Result> with warning UI
  ├─ Auto-retries when `isOnline` becomes true
  └─ Allows manual retry via button
```

### Files involved

| File                                      | Role                                                                     |
| ----------------------------------------- | ------------------------------------------------------------------------ |
| `src/hooks/useOnlineStatus.js`            | Custom hook that tracks browser network connectivity state in real-time. |
| `src/App.jsx`                             | The root component that conditionally blocks the app UI when offline.    |
| `src/components/common/OfflineScreen.jsx` | The fallback UI displayed during an offline state.                       |

---

## 1. `useOnlineStatus` Hook

The `useOnlineStatus` hook provides a simple boolean state representing whether the browser is connected to the network.

```javascript
// src/hooks/useOnlineStatus.js
const useOnlineStatus = () => {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    globalThis.addEventListener("online", handleOnline);
    globalThis.addEventListener("offline", handleOffline);

    return () => {
      globalThis.removeEventListener("online", handleOnline);
      globalThis.removeEventListener("offline", handleOffline);
    };
  }, []);

  return isOnline;
};
```

- Relies on `navigator.onLine` for the initial state.
- Subscribes to global `online` and `offline` events.
- Cleans up event listeners on unmount.

---

## 2. Root Integration in `App.jsx`

The hook is utilized at the very top of the React component tree in `App.jsx`.

```javascript
// src/App.jsx
import RenderRouter from "./routes/RenderRouter.jsx";
import useOnlineStatus from "./hooks/useOnlineStatus";
import OfflineScreen from "./components/common/OfflineScreen";

function App() {
  const isOnline = useOnlineStatus();

  if (!isOnline) {
    return <OfflineScreen />;
  }

  return <RenderRouter />;
}
```

By placing this at the root:

1. It unmounts the entire active application (`RenderRouter`), preventing any background polling or user interactions while offline.
2. The user sees a dedicated offline UI.

---

## 3. `OfflineScreen` Component

The `OfflineScreen` provides the visual feedback using Ant Design components (`Result`, `Typography`, `Button`, `notification`).

### 3.1 Manual Retry Logic

Users can click "Retry Connection". The retry logic validates the current connectivity status using the `isOnline` hook.

```javascript
const handleRetry = useCallback(async () => {
  setIsRetrying(true);
  try {
    if (isOnline) {
      if (onRetry) {
        await onRetry();
      } else {
        globalThis.location.reload();
      }
    } else {
      showOfflineNotification();
    }
  } catch (error) {
    // Show error notification
  } finally {
    setIsRetrying(false);
  }
}, [onRetry, isOnline]);
```

- If `isOnline` is true, it proceeds to recover the session. It either calls a custom `onRetry` prop or forcefully reloads the page (`globalThis.location.reload()`).
- If `isOnline` is still false, it triggers a bottom-aligned warning notification to inform the user they are still offline.

### 3.2 Auto Retry

Instead of forcing users to manually click the button when their internet comes back, `OfflineScreen` watches `isOnline` to recover automatically.

```javascript
useEffect(() => {
  const handleOnline = () => {
    if (isOnline) {
      notification.success({
        // Shows connection restored warning
        // ...
      });
      setTimeout(() => {
        handleRetry();
      }, 1000);
    }
  };

  if (isOnline) {
    handleOnline();
  }
}, [isOnline, handleRetry]);
```

- Once the `online` event fires (updating `isOnline`), a success notification confirms the restoration.
- A `setTimeout` of 1 second gives the user time to read the notification before the page reloads.

### 3.3 UI Layout

The UI uses centralized Flexbox container styles and an Ant Design `Result` component to render the standard "No Internet Connection" visual with an icon (`WifiOutlined`).

---

## 4. How to Use / Extend

This implementation works universally across the app.

If a specific view requires custom offline behavior without tearing down the entire UI, developers can:

1. Extract `useOnlineStatus` inside the specific component.
2. Render `<OfflineScreen onRetry={customFetchLogic} />` component inside their specific view. Passing the `onRetry` allows bypassing the default global page reload behavior.
