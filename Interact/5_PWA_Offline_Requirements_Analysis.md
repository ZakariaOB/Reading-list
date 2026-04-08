# PWA & Offline Requirements Analysis - Detailed Summary

**Date:** April 8, 2026  
**Sprint:** 187  
**Context:** Interact WebApp - Progressive Web App (PWA) and Offline Capabilities

---

## Table of Contents

1. [Initial Stories Analysis Findings](#1-initial-stories-analysis-findings)
2. [PWA/Offline Deep Dive](#2-pwaoffline-deep-dive)
3. [B2C Authentication with PWA](#3-b2c-authentication-with-pwa---confirmed-working)
4. [Simple PWA Implementation Path](#4-simple-pwa-implementation-path-recommended)
5. [Critical Questions for PO](#5-critical-questions-for-po-prioritized)
6. [Recommended Sprint 187 Approach](#6-recommended-sprint-187-approach)
7. [Angular-Specific Implementation Notes](#7-angular-specific-implementation-notes)
8. [Action Items & Next Steps](#8-action-items--next-steps)
9. [Key Technical Decisions Made](#9-key-technical-decisions-made)
10. [Risk Mitigation](#10-risk-mitigation)
11. [Reference Architecture](#11-reference-architecture)
12. [Email Template for PO Alignment](#12-email-template-for-po-alignment)

---

## 1. Initial Stories Analysis Findings

### Stories Reviewed:

#### **0_WebAppCreation.md**
- Standalone webapp at https://connect.bobst.com/interact
- Landing page for auth
- Take into account when connection is OFF

#### **1_Authentication.md**
- Auth using BOBST Connect credentials (email + password)
- Users see only machines/sites assigned to them in BOBST Connect
- Same authorization model as main app
- Example: eduardo.porteroperaile+admin@bobst.com should only see assigned equipment

#### **2_GetDowntimeData.md**
- Fetch custom downtime categories/reasons configured by admin
- Each time machine is down (after X seconds), show pop-up with categories/reasons
- **PWA requirement mentioned**: Cover PWA (https://developer.mozilla.org/es/docs/Web/Progressive_web_apps)
- Show only "custom reasons" created in Settings → Custom downtime
- Do NOT show "standard BOBST" reasons

#### **3_PostDowntimeData.md**
- POST downtime category + reason to BOBST Connect
- Feed the "downtime" page in Connect
- **Important**: Keep automatic fault code (don't delete it)

#### **4_HistoryList.md**
- Display list of already inserted downtime reasons
- Show: custom category, custom reason, status (opened/closed), start time, duration
- Status: closed = user inserted reason, opened = user has NOT inserted reason
- Goal: Operator checks previous shift's entries

#### **5_MakeLink.md** (Incomplete)
- Notes about default "shopfloor process defect" 
- Question about historical data handling
- If no custom reasons configured, display nothing

#### **6_DowntimePageDisplay.md**
- Display logic: If manual downtime reason exists, show it; otherwise show automatic fault code
- Manual entry takes precedence over automatic

#### **Andon/architecture.md**
- Reference architecture for separate deployment
- Separate resource group from main BOBST Connect Frontend
- Front Door routing: `/andon` → Andon RG, `/*` → Main Connect RG
- Bicep deployment scripts, pipelines structure

### Critical Questions Identified (from story analysis):

1. **Architecture**: Should Interact follow Andon deployment pattern (separate RG + Front Door routing)?
2. **API Endpoints**: Need concrete API specs for GET/POST downtime, categories, reasons, history
3. **Pop-up Trigger**: How does "machine down after X seconds" work? Real-time via WebSocket/SignalR?
4. **Custom vs Standard Reasons**: How are these differentiated in API responses?
5. **Automatic Fault Codes**: Where stored? How merged with manual entries?
6. **History Scope**: How far back? Filtered by machine or all machines?
7. **Offline/PWA Requirements**: What level of offline capability needed?
8. **No Custom Reasons**: What displays if admin hasn't configured any custom reasons?
9. **Conflict Handling**: What if same downtime edited from both Interact and BOBST Connect?
10. **Shift Handovers**: How to handle downtime spanning multiple operators?

---

## 2. PWA/Offline Deep Dive

### PWA Capability Levels Defined:

#### **Level 1: Basic Installable App (Minimal - 1-2 days)**

**Requirements:**
- manifest.json with app metadata, icons, theme colors
- Service worker registration via `ng add @angular/pwa`
- HTTPS deployment (already have via Front Door)

**What it provides:**
- ✅ Users can "install" Interact to home screen/desktop
- ✅ Standalone window experience (no browser chrome)
- ✅ App icon like native mobile app
- ✅ Fast repeat visits (cached assets)
- ❌ No offline functionality for data

**Effort:** Low (~1-2 days)

**Recommendation:** This is the baseline - provides 90% of value for 10% of effort

---

#### **Level 2: Offline-First with Read Capability (3-5 days)**

**Requirements:**
- Cache static assets (HTML, JS, CSS, images) ← Automatic with Angular PWA
- Cache API responses for viewing historical data
- Display cached data when offline with visual indicator

**Use Cases for Interact:**
- Operator can view downtime history list while offline
- App shell loads even without network
- Previously loaded categories/reasons remain visible for reference

**Technical Implementation:**
```typescript
// Service Worker Caching Strategy
- Cache-First for static assets
- Network-First with Cache Fallback for API data
- Stale-While-Revalidate for downtime history
```

**Key Questions:**
1. **How old can cached data be?** (1 hour? 8 hour shift? 24 hours?)
2. **What should display if operator opens app offline for first time?** (never cached anything)
3. **Should we show a visual indicator that data is stale/cached?**

**Technical Impact:**
- Need IndexedDB or Cache API strategy
- API response caching with TTL (Time To Live)
- UI state management for "offline mode" display
- Offline detection logic (online/offline event listeners)

**Effort:** Medium (~3-5 days)

---

#### **Level 3: Offline Write with Background Sync (5-8 days, COMPLEX)** ⭐ **CRITICAL DECISION**

**Requirements:**
- Queue downtime submissions offline (IndexedDB)
- Sync when connection restored (Background Sync API)
- Handle sync conflicts/failures
- Retry logic with exponential backoff

**Interact Scenario:**
```
Timeline:
1. Machine goes down at 10:00 AM
2. Network drops at 10:02 AM
3. Operator selects "Maintenance" > "Compressor Oil Change" at 10:05 AM
4. Data queued locally (IndexedDB/LocalStorage)
5. Network restored at 10:15 AM
6. Background Sync API sends queued data
7. Success/failure feedback to operator
```

**Critical Questions:**

**Q1: What happens if operator submits downtime offline but another operator already submitted it online?**
- First-write-wins? Last-write-wins?
- Merge with timestamp precedence?
- Show conflict resolution UI?

**Q2: What if sync fails repeatedly?**
- Retry strategy? (exponential backoff? manual retry?)
- Maximum queue size?
- Data expiration in queue after X hours/days?

**Q3: Should operators be able to submit multiple downtimes offline?**
- Yes → Complex queue management with ordering
- No → Block submission with "waiting for sync" message

**Q4: Can operators edit/delete queued items before sync?**
- Adds complexity to queue management

**Technical Stack Needed:**
```typescript
// Background Sync API
self.addEventListener('sync', async (event) => {
  if (event.tag === 'sync-downtimes') {
    event.waitUntil(syncQueuedDowntimes());
  }
});

// Fallback: Periodic sync check (when Background Sync not supported)
// Or: Manual "Retry Sync" button in UI
```

**Browser Support Concern:**
- Background Sync API: 
  - Chrome/Edge ✅ 
  - Firefox ⚠️ (limited support)
  - Safari ❌ (as of 2026, not supported)
- **Need fallback strategy for iOS operators using Safari**

**Technical Impact:**
- IndexedDB for queue persistence
- Sync queue manager service with state machine
- Conflict resolution logic
- Retry/failure handling with error states
- UI for pending sync status (badge, list of queued items)
- Testing for various failure scenarios
- **Effort:** High (~5-8 days)

**Risk Level:** HIGH - adds significant complexity

---

### Real-Time Requirements vs Offline Tension

**Major Tension:**
Story 2 states: *"Each time the machine is down (after X seconds), the user should see a pop-up"*

**Question:** How does this work offline?

#### **Option A: Offline mode disables real-time pop-ups** ✅ SIMPLE
- When offline, operator manually checks app for open downtimes
- Pop-ups only work when online with WebSocket/SignalR connection
- Clear UX: "Offline Mode - Manual Entry Only" banner

#### **Option B: Local timer-based detection** ⚠️ RISKY
- App stores machine state locally
- Local logic triggers pop-up based on cached state
- **Risk**: might not match server reality, could show false alerts

#### **Option C: Hybrid** ✅ RECOMMENDED
- **Online**: Real-time pop-ups via WebSocket/SignalR
- **Offline**: Manual mode only, no pop-ups, clear banner
- **Reconnect**: Sync state, show missed pop-ups if any

**Technical Stack Impact:**
```typescript
// Online Mode
- SignalR/WebSocket for real-time machine state
- Server pushes "machine down" events to connected clients
- Client shows pop-up immediately

// Offline Mode
- Disable real-time listeners
- Show banner: "Offline Mode - Manual Entry Only"
- Operator navigates to history, manually adds reason
- Queue submission until online (if Level 3)

// Reconnect Logic
- Re-establish WebSocket connection
- Fetch missed events since disconnect
- Update UI with current machine states
```

**Recommendation:** Implement Option C (Hybrid)

---

### PWA Storage Limits & Data Management

**Storage Available:**
- **IndexedDB**: ~50MB typical (varies by browser/OS, can request more)
- **Cache API**: ~50MB typical
- **LocalStorage**: ~5-10MB (don't use for large data sets)

**Interact Data Volumes (Estimated):**
- Downtime categories/reasons: ~50-200 KB per site
- History (last 100 entries): ~200-500 KB
- Static assets (JS/CSS/images): ~2-5 MB
- **Total**: ~3-6 MB typical usage

**Conclusion:** Storage is sufficient for Interact use case

**Best Practices:**
1. **Cache eviction strategy** (LRU - Least Recently Used? Time-based?)
2. **Limit history cache** (last N days vs all time)
3. **Clear old data** periodically to avoid bloat
4. **Monitor quota** usage and warn user if approaching limit

---

### Authentication Challenges Offline

**Problem:** JWT/OAuth tokens expire (typically 1 hour)

**Scenarios:**

#### **Scenario A: Operator goes offline mid-shift**
- Token valid for 30 more minutes at time of disconnect
- Can use app until token expires
- Then locked out until online (cannot refresh token offline)

#### **Scenario B: Operator opens app offline**
- No valid token in storage OR expired token
- Cannot authenticate (B2C requires online connection)
- App unusable - shows login screen but can't complete

**Mitigation Options:**

##### **Option 1: Refresh Token with offline grace period**
```typescript
// Token expiring soon but offline
if (isOffline && tokenExpiresIn < 5min) {
  // Allow continued use with stored credentials
  // Risk: Security concern, token might be compromised
  // Benefit: Better UX for operators
}
```
⚠️ **Security Risk** - requires approval

##### **Option 2: Extended token lifetime for Interact**
- Issue 8-hour tokens (shift duration) instead of 1-hour
- Reduces security (longer exposure if token stolen)
- Increases offline usability significantly
- **Requires B2C configuration change + security team approval**

##### **Option 3: Offline-only read mode**
- View cached data without requiring valid auth token
- Require valid auth for POST operations (submit downtimes)
- Hybrid security model

##### **Option 4: Credential caching with encryption** (Advanced)
- Cache encrypted credentials locally
- Auto-refresh token when back online
- Highest security, most complex implementation

**Key Question for Stakeholders:**
> "What's acceptable for factory floor usage?"
> - Security vs Usability tradeoff
> - Factory network stability expectations
> - Typical shift duration vs token lifetime

**Recommendation:** Start with Option 2 (extended token lifetime to 8 hours) if security approves

---

## 3. B2C Authentication with PWA - CONFIRMED WORKING

### Key Finding: **PWA installation does NOT change authentication mechanism**

✅ **MSAL.js works identically** whether browser-accessed or installed as PWA  
✅ **Same redirect URIs** in B2C configuration (`https://connect.bobst.com/interact`)  
✅ **Token persistence** - Tokens stored in localStorage/sessionStorage survive app restarts  
✅ **Shared auth session** - If user logs into BOBST Connect in browser, PWA can share tokens (same origin)

### No Special PWA Setup Required for Auth

Since Interact will reuse BOBST Connect's existing B2C tenant and app registration:

1. ✅ Register `/interact` redirect URI in B2C app registration
2. ✅ Same MSAL configuration as BOBST Connect frontend
3. ✅ Proper manifest.json with scope and start_url
4. ✅ APIM already accepts tokens from same B2C tenant
5. ✅ Same authorization claims/roles for site/machine access

**No additional B2C setup needed specifically for PWA!**

### Implementation Details

#### **manifest.json Configuration:**
```json
{
  "name": "Bobst Interact",
  "short_name": "Interact",
  "start_url": "/interact",
  "display": "standalone",  // ⚠️ Important for auth flow
  "scope": "/interact/",
  "background_color": "#ffffff",
  "theme_color": "#0066cc",
  "icons": [...]
}
```

**Why "display: standalone" matters:**
- **"standalone"** - Auth redirects open within PWA window ✅ (recommended)
- **"fullscreen"** - May cause auth redirects to open in system browser ⚠️
- **"minimal-ui"** - Shows minimal browser UI, auth works but less immersive

### Platform-Specific Auth Behavior

| Platform | Auth Redirect Behavior | Notes |
|----------|----------------------|-------|
| **Chrome/Edge (Desktop)** | Opens in PWA window ✅ | Seamless experience |
| **Chrome/Edge (Android)** | Opens in PWA window ✅ | Seamless experience |
| **Safari (iOS)** | May open in Safari browser ⚠️ | Use popup fallback |
| **Firefox (Desktop)** | Opens in PWA window ✅ | Generally works well |

### iOS Safari Workaround (if needed)

```typescript
// Configure MSAL for popup instead of redirect on iOS PWA
const msalConfig = {
  auth: {
    clientId: "your-client-id",
    authority: "https://bobstb2c.b2clogin.com/...",
    redirectUri: "https://connect.bobst.com/interact"
  },
  cache: {
    cacheLocation: "localStorage", // Persist across sessions
    storeAuthStateInCookie: true   // Helps with Safari iOS issues
  }
};

// Detect if running as PWA
const isPWA = window.matchMedia('(display-mode: standalone)').matches;
const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);

// Use popup on iOS PWA to avoid browser redirect issues
if (isPWA && isIOS) {
  await msalInstance.loginPopup(loginRequest);
} else {
  await msalInstance.loginRedirect(loginRequest); // Standard flow
}
```

### Security Considerations

**Token Storage:**
- localStorage: Persists across sessions ✅ (recommended for PWA)
- sessionStorage: Cleared when app closes ❌ (poor PWA UX)
- Cookies: Can work but less common with MSAL

**Token Refresh:**
- Online: Standard MSAL token refresh flow works
- Offline: Cannot refresh (needs offline handling strategy - see Authentication Challenges section)

### Your Existing Setup Compatibility

From your notes: *"Ensure Interact token is accepted by APIM policies"*

**What you need:**
1. ✅ Register `/interact` redirect URI in existing B2C app registration
2. ✅ Same MSAL client ID and configuration as BOBST Connect
3. ✅ APIM already configured to accept tokens from this B2C tenant
4. ✅ Same scope/permission claims for API access
5. ✅ User Management service already handles site/machine authorization

**Testing Checklist:**
- [ ] Install PWA and verify auth redirect stays in app window
- [ ] Test login flow (email + password)
- [ ] Verify token stored in localStorage
- [ ] Test API calls with token (APIM acceptance)
- [ ] Verify user sees only assigned machines (authorization)
- [ ] Test token persistence after closing/reopening PWA
- [ ] Test on iOS Safari (popup fallback if needed)

---

## 4. Simple PWA Implementation Path (Recommended)

### Step-by-Step Incremental Approach

#### **Step 1: Bare Minimum PWA (1-2 days)** ✅ START HERE

##### **What to do:**

**1. Run Angular PWA Schematic:**
```bash
ng add @angular/pwa --project=interact
```

This automatically:
- Creates `manifest.webmanifest` template
- Adds `ngsw-config.json` with default caching
- Registers service worker in `app.module.ts`
- Adds icons placeholders
- Updates `angular.json` configuration

**2. Customize manifest.webmanifest:**
```json
{
  "name": "Bobst Interact",
  "short_name": "Interact",
  "description": "Downtime reporting for BOBST Connect operators",
  "start_url": "/interact",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0066cc",
  "scope": "/interact/",
  "icons": [
    {
      "src": "/interact/assets/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/interact/assets/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable any"
    }
  ]
}
```

**3. Add to index.html:**
```html
<head>
  <!-- Existing head content -->
  <link rel="manifest" href="manifest.webmanifest">
  <meta name="theme-color" content="#0066cc">
  
  <!-- iOS specific -->
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black">
  <meta name="apple-mobile-web-app-title" content="Interact">
  <link rel="apple-touch-icon" href="/interact/assets/icons/icon-152x152.png">
</head>
```

**What this provides:**
- ✅ Install button in Chrome/Edge (desktop & mobile)
- ✅ Add to Home Screen on iOS/Android
- ✅ App icon on device
- ✅ Standalone window (no browser chrome)
- ✅ Splash screen on launch
- ✅ Automatic updates when you deploy new version

**What it does NOT provide:**
- ❌ Offline functionality for data (API calls still fail)
- ❌ Background sync
- ❌ Push notifications

**Effort:** 1-2 days (including icon generation)

---

#### **Step 2: Cache Static Files (Automatic)** ✅ FREE

Angular PWA automatically creates `ngsw-config.json`:

```json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/interact/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.webmanifest",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2)"
        ]
      }
    }
  ]
}
```

**What this does:**
- ✅ App shell (HTML/CSS/JS) loads instantly on repeat visits
- ✅ App opens even if briefly offline (but shows errors when trying to load data)
- ✅ Assets cached, no re-download on each visit
- ✅ Versioning handled automatically (new version = cache busted)

**Operators see:**
- Fast loading ⚡
- "No connection" errors when offline and trying to fetch downtime data

**Effort:** 0 days (comes free with `ng add @angular/pwa`)

---

#### **Step 3: View Cached Data Offline (Optional - 2-3 days)** ⚠️ ADD ONLY IF NEEDED

**When to add this:**
- If PO confirms operators need to VIEW history when offline
- If factory network is unreliable
- If operators work in low-signal areas

**What to add to ngsw-config.json:**

```json
{
  "dataGroups": [
    {
      "name": "api-downtime-categories",
      "urls": [
        "https://api.connect.bobst.com/api/downtime/categories/**",
        "https://api.connect.bobst.com/api/downtime/reasons/**"
      ],
      "cacheConfig": {
        "maxSize": 50,
        "maxAge": "1h",
        "timeout": "5s",
        "strategy": "freshness"
      }
    },
    {
      "name": "api-downtime-history",
      "urls": [
        "https://api.connect.bobst.com/api/downtime/history/**"
      ],
      "cacheConfig": {
        "maxSize": 100,
        "maxAge": "30m",
        "timeout": "5s",
        "strategy": "performance"
      }
    }
  ]
}
```

**Cache Strategies:**
- **"freshness"** (Network First): Try network, fall back to cache if offline
  - Use for: Categories/reasons (want latest but can tolerate stale)
- **"performance"** (Cache First): Use cache, update in background
  - Use for: History list (less critical to be real-time)

**Add offline detection in Angular:**

```typescript
// app.component.ts or shared service
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NetworkStatusService {
  private online$ = new BehaviorSubject<boolean>(navigator.onLine);

  constructor() {
    window.addEventListener('online', () => this.online$.next(true));
    window.addEventListener('offline', () => this.online$.next(false));
  }

  get isOnline$() {
    return this.online$.asObservable();
  }

  get isOnline() {
    return this.online$.value;
  }
}
```

**Show offline banner in template:**

```html
<!-- app.component.html or layout component -->
<div *ngIf="!(networkStatus.isOnline$ | async)" class="offline-banner">
  <mat-icon>cloud_off</mat-icon>
  <span>Offline Mode - Showing cached data. Submissions will sync when online.</span>
</div>

<!-- Styling -->
<style>
.offline-banner {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  background: #ff9800;
  color: white;
  padding: 8px 16px;
  display: flex;
  align-items: center;
  gap: 8px;
  z-index: 9999;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}
</style>
```

**What this provides:**
- ✅ Last API responses cached for configured duration
- ✅ If offline, shows cached data with "Offline Mode" banner
- ✅ Smooth degradation - app still usable
- ❌ Cannot submit new downtimes offline (unless Level 3)

**Effort:** 2-3 days (service worker config + UI + testing)

---

#### **What to SKIP (for now)** ❌

**Skip these unless absolutely necessary:**

1. ❌ **Background sync** (Level 3) - too complex, wait for validated need
2. ❌ **Offline writes with queue** - requires conflict resolution, retry logic
3. ❌ **Push notifications** - not in MVP requirements
4. ❌ **Custom service worker** - Angular handles it, don't reinvent
5. ❌ **Periodic background sync** - not widely supported, adds complexity
6. ❌ **Advanced caching strategies** - start simple, optimize later

---

### Simple Decision Tree

```
┌─────────────────────────────────────────────┐
│ Do operators need to use app offline?      │
└────────────┬────────────────────────────────┘
             │
        NO   │   YES
     ┌───────┴───────┐
     │               │
     ▼               ▼
┌─────────┐   ┌──────────────────────────────────┐
│ Step 1  │   │ View data or Submit data offline?│
│  only   │   └────────────┬─────────────────────┘
│         │                │
│ 1 day   │      VIEW ONLY │  SUBMIT OFFLINE
└─────────┘        ┌───────┴────────┐
                   ▼                ▼
            ┌──────────────┐  ┌────────────────┐
            │ Step 1+2+3   │  │ Level 3 Sync   │
            │              │  │                │
            │ 3-4 days     │  │ 8+ days ⚠️     │
            └──────────────┘  │ DEFER          │
                              │ Sprint 188     │
                              └────────────────┘
```

---

## 5. Critical Questions for PO (Prioritized)

### 🎯 **The ONE Question That Decides Everything:**

> **"Is it acceptable if operators cannot submit downtime reasons when offline, and they just submit them when connection is restored?"**

**If YES →** Build Level 1-3 (simple, 3-4 days total)  
**If NO →** Build Level 4 with sync queue (complex, 8+ days, needs conflict resolution)

---

### **Top 3 Priority Questions (Must Answer):**

#### **Q1: Network Reliability**
**Ask:** "How often do shop floor network drops happen on the factory floor?"

**Options:**
- □ **Rarely** (< once per month) → **Level 1 sufficient** (installable app)
- □ **Sometimes** (weekly brief drops) → **Level 2 recommended** (offline read)
- □ **Frequently** (daily or extended outages) → **Level 3 needed** (offline write)

**Follow-up:** "How long do outages typically last?"
- < 5 minutes → Operators can wait (simple solution)
- 5-60 minutes → View-only offline mode useful
- > 60 minutes → Need full offline capability

---

#### **Q2: Immediate Recording Requirement**
**Ask:** "When a machine goes down, can the operator wait 2-5 minutes to record the downtime reason?"

**Scenario:**
> "Machine stops at 10:00. Network is down. Can operator wait until 10:05 when network is back to submit the reason?"

**Options:**
- □ **Can wait** → Simple solution ✅ (no offline write needed)
- □ **Must be instant** → Complex solution ⚠️ (need offline write + sync)

**Business Impact:**
- Waiting acceptable → Save 5+ days of development
- Must be instant → Invest in offline write capability

---

#### **Q3: Data Loss Tolerance**
**Ask:** "If an operator is offline and forgets to submit the downtime reason when back online, is that acceptable?"

**Scenario:**
> "Operator enters reason while offline. App queues it to send later. But operator closes app before sync happens. Data lost. OK?"

**Options:**
- □ **Acceptable loss** → Simple, operators retry manually ✅
- □ **Cannot lose data** → Need persistent queue + auto-sync ⚠️

**Implications:**
- Acceptable loss → Manual retry button, simpler UX
- Cannot lose → Background sync, persistent queue, retry logic, conflict resolution

---

### **Secondary Questions (Should Answer):**

#### **Q4: View Cached History Offline?**
**Show scenario:**
> "Operator opens app offline. Should they see:
> - **A)** Error message: 'No connection, try again later'
> - **B)** Last hour's downtime history (read-only, cached data)"

**Ask:** "Which is better for operators?"

**Translation:**
- **A is fine** → No offline read needed ✅ (save 2 days)
- **B is needed** → Add API response caching (~2-3 days work)

---

#### **Q5: How Is the App Used?**
**Ask:**
- "Is there one shared tablet per production line? Or personal devices?"
- "Do operators carry tablets with them? Or go to a fixed station?"
- "Typical device: Android tablet? iPad? Desktop PC?"

**Why it matters:**
- **Fixed station** → Less critical (operators can walk to station when online) ✅
- **Mobile/roaming** → More critical (might be in Wi-Fi dead zone) ⚠️
- **iOS devices** → Need popup auth fallback, no Background Sync support

---

#### **Q6: Real-time Pop-up Offline Behavior**
**Show scenario:**
> "Story says: 'Machine goes down → pop-up appears automatically'
> 
> If operator is offline when machine goes down:
> - **A)** No pop-up, they manually check app and add reason later
> - **B)** Pop-up still appears based on locally stored state
> - **C)** Queue it, pop-up shows when back online"

**Ask:** "Which option makes most sense for operators?"

**Translation:**
- **A** → Simple, disable real-time when offline ✅ (recommended)
- **B** → Complex, needs local machine state sync ⚠️
- **C** → Complex, needs notification queue ⚠️

---

### Summary Decision Matrix

Present this table to PO for quick decision:

| If PO Says... | Then Build... | Effort | Risk |
|---------------|---------------|--------|------|
| "Network is stable, outages rare" | **Level 1**: Installable app only | 1-2 days | Low ✅ |
| "Brief drops happen, operators can wait" | **Level 1+2**: Installable + fast loading | 2-3 days | Low ✅ |
| "Operators need to VIEW history offline" | **Level 1-3**: Add read-only cache | 3-4 days | Medium ⚠️ |
| "Operators MUST submit even offline" | **Level 4**: Add offline write + sync queue | 8+ days | High 🔴 |

---

### Simple Email/Meeting Script

```text
Subject: Interact PWA - Offline Requirements Alignment Needed

Hi [PO Name],

For Sprint 187 Interact development, I need quick alignment on offline capabilities 
to properly scope the PWA implementation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ONE KEY QUESTION (decides 50% of effort):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"Is it acceptable if operators cannot submit downtime reasons when offline, 
and they submit them later when connection is restored?"

□ YES - Simple path (3-4 days total PWA work) ✅
□ NO  - Complex path (8+ days, needs background sync queue) ⚠️

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CONTEXT:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

If YES: 
• App installs to home screen (like mobile app)
• Loads fast with offline-capable shell
• Works great when online
• When offline, operators see "No connection" and wait
• Can optionally view cached history (read-only)

If NO:  
• Need to build: offline submission queue, auto-sync when online,
  conflict resolution, retry logic
• Significant complexity and testing requirements
• iOS Safari doesn't support Background Sync (need fallback)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FOLLOW-UP QUESTIONS (if you have 5 more minutes):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. How often do factory floor network drops happen?
   □ Rarely (< once/month)
   □ Sometimes (weekly)
   □ Frequently (daily)

2. Should operators VIEW cached history when offline?
   □ Yes (add 2 days work)
   □ No, error message is fine

3. When machine goes down, can operator wait 2-5 min to submit?
   □ Yes, waiting is fine
   □ No, must be instant

Let me know and I'll plan Sprint 187 accordingly!

Thanks,
[Your Name]
```

---

## 6. Recommended Sprint 187 Approach

### **Phase 1: Weeks 1 - Build Foundation (Days 1-5)**

#### **Day 1-2: PWA Baseline Setup**
- [ ] Run `ng add @angular/pwa --project=interact`
- [ ] Design and generate app icons (72x72 to 512x512)
  - Use tool like https://realfavicongenerator.net/ or Figma
  - Ensure "maskable" icons for Android
- [ ] Customize `manifest.webmanifest` with Bobst branding
- [ ] Add theme-color and Apple-specific meta tags to index.html
- [ ] Build and deploy to feature environment
- [ ] Test installation on:
  - [ ] Chrome Desktop (Windows)
  - [ ] Edge Desktop (Windows)
  - [ ] Chrome Android (if available)
  - [ ] Safari iOS (if available)

**Success Criteria:**
- ✅ Install button appears in browser
- ✅ App installs to desktop/home screen
- ✅ App opens in standalone window (no browser chrome)
- ✅ Correct icon and name display

---

#### **Day 3: B2C Authentication in PWA**
- [ ] Verify MSAL configuration works in installed PWA
- [ ] Test login flow (email + password)
- [ ] Verify redirect stays within PWA window (not opening browser)
- [ ] Test token persistence (close app, reopen, still logged in)
- [ ] Test on multiple platforms
- [ ] Add iOS popup fallback if needed

**Success Criteria:**
- ✅ Login works in installed PWA
- ✅ Auth redirect doesn't break PWA experience
- ✅ Token persists across app restarts
- ✅ User sees only assigned machines (authorization check)

---

#### **Day 4-5: Offline Detection & Basic UX**
- [ ] Create `NetworkStatusService` with online/offline detection
- [ ] Add offline banner component
- [ ] Show banner when connection drops
- [ ] Hide banner when connection restored
- [ ] Test: Disconnect Wi-Fi, verify banner appears
- [ ] Test: API calls fail gracefully with user-friendly error

**Success Criteria:**
- ✅ "Offline Mode" banner appears when no connection
- ✅ Banner disappears when connection restored
- ✅ Graceful error messages (not generic HTTP errors)

---

### **Phase 2: Week 2 - Evaluate & Decide (Days 6-10)**

#### **Day 6-7: PO Alignment Meeting**
- [ ] Schedule 30-min meeting with PO
- [ ] Present PWA demo (installed app)
- [ ] Walk through offline scenarios
- [ ] Answer critical questions (see Section 5)
- [ ] **Document decisions clearly**
- [ ] Update Sprint backlog based on decisions

**Decisions Needed:**
- [ ] Offline write requirement? (YES/NO)
- [ ] View cached data? (YES/NO)
- [ ] Real-time pop-up offline behavior?
- [ ] Token lifetime extension approval?

---

#### **Day 8-9: Implement Step 3 IF Approved**
**Only if PO confirms need for offline data viewing:**

- [ ] Add `dataGroups` to `ngsw-config.json`
- [ ] Configure caching for categories/reasons API
- [ ] Configure caching for history API
- [ ] Test: Load data online, go offline, verify cached data displays
- [ ] Add "Showing cached data" indicator in UI
- [ ] Add cache age display (e.g., "Last updated 5 minutes ago")

**Success Criteria:**
- ✅ History list shows cached data when offline
- ✅ Categories/reasons available offline
- ✅ Clear UI indicator that data is cached

---

#### **Day 10: Testing & Documentation**
- [ ] E2E Cypress tests for PWA scenarios
- [ ] Test install flow
- [ ] Test offline mode
- [ ] Test auth in PWA
- [ ] Document PWA features in user guide
- [ ] Document deployment process for PWA assets

---

### **Phase 3: Sprint 188 - Advanced Features (Conditional)**

**Only proceed if Sprint 187 evaluation shows clear need:**

#### **If Offline Write Needed (8+ days):**
- Week 1: Design sync queue architecture
- Week 2: Implement IndexedDB queue + Background Sync API
- Week 3: Implement conflict resolution + retry logic
- Week 4: Testing, error handling, fallbacks

**Risk:** High complexity, recommend deferring unless critical

---

### Testing Strategy for Sprint 187

#### **Manual Testing Checklist:**

**PWA Installation:**
- [ ] Chrome Desktop (Windows) - Install and launch
- [ ] Edge Desktop (Windows) - Install and launch
- [ ] Chrome Android - Add to Home Screen
- [ ] Safari iOS - Add to Home Screen
- [ ] Verify correct icon at all sizes
- [ ] Verify app name display
- [ ] Verify standalone mode (no browser UI)

**Authentication:**
- [ ] Login in browser, then install → Verify still logged in
- [ ] Install app, then login → Verify works
- [ ] Close app, reopen → Verify session persisted
- [ ] Logout → Verify cleared
- [ ] Test on iOS Safari (popup flow if needed)

**Offline Mode:**
- [ ] Load app online → Disconnect Wi-Fi → Verify banner appears
- [ ] Try to load new page → Verify graceful error
- [ ] Reconnect → Verify banner disappears
- [ ] Verify cached static assets still work offline (app shell)

**If Step 3 Implemented:**
- [ ] Load history online → Go offline → Verify history still displays
- [ ] Check data staleness indicator
- [ ] Go back online → Verify fresh data loads

---

#### **Cypress E2E Tests:**

```typescript
// cypress/e2e/pwa.cy.ts

describe('PWA Functionality', () => {
  it('should have valid manifest', () => {
    cy.request('/interact/manifest.webmanifest')
      .its('body')
      .should('have.property', 'name', 'Bobst Interact')
      .and('have.property', 'display', 'standalone');
  });

  it('should register service worker', () => {
    cy.visit('/interact');
    cy.window().then(win => {
      expect(win.navigator.serviceWorker).to.exist;
    });
  });

  it('should show offline banner when disconnected', () => {
    cy.visit('/interact');
    cy.window().then(win => {
      win.dispatchEvent(new Event('offline'));
    });
    cy.get('.offline-banner').should('be.visible');
  });

  it('should hide offline banner when reconnected', () => {
    cy.visit('/interact');
    cy.window().then(win => {
      win.dispatchEvent(new Event('offline'));
    });
    cy.get('.offline-banner').should('be.visible');
    
    cy.window().then(win => {
      win.dispatchEvent(new Event('online'));
    });
    cy.get('.offline-banner').should('not.exist');
  });
});
```

---

### Observability & Metrics

**Metrics to Track (if possible):**
- Install rate: % of users who install PWA vs use in browser
- Offline frequency: How often do users experience offline mode?
- Offline duration: How long are typical offline periods?
- Error rates: API failures when offline

**Tools:**
- Application Insights custom events
- Service Worker analytics (cache hit rate)

**Example:**
```typescript
// Track PWA installation
window.addEventListener('appinstalled', () => {
  appInsights.trackEvent({ name: 'PWA_Installed' });
});

// Track offline mode
window.addEventListener('offline', () => {
  appInsights.trackEvent({ name: 'Offline_Mode_Entered' });
});
```

This data will inform decisions for Sprint 188+.

---

## 7. Angular-Specific Implementation Notes

### Service Worker Configuration Details

Angular Service Worker provides:
- ✅ App shell caching (automatic)
- ✅ Asset versioning with cache busting
- ✅ Update notifications when new version deployed
- ✅ Configurable caching strategies per API route

### ngsw-config.json Deep Dive

```json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/interact/index.html",
  
  // Static assets (HTML, JS, CSS)
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",  // Download immediately on install
      "updateMode": "prefetch",   // Check for updates on every navigation
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.webmanifest",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",      // Download when first accessed
      "updateMode": "prefetch",   // Check for updates proactively
      "resources": {
        "files": [
          "/assets/**",
          "/*.(svg|cur|jpg|jpeg|png|webp|gif|otf|ttf|woff|woff2)"
        ]
      }
    }
  ],

  // API responses (dynamic data)
  "dataGroups": [
    {
      "name": "api-categories-reasons",
      "urls": [
        "https://api.connect.bobst.com/api/performance/downtime/categories",
        "https://api.connect.bobst.com/api/performance/downtime/reasons"
      ],
      "cacheConfig": {
        "maxSize": 50,           // Max 50 responses cached
        "maxAge": "1h",          // Cache valid for 1 hour
        "timeout": "5s",         // Network timeout before using cache
        "strategy": "freshness"  // Network-first, cache fallback
      },
      "cacheQueryOptions": {
        "ignoreSearch": true     // Ignore query params for cache key
      }
    },
    {
      "name": "api-downtime-history",
      "urls": [
        "https://api.connect.bobst.com/api/performance/downtime/history/**"
      ],
      "cacheConfig": {
        "maxSize": 100,
        "maxAge": "30m",
        "timeout": "3s",
        "strategy": "performance"  // Cache-first for faster loading
      }
    }
  ],

  // Navigation fallback (SPA support)
  "navigationUrls": [
    "/**",           // Match all routes
    "!/**/*.*",      // Except files with extensions
    "!/**/*__*",     // Except Angular internal routes
    "!/**/*__*/**"
  ]
}
```

### Cache Strategies Explained

#### **Strategy: "freshness" (Network-First)**
```
Request → Try Network (with timeout)
         ↓
      Success? → Return response + update cache
         ↓ No
      Return cached response (if exists)
         ↓ None
      Return error
```

**Use for:**
- Data that changes frequently
- Critical to have latest version
- Example: Downtime categories, reasons

#### **Strategy: "performance" (Cache-First)**
```
Request → Check cache
         ↓
      Found? → Return cached + update in background
         ↓ No
      Try network
         ↓
      Return response + cache it
```

**Use for:**
- Data that doesn't change often
- Fast loading more important than freshness
- Example: Historical downtime list, user profile

---

### Update Handling

Angular Service Worker detects updates automatically. Handle in `app.component.ts`:

```typescript
import { SwUpdate, VersionReadyEvent } from '@angular/service-worker';
import { filter } from 'rxjs/operators';

export class AppComponent implements OnInit {
  constructor(private swUpdate: SwUpdate) {}

  ngOnInit() {
    // Check for updates every 6 hours
    if (this.swUpdate.isEnabled) {
      interval(6 * 60 * 60 * 1000).subscribe(() => {
        this.swUpdate.checkForUpdate();
      });
    }

    // Listen for available updates
    this.swUpdate.versionUpdates
      .pipe(filter((evt): evt is VersionReadyEvent => evt.type === 'VERSION_READY'))
      .subscribe(evt => {
        if (confirm('New version available. Load new version?')) {
          window.location.reload();
        }
      });
  }
}
```

**Better UX:** Show snackbar instead of alert:

```typescript
this.swUpdate.versionUpdates
  .pipe(filter((evt): evt is VersionReadyEvent => evt.type === 'VERSION_READY'))
  .subscribe(evt => {
    const snackBarRef = this.snackBar.open(
      'New version available!', 
      'Update',
      { duration: 0 }
    );
    
    snackBarRef.onAction().subscribe(() => {
      window.location.reload();
    });
  });
```

---

### Checking Service Worker Status

```typescript
import { SwUpdate } from '@angular/service-worker';

export class DebugComponent {
  constructor(private swUpdate: SwUpdate) {}

  checkServiceWorkerStatus() {
    console.log('Service Worker enabled:', this.swUpdate.isEnabled);
    
    this.swUpdate.versionUpdates.subscribe(evt => {
      console.log('Version update event:', evt);
    });
  }

  async forceUpdate() {
    try {
      const updateAvailable = await this.swUpdate.checkForUpdate();
      console.log('Update available:', updateAvailable);
      
      if (updateAvailable) {
        await this.swUpdate.activateUpdate();
        window.location.reload();
      }
    } catch (err) {
      console.error('Update check failed:', err);
    }
  }
}
```

---

### Background Sync (Level 3 - If Needed)

⚠️ **Angular Service Worker does NOT support Background Sync API directly.**

**Options:**

#### **Option A: Use @ngx-pwa/offline library**
```bash
npm install @ngx-pwa/offline
```

```typescript
import { Queue } from '@ngx-pwa/offline';

export class DowntimeService {
  private queue = this.queueService.queue('downtimes');

  constructor(private queueService: Queue) {}

  async submitDowntime(data: DowntimeEntry) {
    if (navigator.onLine) {
      return this.http.post('/api/downtime', data).toPromise();
    } else {
      // Queue for later
      await this.queue.add(data);
      return { queued: true };
    }
  }

  // Process queue when back online
  async processQueue() {
    const items = await this.queue.getAll();
    for (const item of items) {
      try {
        await this.http.post('/api/downtime', item).toPromise();
        await this.queue.remove(item.id);
      } catch (err) {
        console.error('Sync failed:', err);
      }
    }
  }
}
```

#### **Option B: Custom Service Worker**
Create `custom-service-worker.js` alongside Angular's:

```javascript
// custom-service-worker.js
self.addEventListener('sync', event => {
  if (event.tag === 'sync-downtimes') {
    event.waitUntil(syncDowntimes());
  }
});

async function syncDowntimes() {
  const db = await openIndexedDB();
  const pendingDowntimes = await db.getAll('pending');
  
  for (const downtime of pendingDowntimes) {
    try {
      await fetch('/api/downtime', {
        method: 'POST',
        body: JSON.stringify(downtime),
        headers: { 'Content-Type': 'application/json' }
      });
      await db.delete('pending', downtime.id);
    } catch (err) {
      console.error('Sync failed:', err);
      // Will retry on next sync
    }
  }
}
```

Register in `app.module.ts`:
```typescript
ServiceWorkerModule.register('custom-service-worker.js', {
  enabled: environment.production
})
```

⚠️ **Recommendation:** Defer Background Sync to Sprint 188. Start with simple manual retry button.

---

### IndexedDB for Offline Queue (If Level 3)

Use Angular's IndexedDB wrapper or native API:

```typescript
import { openDB, DBSchema, IDBPDatabase } from 'idb';

interface InteractDB extends DBSchema {
  'pending-downtimes': {
    key: string;
    value: {
      id: string;
      machineId: string;
      category: string;
      reason: string;
      timestamp: Date;
      synced: boolean;
    };
  };
}

export class OfflineQueueService {
  private db: IDBPDatabase<InteractDB>;

  async init() {
    this.db = await openDB<InteractDB>('interact-db', 1, {
      upgrade(db) {
        db.createObjectStore('pending-downtimes', { keyPath: 'id' });
      },
    });
  }

  async queueDowntime(downtime: any) {
    const entry = {
      ...downtime,
      id: crypto.randomUUID(),
      timestamp: new Date(),
      synced: false
    };
    await this.db.add('pending-downtimes', entry);
    return entry.id;
  }

  async getPending() {
    return await this.db.getAll('pending-downtimes');
  }

  async markSynced(id: string) {
    await this.db.delete('pending-downtimes', id);
  }
}
```

---

## 8. Action Items & Next Steps

### **Immediate (Before Sprint 187 Kickoff):**

**Week Before Sprint Starts:**
- [ ] Schedule 30-min meeting with PO to answer priority questions
- [ ] Present Decision Matrix (Section 5) and get clear direction
- [ ] Document PO decisions in backlog
- [ ] Update Sprint 187 scope based on decisions
- [ ] Identify devices for testing (Android tablet, iOS tablet, desktop)

**Key Decisions Needed:**
- [ ] Offline write requirement (YES/NO)
- [ ] View cached data offline (YES/NO)
- [ ] Real-time pop-up offline behavior (disable/queue/local)
- [ ] Token lifetime extension approval (1h vs 8h)

---

### **Sprint 187 Week 1: Foundation (Days 1-5)**

**PWA Setup:**
- [ ] Run `ng add @angular/pwa --project=interact`
- [ ] Design app icons (72x72 through 512x512)
  - [ ] Regular icons
  - [ ] Maskable icons for Android
  - [ ] Apple touch icons
- [ ] Customize `manifest.webmanifest` with Bobst branding
- [ ] Add theme-color and Apple meta tags
- [ ] Configure `ngsw-config.json` asset groups
- [ ] Build and deploy to feature environment

**Testing:**
- [ ] Test install on Chrome Desktop (Windows)
- [ ] Test install on Edge Desktop (Windows)
- [ ] Test install on Chrome Android (if device available)
- [ ] Test install on Safari iOS (if device available)
- [ ] Verify standalone mode (no browser chrome)
- [ ] Verify correct icon display at all sizes

**B2C Auth:**
- [ ] Verify MSAL works in installed PWA
- [ ] Test login flow in PWA
- [ ] Test auth redirect stays in PWA window
- [ ] Test token persistence across app restarts
- [ ] Add iOS popup fallback if needed
- [ ] Test on multiple platforms

**Offline Detection:**
- [ ] Create `NetworkStatusService`
- [ ] Create offline banner component
- [ ] Wire up online/offline event listeners
- [ ] Test: Disconnect Wi-Fi → Banner appears
- [ ] Test: Reconnect → Banner disappears
- [ ] Add graceful API error handling

---

### **Sprint 187 Week 2: Evaluation & Decision (Days 6-10)**

**PO Alignment:**
- [ ] Demo installed PWA to PO
- [ ] Walk through offline scenarios
- [ ] Get decisions on all priority questions
- [ ] Document decisions in Confluence/Wiki
- [ ] Update Sprint backlog for remaining days

**Conditional Implementation:**
- [ ] IF offline read approved: Add dataGroups to ngsw-config.json
- [ ] IF offline read approved: Configure API caching
- [ ] IF offline read approved: Add cache staleness UI indicator
- [ ] Test offline data viewing thoroughly

**Testing & Documentation:**
- [ ] Write Cypress E2E tests for PWA
- [ ] Test install flow automated
- [ ] Test offline mode scenarios
- [ ] Document PWA features in user guide
- [ ] Document deployment process for PWA assets
- [ ] Update README with PWA information

---

### **Sprint 188 (Conditional):**

**Only IF PO Confirms Need for Offline Write:**

**Week 1: Architecture & Design**
- [ ] Design sync queue architecture (sequence diagrams)
- [ ] Design conflict resolution strategy
- [ ] Design retry logic and error handling
- [ ] Review design with team
- [ ] Get security approval for data storage

**Week 2: Implementation**
- [ ] Implement IndexedDB queue service
- [ ] Implement Background Sync API integration
- [ ] Implement fallback for Safari/iOS
- [ ] Implement queue UI (pending items list)

**Week 3: Sync Logic & Error Handling**
- [ ] Implement sync on reconnect logic
- [ ] Implement conflict detection
- [ ] Implement retry with exponential backoff
- [ ] Implement failure notifications
- [ ] Implement manual retry button

**Week 4: Testing & Polish**
- [ ] E2E tests for offline write scenarios
- [ ] Test: Submit offline → Reconnect → Verify synced
- [ ] Test: Multiple queued items
- [ ] Test: Sync failure scenarios
- [ ] Test: Conflict scenarios
- [ ] Load testing with large queue
- [ ] Cross-browser testing
- [ ] User acceptance testing

---

### **Future Enhancements (Post-188):**

**Nice-to-Have Features:**
- [ ] Push notifications for machine down events
- [ ] Periodic background sync (check for updates while app closed)
- [ ] Advanced caching strategies (predictive prefetch)
- [ ] Offline analytics (track usage patterns)
- [ ] App shortcuts (quick actions from home screen icon)
- [ ] Share target (receive data from other apps)
- [ ] Badge API (show notification count on icon)

---

## 9. Key Technical Decisions Made

### ✅ **Confirmed Decisions:**

1. **PWA + B2C Auth Compatibility**
   - **Decision:** Confirmed compatible, no special setup needed
   - **Rationale:** MSAL.js works identically in PWA and browser
   - **Action:** Just register `/interact` redirect URI in existing B2C app

2. **Incremental Approach**
   - **Decision:** Start simple (Level 1), add complexity only if proven necessary
   - **Rationale:** Avoid over-engineering, validate requirements with real usage
   - **Action:** Build → Observe → Decide, not Big Bang

3. **Angular PWA Package**
   - **Decision:** Use `@angular/pwa` package, don't build custom service worker
   - **Rationale:** Mature, well-tested, automatic updates, less maintenance
   - **Action:** `ng add @angular/pwa` for baseline

4. **Testing Strategy**
   - **Decision:** Cypress E2E for PWA install and offline scenarios
   - **Rationale:** Can simulate offline mode, test install flow, cross-browser
   - **Action:** Add PWA test suite to existing Cypress setup

5. **Real-Time Offline Behavior**
   - **Decision:** Disable real-time pop-ups when offline (Option C: Hybrid)
   - **Rationale:** Simpler, avoids local state sync complexity
   - **Action:** Show "Offline - Manual Entry Only" banner, disable WebSocket

6. **Deployment Pattern**
   - **Decision:** Follow Andon architecture (separate RG, Front Door routing)
   - **Rationale:** Proven pattern, isolation, independent deployment
   - **Action:** Create Bicep templates based on Andon reference

---

### ❓ **Pending Decisions (Need PO Input):**

1. **Offline Write Requirement**
   - **Question:** Can operators wait to submit when online?
   - **Options:** 
     - A) YES → Simple (no offline write) ✅
     - B) NO → Complex (sync queue) ⚠️
   - **Status:** AWAITING PO DECISION
   - **Impact:** 5+ days development difference

2. **View Cached Data Offline**
   - **Question:** Should operators see history when offline?
   - **Options:**
     - A) NO → Just error message ✅
     - B) YES → Add caching (+2 days) ⚠️
   - **Status:** AWAITING PO DECISION
   - **Impact:** 2-3 days development

3. **Token Lifetime Extension**
   - **Question:** Extend JWT tokens from 1 hour to 8 hours for factory use?
   - **Options:**
     - A) YES → Better offline UX (needs security approval) ⚠️
     - B) NO → Operators re-auth more frequently ✅
   - **Status:** AWAITING SECURITY APPROVAL
   - **Impact:** UX quality, security posture

4. **Network Reliability Baseline**
   - **Question:** How stable is factory floor network?
   - **Action:** Measure during Sprint 187 with operators
   - **Status:** TO BE MEASURED
   - **Impact:** Informs Level 3 necessity

---

## 10. Risk Mitigation

### **Risk 1: Over-engineering Offline Features**

**Risk:** Building complex offline sync queue that nobody needs

**Probability:** Medium  
**Impact:** High (5+ days wasted)

**Mitigation:**
- ✅ Start with Level 1 (minimal)
- ✅ Validate requirements with real operator usage in Sprint 187
- ✅ Measure actual network stability on factory floor
- ✅ Get explicit PO approval before Level 3 work
- ✅ Defer complex features to Sprint 188 after validation

**Detection:**
- Surveys: Ask operators "did offline mode help you?"
- Telemetry: Track offline frequency and duration
- Support tickets: Monitor complaints about connectivity

---

### **Risk 2: iOS Safari Compatibility Issues**

**Risk:** PWA behaves differently on Safari, especially auth and Background Sync

**Probability:** High (known Safari limitations)  
**Impact:** Medium (affects iOS tablet users)

**Mitigation:**
- ✅ Test on real iOS device early (Week 1)
- ✅ Use MSAL popup mode for iOS (instead of redirect)
- ✅ Document Safari limitations for PO
- ✅ Plan fallback: Manual sync button instead of Background Sync
- ✅ storeAuthStateInCookie: true in MSAL config

**Known Safari Limitations:**
- Background Sync API not supported (as of 2026)
- Periodic Background Sync not supported
- Some Cache API quirks
- Service Worker lifecycle differences

**Fallback Plan:**
- For Background Sync → Manual "Retry Sync" button
- For Push Notifications → Web notifications only (when app open)

---

### **Risk 3: Operator Confusion with Offline Mode**

**Risk:** Operators don't understand offline mode, submit fails silently, data lost

**Probability:** Medium  
**Impact:** High (critical data loss, operator frustration)

**Mitigation:**
- ✅ Clear visual indicators (prominent offline banner)
- ✅ Simple UX: "You are offline. Features limited."
- ✅ Block submissions with clear message vs silent failure
- ✅ If queuing: Show "3 items waiting to sync" with list
- ✅ Success feedback: "Submitted successfully" when online
- ✅ Error feedback: "Submission failed, saved for retry"
- ✅ User training: Document offline behavior in user manual
- ✅ UAT with real operators before production

**UX Best Practices:**
```html
<!-- Good: Clear, actionable -->
<div class="offline-banner">
  <icon>⚠️</icon>
  <text>You are offline. You can view history but cannot submit new downtimes.</text>
</div>

<!-- Bad: Technical, unclear -->
<div class="error">
  <text>NetworkError: Failed to fetch</text>
</div>
```

---

### **Risk 4: Complex Sync Conflicts**

**Risk:** Two operators submit same downtime while one is offline, causing duplicates or conflicts

**Probability:** Low (if offline write not implemented)  
**Impact:** High (data integrity issues)

**Mitigation:**
- ✅ **Avoid Level 4 (offline write) unless absolutely necessary** ← Primary mitigation
- ✅ If needed: Implement idempotency keys (UUID per submission)
- ✅ Backend checks: Reject duplicate downtime for same machine+timespan
- ✅ Conflict resolution UI: Show both entries, let operator choose
- ✅ Last-write-wins with timestamp precedence (document tradeoff)

**Alternative Approach:**
- Don't allow offline write at all
- Simple pattern: "Cannot submit offline. Please wait for connection."
- Operators accept this limitation vs complexity

---

### **Risk 5: Service Worker Update Issues**

**Risk:** New version deployed but users stuck on old version

**Probability:** Low (Angular handles well)  
**Impact:** Medium (users don't get bug fixes/features)

**Mitigation:**
- ✅ Implement update notification (snackbar)
- ✅ Prompt user to reload: "New version available"
- ✅ Background update check every 6 hours
- ✅ Force update on critical security fixes
- ✅ Skip waiting: Activate new SW immediately on reload
- ✅ Test update flow in Cypress

**Code:**
```typescript
this.swUpdate.versionUpdates.subscribe(evt => {
  if (evt.type === 'VERSION_READY') {
    // Prompt user
    const snackBar = this.snackBar.open('Update available', 'Reload');
    snackBar.onAction().subscribe(() => window.location.reload());
  }
});
```

---

### **Risk 6: Storage Quota Exceeded**

**Risk:** IndexedDB/Cache fills up, new data can't be cached

**Probability:** Low (Interact data is small)  
**Impact:** Low (degrades to online-only)

**Mitigation:**
- ✅ Limit cache sizes in ngsw-config.json (maxSize: 50-100)
- ✅ maxAge expiration to clean old data
- ✅ Monitor quota usage in DevTools
- ✅ Graceful degradation: If storage full, just don't cache (app still works online)
- ✅ Clear old cache on version updates

**Monitoring:**
```typescript
if ('storage' in navigator && 'estimate' in navigator.storage) {
  const estimate = await navigator.storage.estimate();
  console.log(`Using ${estimate.usage} of ${estimate.quota} bytes`);
  
  if (estimate.usage / estimate.quota > 0.8) {
    console.warn('Storage nearly full, consider clearing cache');
  }
}
```

---

## 11. Reference Architecture

### Deployment Pattern (Following Andon Model)

```
Azure Resource Structure:
┌─────────────────────────────────────────────────┐
│  Resource Group: rg-interact-{env}              │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  Storage Account: stinteract{env}{random} │ │
│  │  - Static Website Hosting                 │ │
│  │  - /interact/index.html                   │ │
│  │  - /interact/assets/**                    │ │
│  │  - /interact/manifest.webmanifest         │ │
│  └───────────────────────────────────────────┘ │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  CDN Endpoint (optional)                  │ │
│  │  - Caching rules                          │ │
│  │  - HTTPS certificate                      │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Front Door Configuration

```
Front Door: fd-bobst-connect-{env}
┌─────────────────────────────────────────────┐
│                                             │
│  Route Rules:                               │
│                                             │
│  /interact/*  ───►  Interact Storage        │
│                     rg-interact-{env}       │
│                     stinteract{env}         │
│                                             │
│  /andon/*     ───►  Andon Storage           │
│                     rg-andon-{env}          │
│                     standon{env}            │
│                                             │
│  /*           ───►  Main Connect Storage    │
│                     rg-frontend-{env}       │
│                     stconnect{env}          │
│                                             │
└─────────────────────────────────────────────┘
```

### Project Structure

```
interact-frontend/
├── deployment/
│   ├── ci/
│   │   ├── templates/
│   │   │   ├── build.yml
│   │   │   ├── deploy.yml
│   │   │   └── test.yml
│   │   ├── feature-pipeline.yml
│   │   ├── develop-pipeline.yml
│   │   ├── master-pipeline.yml
│   │   └── release-pipeline.yml
│   │
│   ├── bicep/
│   │   ├── main.bicep              # Entry point
│   │   ├── modules/
│   │   │   ├── storage.bicep       # Storage account
│   │   │   ├── frontdoor.bicep     # Front Door rule
│   │   │   └── cdn.bicep           # CDN (optional)
│   │   └── parameters/
│   │       ├── feature.bicepparam
│   │       ├── develop.bicepparam
│   │       └── master.bicepparam
│   │
│   ├── deploy.ps1                  # Deployment script
│   ├── destroy.ps1                 # Cleanup script
│   └── generate-config.ps1         # config.runtime.json generation
│
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── auth/
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── auth.guard.ts
│   │   │   │   └── msal.config.ts
│   │   │   ├── services/
│   │   │   │   ├── network-status.service.ts
│   │   │   │   ├── offline-queue.service.ts  # If Level 3
│   │   │   │   └── api.service.ts
│   │   │   └── interceptors/
│   │   │       └── auth.interceptor.ts
│   │   │
│   │   ├── features/
│   │   │   ├── downtime/
│   │   │   │   ├── downtime-entry/
│   │   │   │   ├── downtime-history/
│   │   │   │   └── downtime.service.ts
│   │   │   └── machines/
│   │   │       └── machine-selection/
│   │   │
│   │   ├── shared/
│   │   │   ├── components/
│   │   │   │   ├── offline-banner/
│   │   │   │   └── loading-spinner/
│   │   │   └── models/
│   │   │
│   │   └── app.component.ts
│   │
│   ├── assets/
│   │   ├── icons/                  # PWA icons
│   │   ├── i18n/                   # Translations
│   │   └── config/
│   │       └── config.runtime.json # Runtime config
│   │
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.prod.ts
│   │
│   ├── index.html
│   ├── manifest.webmanifest
│   └── ngsw-config.json            # Service Worker config
│
├── cypress/
│   ├── e2e/
│   │   ├── pwa.cy.ts
│   │   ├── offline.cy.ts
│   │   └── auth.cy.ts
│   └── support/
│
├── angular.json
├── package.json
├── tsconfig.json
└── README.md
```

### Technology Stack Summary

| Component | Technology | Notes |
|-----------|------------|-------|
| **Frontend Framework** | Angular 15+ | Existing standard |
| **PWA** | @angular/pwa | Service Worker, caching |
| **Authentication** | MSAL.js + Azure AD B2C | Existing B2C tenant |
| **Deployment** | Azure Storage Static Website | Like Andon |
| **Routing** | Azure Front Door | Single domain, multiple backends |
| **CI/CD** | Azure DevOps Pipelines | Existing setup |
| **IaC** | Bicep | For resource deployment |
| **Testing** | Cypress | E2E tests |
| **Offline Storage** | IndexedDB (if Level 3) | Via idb library |
| **Real-Time** | SignalR/WebSocket | For machine state push |

### API Endpoints (Estimated)

Need to confirm with backend team:

```typescript
// Authentication (existing)
POST /api/auth/login
POST /api/auth/logout
POST /api/auth/refresh

// User & Authorization (existing)
GET /api/users/me
GET /api/users/me/sites
GET /api/users/me/machines

// Downtime Categories & Reasons (new or existing?)
GET /api/performance/downtime/categories?siteId={siteId}
GET /api/performance/downtime/reasons?categoryId={categoryId}&siteId={siteId}

// Downtime Submission (new or existing?)
POST /api/performance/downtime/entries
{
  "machineId": "string",
  "categoryId": "string",
  "reasonId": "string",
  "timestamp": "2026-04-08T10:00:00Z",
  "duration": 300,  // seconds
  "notes": "string (optional)"
}

// Downtime History (new or existing?)
GET /api/performance/downtime/entries?machineId={machineId}&since={timestamp}
Response: Array of {
  "id": "string",
  "machineId": "string",
  "category": "string",
  "reason": "string",
  "timestamp": "2026-04-08T10:00:00Z",
  "duration": 300,
  "status": "open" | "closed",
  "enteredBy": "user@email.com"
}

// Machine State (existing via SignalR?)
WebSocket: wss://api.connect.bobst.com/hubs/machine-state
Events:
- MachineDown: { machineId, timestamp }
- MachineUp: { machineId, timestamp }
```

**Action Item:** Validate these endpoints with PerformanceManagementService team

---

## 12. Email Template for PO Alignment

```text
To: [PO Name, Stakeholders]
Cc: [Team Lead, Architects]
Subject: Interact PWA - Offline Requirements Alignment Needed for Sprint 187
Priority: High

Hi [PO Name],

For Sprint 187 Interact development, we need to align on PWA offline capabilities 
to properly scope the implementation. This decision affects 50% of the development effort.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 ONE KEY DECISION (Affects 5+ Days of Development)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"Is it acceptable if operators cannot submit downtime reasons when offline, 
and they submit them later when connection is restored?"

□ YES - Simple path (3-4 days total PWA work) ✅ Recommended
□ NO  - Complex path (8+ days, needs background sync queue, conflict resolution)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 CONTEXT & IMPLICATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

If YES (Operators can wait):
✅ App installs to home screen/desktop (like mobile app)
✅ Fast loading with offline-capable shell
✅ Works great when online with real-time pop-ups
✅ When offline: Shows "No connection" banner, submission disabled
✅ Optionally: Can view cached history (read-only) when offline
⏱️  Effort: 3-4 days total

If NO (Must submit offline):
✅ All of the above PLUS:
⚠️  Offline submission queue (IndexedDB)
⚠️  Auto-sync when connection restored
⚠️  Conflict resolution (if another operator already submitted)
⚠️  Retry logic for failed syncs
⚠️  Complex testing scenarios
⚠️  iOS Safari doesn't support Background Sync (need fallback)
⏱️  Effort: 8+ days total
🔴 Risk: High complexity

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 FOLLOW-UP QUESTIONS (5 Minutes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

These help us fine-tune the implementation:

1️⃣ How often do factory floor network drops happen?
   □ Rarely (< once/month) → Simple PWA sufficient
   □ Sometimes (weekly brief drops) → Consider offline read
   □ Frequently (daily or extended outages) → May need offline write

2️⃣ Should operators VIEW cached history when offline?
   □ Yes (+2 days work) - Shows last hour's data when offline
   □ No, error message is fine - Simpler, faster to deliver

3️⃣ When machine goes down, can operator wait 2-5 min to submit?
   □ Yes, waiting is fine → Simplest solution
   □ No, must be instant → Need offline write capability

4️⃣ What devices will operators use?
   □ Android tablets → Full PWA support ✅
   □ iOS tablets → Limited support, need workarounds ⚠️
   □ Desktop PCs → Full support ✅
   □ Mix of all → Design for lowest common denominator

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📅 PROPOSED APPROACH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Sprint 187 Week 1:
• Build basic installable PWA (Level 1)
• Implement B2C authentication
• Add offline detection banner
• Test on target devices

Sprint 187 Week 2:
• Alignment meeting with you (30 min)
• Conditional: Add offline read caching if needed (Level 2)
• Testing and documentation

Sprint 188 (Only if necessary):
• Offline write + sync queue (Level 3)
• Only proceed if Sprint 187 data shows clear need

This approach lets us validate requirements before investing in complex features.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎬 NEXT STEPS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Can we schedule a 30-minute meeting this week to:
1. Answer the key decision question above
2. Walk through the 4 follow-up questions
3. Align on Sprint 187 scope

Tentative slots:
• [Date/Time Option 1]
• [Date/Time Option 2]
• [Date/Time Option 3]

Or reply with your answers directly via email and we can proceed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📎 ATTACHMENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Detailed analysis document: [Link to this document]
PWA capability matrix: [Link if separate]
Andon reference architecture: [Link to wiki]

Thanks for your quick turnaround on this!

Best regards,
[Your Name]
[Your Role]
[Contact Info]
```

---

## Summary & Conclusion

### Key Takeaways:

1. **Start Simple**: Level 1 PWA (installable app, 1-2 days) provides 90% of value
2. **Validate First**: Don't build complex offline sync until real need is proven
3. **Incremental Approach**: Build → Observe → Decide, not Big Bang
4. **B2C Works**: No special setup needed for PWA authentication
5. **PO Decision Critical**: Offline write vs online-only determines 5+ days of work

### Recommended Path:

```
Sprint 187 Week 1:
└─ Build Level 1 (installable PWA)
   └─ Add offline detection
      └─ Test on target devices

Sprint 187 Week 2:
└─ Get PO decision on offline features
   └─ Conditionally add Level 2 (read cache) if needed
      └─ Document and test

Sprint 188 (Only if validated need):
└─ Build Level 3 (offline write + sync) if critical
```

### Success Criteria:

✅ Operators can install Interact like mobile app  
✅ Auth works seamlessly in PWA  
✅ Clear offline mode indication  
✅ Graceful degradation when offline  
✅ Fast loading on repeat visits  
✅ Informed decision on advanced features  

### Next Action:

**Schedule 30-min PO meeting to answer ONE question:**
> "Can operators wait to be online to submit downtimes?"

Everything else follows from this decision.

---

**Document Version:** 1.0  
**Last Updated:** April 8, 2026  
**Author:** Development Team  
**Status:** Ready for PO Review
