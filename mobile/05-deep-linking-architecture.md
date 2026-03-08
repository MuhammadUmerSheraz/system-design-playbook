# Deep Linking Architecture

## 1. Introduction

### Purpose

Deep links open specific in-app content from external sources (web, email, ads). Universal Links (iOS) and App Links (Android) enable seamless handoff. Deferred deep linking delivers the intended destination even when the user installs the app from the link. Critical for marketing and user acquisition.

### Overview

The architecture covers link format (https vs app://), link server for resolution, platform config (AASA, assetlinks.json), attribution service for install matching, and integration with marketing platforms.

---

## 2. Requirements

### Functional Requirements

- Open app to specific screen from web link
- Fallback to web or store if app not installed
- Deferred deep link: install → first open to intended content
- Attribution: which campaign drove install
- Custom schemes and universal links
- UTM, referral codes in link params

### Non-Functional Requirements

- **Reliability:** Work across browsers, email, social
- **Security:** Validate source; prevent spoofing
- **Performance:** Minimal redirects
- **Analytics:** Click, install, conversion tracking

---

## 3. High-Level Architecture

### Components

1. **Link Format** — https://app.example.com/path or app://path
2. **Link Server** — Resolves short links; redirects by platform
3. **App Config** — apple-app-site-association, assetlinks.json
4. **Attribution Service** — Fingerprint matching, install tracking
5. **Marketing Platform** — Campaigns, link generation

### Communication Flow

- User taps link → Link server → Check app installed
- If yes: Open app with path
- If no: Redirect to store; store fingerprint + link data
- First launch: Match fingerprint → Retrieve link → Navigate

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │  User taps link │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Link Server    │
                    │  (resolve ID)   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
       ┌──────▼──────┐ ┌─────▼─────┐ ┌─────▼─────┐
       │ App         │ │ Web       │ │ App Store │
       │ installed   │ │ fallback  │ │           │
       │ → open app  │ │           │ │           │
       └──────┬──────┘ └───────────┘ └─────┬─────┘
              │                            │
              │     Install                │
              │     ┌──────────────────────┘
              │     │
       ┌──────▼─────▼──────┐
       │ Attribution       │
       │ (fingerprint      │
       │  match)           │
       └──────┬────────────┘
              │
       ┌──────▼──────┐
       │ App opens   │
       │ with deep   │
       │ link data   │
       └─────────────┘
```

---

## 5. Key Components

### Universal Links (iOS)

- https://domain.com/path
- apple-app-site-association on server
- iOS validates; opens app if installed
- Must be user tap; not programmatic

### App Links (Android)

- https://domain.com/path
- assetlinks.json or intent filters
- Digital Asset Links verify app-signing

### Link Server

- Short link → full URL + params
- Redirect by User-Agent, query params
- Log click for attribution

### Deferred Deep Linking

- Save (fingerprint, link_data) on click
- Fingerprint: device ID, IP, UA, time
- First launch: match → retrieve → navigate
- Probabilistic matching; deterministic if IDFV/GAID available

### Attribution

- Click ID, UTM params, campaign ID
- Match install to click via fingerprint or referrer
- Tools: Adjust, AppsFlyer, Branch

---

## 6. Database / Storage Design

### links

```
short_id, destination_path, campaign_id, utm_source, utm_medium, created_at
```

### clicks

```
click_id, short_id, fingerprint_hash, ip, user_agent, created_at
```

### attribution

```
install_id, click_id, campaign_id, attributed_at
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **CDN** | Serve AASA, assetlinks from CDN |
| **Caching** | Cache link resolution |
| **Async** | Click logging async |
| **Idempotency** | Dedupe attribution |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Server** | Node.js, Go; Edge (Cloudflare Workers) |
| **Attribution** | Adjust, AppsFlyer, Branch |
| **Config** | AASA, assetlinks.json |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Deferred link accuracy** | Probabilistic fingerprint; improve with IDFA/GAID |
| **iOS 14+ ATT** | Reduced deterministic match; probabilistic fallback |
| **Link hijacking** | Validate domain; use universal links (verified) |
| **Campaign overlap** | Last-click or first-click attribution model |
| **AASA caching** | CDN; reasonable TTL; validate with Apple |
