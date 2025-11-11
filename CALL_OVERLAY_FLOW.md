# Call Overlay System - Flow Explanation

## Overview
This system shows a modal/overlay when a user makes or receives a call, even when the app is killed.

---

## Architecture Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER MAKES/RECEIVES CALL                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│         Android System Broadcasts Call State Change              │
│         ACTION: TelephonyManager.ACTION_PHONE_STATE_CHANGED     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ┌────────┴────────┐
                    │                 │
         ┌──────────▼──────────┐     │
         │  Is App Running?    │     │
         └──────────┬──────────┘     │
                    │                 │
        ┌───────────┴───────────┐     │
        │                       │     │
    YES │                       │ NO  │
        │                       │     │
        ▼                       ▼     │
┌───────────────┐    ┌──────────────────┐
│ App Running   │    │ App Killed/      │
│ (Foreground)  │    │ Background       │
└───────┬───────┘    └────────┬─────────┘
        │                     │
        │                     │
        ▼                     ▼
┌──────────────────┐  ┌──────────────────────┐
│ CallStateModule  │  │ CallStateReceiver    │
│ (Dynamic         │  │ (Manifest Registered)│
│  Receiver)       │  │                      │
└──────┬───────────┘  └──────────┬───────────┘
       │                         │
       │                         │
       ▼                         ▼
┌──────────────────┐  ┌──────────────────────┐
│ Emits Event to   │  │ Starts              │
│ React Native     │  │ CallOverlayService  │
│ via EventEmitter │  │ directly            │
└──────┬───────────┘  └──────────┬───────────┘
       │                         │
       │                         │
       └──────────┬──────────────┘
                  │
                  ▼
         ┌─────────────────────┐
         │ useCallState Hook   │
         │ (React Native)      │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ CallOverlayModule   │
         │ .showCallActive()   │
         │ or                  │
         │ .showCallEnded()    │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ CallOverlayService  │
         │ (Foreground Service)│
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ Checks Overlay      │
         │ Permission          │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ Creates Overlay View│
         │ (XML or Programmatic)│
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ Shows Overlay using │
         │ WindowManager       │
         │ (System-wide)       │
         └─────────────────────┘
```

---

## Detailed Component Flow

### 1. **Call Detection Layer (Native Android)**

#### When App is RUNNING:
```
User Dials Call
    ↓
Android System Broadcasts: ACTION_PHONE_STATE_CHANGED
    ↓
CallStateModule (Dynamic Receiver) catches it
    ↓
Emits event to React Native: "onCallStateChanged"
    ↓
React Native useCallState hook receives event
    ↓
Calls CallOverlayModule.showCallActiveOverlay()
```

#### When App is KILLED:
```
User Dials Call
    ↓
Android System Broadcasts: ACTION_PHONE_STATE_CHANGED
    ↓
CallStateReceiver (Manifest Receiver) catches it
    ↓
Checks: Is app running? → NO
    ↓
Directly starts CallOverlayService
    ↓
Service shows overlay (no React Native needed)
```

---

### 2. **React Native Layer**

```javascript
// src/hooks/useCallState.ts
useEffect(() => {
  // 1. Start listening to call events
  CallStateModule.startListening();
  
  // 2. Listen for events
  const subscription = CallStateEmitter.addListener(
    'onCallStateChanged',
    async (event) => {
      if (event.state === 'CALLING') {
        // 3. Fetch contact info
        const info = await fetchContactInfo(event.phoneNumber);
        
        // 4. Show overlay (if permission granted)
        if (hasOverlayPermission) {
          await CallOverlayModule.showCallActiveOverlay(
            event.phoneNumber,
            info.name,
            info.email,
            info.status
          );
        } else {
          // 5. Fallback: Show regular modal
          setIsCallActive(true);
        }
      }
    }
  );
}, []);
```

---

### 3. **Native Service Layer**

```kotlin
// CallOverlayService.kt
onStartCommand() {
  when (action) {
    ACTION_SHOW_CALL_ACTIVE -> {
      // 1. Check overlay permission
      if (!canDrawOverlays()) return
      
      // 2. Create overlay view
      overlayView = inflate(R.layout.call_overlay)
      
      // 3. Update views with contact info
      tvContactName.text = contactName
      tvPhoneNumber.text = phoneNumber
      
      // 4. Show overlay using WindowManager
      windowManager.addView(overlayView, layoutParams)
    }
  }
}
```

---

## Key Components

### **Native Modules (Kotlin)**

1. **CallStateModule** - Detects calls when app is running
   - Dynamically registers BroadcastReceiver
   - Emits events to React Native

2. **CallStateReceiver** - Detects calls when app is killed
   - Registered in AndroidManifest.xml
   - Works independently of React Native

3. **CallOverlayService** - Shows the overlay
   - Foreground service (runs even when app is killed)
   - Uses WindowManager to show system-wide overlay

4. **CallOverlayModule** - React Native bridge
   - Methods: `showCallActiveOverlay()`, `showCallEndedOverlay()`, `hideOverlay()`
   - Checks overlay permission

### **React Native Components**

1. **useCallState Hook** - Main orchestrator
   - Listens to call events
   - Fetches contact info
   - Shows overlay or modal

2. **CallActiveModal** - Regular modal (fallback)
   - Shows when overlay permission not granted
   - Only works when app is in foreground

3. **CallEndedModal** - Regular modal (fallback)
   - Shows after call ends
   - Only works when app is in foreground

---

## Permission Flow

```
App Starts
    ↓
useCallState hook checks overlay permission
    ↓
┌─────────────────────────────┐
│ Has Overlay Permission?     │
└───────────┬─────────────────┘
            │
    ┌───────┴───────┐
    │               │
  YES              NO
    │               │
    │               ▼
    │      ┌──────────────────┐
    │      │ Log warning      │
    │      │ (will use modal) │
    │      └──────────────────┘
    │
    ▼
When call happens:
    │
    ├─► Show System Overlay (works everywhere)
    │
    └─► OR Show Regular Modal (only in app)
```

---

## State Management

### Call States:
1. **RINGING** - Incoming call ringing
2. **OFFHOOK** - Call active (answered/outgoing)
3. **IDLE** - Call ended

### Overlay States:
- `isOverlayVisible` - Is overlay currently shown?
- `lastOverlayAction` - Last action (to prevent duplicates)
- `lastOverlayTime` - Timestamp (for debouncing)

---

## Error Handling Flow

```
Try to show overlay
    ↓
┌──────────────────────┐
│ Check permission     │
└──────────┬───────────┘
           │
      ┌────┴────┐
      │         │
   Granted   Denied
      │         │
      │         └─► Return (silently fail)
      │
      ▼
┌──────────────────────┐
│ Try XML layout       │
└──────────┬───────────┘
           │
      ┌────┴────┐
      │         │
   Success   Fails
      │         │
      │         └─► Create programmatic view
      │
      ▼
┌──────────────────────┐
│ Add to WindowManager │
└──────────┬───────────┘
           │
      ┌────┴────┐
      │         │
   Success   Fails
      │         │
      │         └─► Log error (don't crash)
      │
      ▼
   Overlay Shown
```

---

## Important Notes for React Native Developers

1. **Two Receivers**: One for running app, one for killed app
   - Prevents duplicate events
   - Ensures overlay works even when app is killed

2. **Foreground Service**: Required to show overlay when app is killed
   - Shows persistent notification
   - Keeps service alive

3. **System Overlay Permission**: Required for overlay to work
   - User must grant "Display over other apps" permission
   - Falls back to regular modal if not granted

4. **Debouncing**: Prevents duplicate overlays
   - 1 second debounce time
   - Checks if overlay already visible

5. **Error Handling**: All operations wrapped in try-catch
   - Service errors don't crash the app
   - Falls back to programmatic view if XML fails

---

## Testing Flow

1. **Test with App Running**:
   - Make a call
   - Should see overlay (if permission granted)
   - Check logs: "CallStateModule" should appear

2. **Test with App Killed**:
   - Kill the app completely
   - Make a call
   - Should see overlay (if permission granted)
   - Check logs: "CallStateReceiver" should appear

3. **Test without Overlay Permission**:
   - Revoke overlay permission
   - Make a call
   - Should see regular modal (only if app is running)

---

## File Structure

```
android/app/src/main/
├── java/com/b2c/crm/
│   ├── CallStateModule.kt          # Detects calls (app running)
│   ├── CallStateReceiver.kt        # Detects calls (app killed)
│   ├── CallOverlayService.kt       # Shows overlay
│   └── CallOverlayModule.kt        # React Native bridge
│
└── res/
    ├── layout/
    │   └── call_overlay.xml         # Overlay UI layout
    └── drawable/
        └── status_background.xml    # Status tag background

src/
├── hooks/
│   └── useCallState.ts             # Main React Native hook
├── components/CallModals/
│   ├── CallActiveModal.tsx         # Fallback modal
│   └── CallEndedModal.tsx          # Fallback modal
└── types/native/
    ├── CallStateModule.ts          # TypeScript types
    └── CallOverlayModule.ts        # TypeScript types
```

---

## Quick Reference

### To show overlay from React Native:
```typescript
await CallOverlayModule.showCallActiveOverlay(
  phoneNumber,
  contactName,
  contactEmail,
  status
);
```

### To hide overlay:
```typescript
await CallOverlayModule.hideOverlay();
```

### To check permission:
```typescript
const hasPermission = await CallOverlayModule.checkOverlayPermission();
if (!hasPermission) {
  CallOverlayModule.requestOverlayPermission();
}
```

