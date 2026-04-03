# Network Status — React Implementation

Internal reference for the team. Covers how the online/offline status monitoring and the fallback "Connection Lost" modal are implemented on the frontend.

---

## Architecture at a Glance

```
App.jsx
  └─ useNetworkStatus()
       ├─ listens for window "online" / "offline"
       └─ monkey-patches fetch/XHR
  └─ NetworkModal
       └─ reacts to isOnline status
       └─ shows fullscreen overlay if offline
```

### Files involved

| File | Role |
|------|------|
| `src/hooks/useNetworkStatus.js` | Global hook for monitoring connectivity and intercepting requests |
| `src/components/common/NetworkModal.jsx` | UI component for the "Connection Lost" overlay |
| `src/assets/css/NetworkModal.css` | Premium styles (glassmorphism, animations) for the modal |
| `src/App.jsx` | Root component that initializes the hook and modal |

---

## 1. Network Monitoring (`useNetworkStatus`)

This hook acts as the central source of truth for the application's connectivity state. It handles both event listeners and request interception.

### Event Listeners
It subscribes to browser-native events to toggle the `isOnline` state immediately.

```javascript
window.addEventListener("online", handleOnline);
window.addEventListener("offline", handleOffline);
```

### Request Interception
To prevent the application from crashing or showing broken UI during transit when the connection drops, the hook monkey-patches `window.fetch` and `XMLHttpRequest`. 

> [!IMPORTANT]
> If `navigator.onLine` is false, intercepted requests return a **non-resolving Promise**. This effectively pauses the request until the network is restored, preventing "Failed to fetch" errors from bubbling up to the components.

```javascript
const originalFetch = window.fetch;
window.fetch = async function (...args) {
  if (!navigator.onLine) {
    return new Promise(() => {}); // Pause request
  }
  return originalFetch.apply(this, args);
};
```

---

## 2. The Network Modal (`NetworkModal`)

The `NetworkModal` is a global component that reacts to the `isOnline` state provided by the hook. It is designed with a premium, glassmorphic aesthetic to match the app's high-end feel.

### Component Logic
The modal uses local state to handle the transition from "offline" to "online" smoothly. When the connection is restored, it shows a "Success" state for 1 second before automatically closing.

```javascript
useEffect(() => {
  if (!isOnline) {
    setIsVisible(true);
    setStatus("offline");
  } else if (isVisible && status === "offline") {
    setStatus("online");
    const timer = setTimeout(() => setIsVisible(false), 1000);
    return () => clearTimeout(timer);
  }
}, [isOnline]);
```

### Visual Features
- **Glassmorphism**: Uses `backdrop-filter: blur(20px)` and semi-transparent backgrounds for a modern look.
- **Micro-animations**: 
    - `rippleRed` / `rippleGreen`: Pulsing rings around the icon to indicate activity.
    - `pulseOrganic`: Subtle breathing effect on the offline icon.
    - `modalEnter`: Smooth scale-and-fade entry transition.

---

## 3. Global Integration

The feature is integrated at the root level in `App.jsx` to ensure it is always active across all routes.

```jsx
// src/App.jsx
import NetworkModal from "./components/common/NetworkModal";
import useNetworkStatus from "./hooks/useNetworkStatus";

function App() {
  useNetworkStatus(); // Initialize monitoring

  return (
    <>
      <NetworkModal /> {/* Render global overlay */}
      <RenderRouter />
    </>
  );
}
```

---

## 4. CSS Variables & Theming

The modal's look is controlled via CSS variables in `NetworkModal.css`, making it easy to adjust the colors to match the app's brand.

```css
:root {
  --modal-bg: rgba(255, 255, 255, 0.7);
  --modal-border: rgba(255, 255, 255, 0.3);
  --accent-red: #ff4d4f;
  --accent-green: #52c41a;
}
```

Support for "Restored" state is triggered by the `.online` class added to the modal content.
