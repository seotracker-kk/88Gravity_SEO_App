# 88Gravity SEO App — Firebase Setup Guide

> **Time required:** ~20–30 minutes  
> **Skill level:** No coding required — just follow the steps in order.

---

## Overview

The `88Gravity_SEO_App.html` file is a fully self-contained web app that needs a Firebase project as its backend. Firebase provides:

- **Google Sign-In** (restricted to @88gravity.com accounts)
- **Firestore** (real-time database for tasks, status, notes)
- **Firebase Hosting** (optional — to give the app a proper URL)

This guide walks you through every step from zero to a live, working app.

---

## Step 1 — Create a Firebase Project

1. Go to **[https://console.firebase.google.com](https://console.firebase.google.com)** and sign in with your Google account (any Google account is fine for setup — ideally use an 88Gravity account).

2. Click **"Add project"**.

3. Enter a project name — e.g., **`88gravity-seo-tracker`**. Click **Continue**.

4. On the Google Analytics screen — you can disable it (toggle off). Click **Create project**.

5. Wait ~30 seconds, then click **Continue** when the project is ready.

---

## Step 2 — Register a Web App

1. On the Firebase project homepage, click the **Web** icon (`</>`) under "Get started by adding Firebase to your app".

2. Enter an app nickname — e.g., **`SEO Tracker`**.

3. **Check the box** for "Also set up Firebase Hosting for this app" (only if you want a hosted URL — optional but recommended).

4. Click **Register app**.

5. You'll see a code block like this — **copy these values**, you'll need them in Step 5:

```javascript
const firebaseConfig = {
  apiKey:            "AIzaSy...",
  authDomain:        "88gravity-seo-tracker.firebaseapp.com",
  projectId:         "88gravity-seo-tracker",
  storageBucket:     "88gravity-seo-tracker.appspot.com",
  messagingSenderId: "123456789012",
  appId:             "1:123456789012:web:abc123def456"
};
```

6. Click **Continue to console**.

---

## Step 3 — Enable Google Sign-In

1. In the left sidebar, click **Build → Authentication**.

2. Click **Get started**.

3. Under **Sign-in method**, click **Google**.

4. Toggle **Enable** to ON.

5. Set the **Project support email** to `kuldeep@88gravity.com`.

6. Click **Save**.

7. Still on the Authentication page, click the **Settings** tab → **Authorized domains**.

8. Confirm `localhost` and your Firebase app domain (e.g., `88gravity-seo-tracker.firebaseapp.com`) are listed. They should be there by default.

> **Note:** The app is already configured to restrict sign-in to `@88gravity.com` accounts only via `hd: "88gravity.com"` in the Google OAuth provider settings in the HTML code.

---

## Step 4 — Create the Firestore Database

1. In the left sidebar, click **Build → Firestore Database**.

2. Click **Create database**.

3. Select **Start in production mode** (we'll add the correct rules next). Click **Next**.

4. Choose a Firestore location closest to your team — e.g., **`asia-south1` (Mumbai)** for an India-based team. Click **Enable**.

5. Wait ~30 seconds for Firestore to provision.

---

## Step 5 — Set Firestore Security Rules

This is the most important step — it enforces role-based access control (RBAC) at the database level so AMs can only access their own tasks.

1. In **Firestore Database**, click the **Rules** tab.

2. Replace the existing rules with the following:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper: check if user is authenticated with an 88gravity email
    function is88gravity() {
      return request.auth != null &&
             request.auth.token.email.matches('.*@88gravity\\.com');
    }

    // Helper: check if the signed-in user is the AVP
    function isAVP() {
      return is88gravity() &&
             request.auth.token.email == 'kuldeep@88gravity.com';
    }

    // Helper: derive amId from email (birbal, ajay, ambuj)
    function amIdFromEmail() {
      return request.auth.token.email.split('@')[0];
    }

    // Task templates — AVP can read/write; AMs can only read
    match /taskTemplates/{docId} {
      allow read:  if is88gravity();
      allow write: if isAVP();
    }

    // Task instances — AM can read/write only their own amId subtree
    match /taskInstances/{docId} {
      allow read:  if isAVP() ||
                     (is88gravity() && resource.data.amId == amIdFromEmail());
      allow create: if isAVP() ||
                      (is88gravity() && request.resource.data.amId == amIdFromEmail());
      allow update: if isAVP() ||
                      (is88gravity() && resource.data.amId == amIdFromEmail());
      allow delete: if isAVP();
    }

    // Period init markers — AVP writes; AMs read only their own
    match /periodInit/{docId} {
      allow read:  if isAVP() ||
                     (is88gravity() && docId.matches(amIdFromEmail() + '-.*'));
      allow write: if is88gravity();
    }

    // User profiles — each user reads/writes their own doc
    match /userProfiles/{userId} {
      allow read, write: if request.auth != null &&
                            request.auth.uid == userId;
    }

  }
}
```

3. Click **Publish**.

---

## Step 6 — Replace Placeholders in the HTML File

1. Open `88Gravity_SEO_App.html` in any text editor (Notepad, VS Code, etc.).

2. Find this section near the top of the file (around line 20–30):

```javascript
const firebaseConfig = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT_ID.firebaseapp.com",
  projectId:         "YOUR_PROJECT_ID",
  storageBucket:     "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId:             "YOUR_APP_ID"
};
```

3. Replace each `"YOUR_..."` value with the actual values you copied in Step 2.

4. **Save the file.**

---

## Step 7 — First-Run Instructions (IMPORTANT)

The app seeds 33 default SEO tasks into Firestore the very first time an AVP logs in. **This must happen before any AM logs in.**

1. Open `88Gravity_SEO_App.html` in your browser (double-click the file, or use a local server).

2. Click **"Sign in with Google"** and sign in as **kuldeep@88gravity.com**.

3. The app will detect that tasks haven't been seeded yet and will automatically create all 33 default task templates in Firestore. You'll see a green toast notification: *"Default tasks seeded!"*

4. You'll land on the **AVP Dashboard** — confirm you can see the Overview, Allocate Task, and team boards.

5. Now share the `88Gravity_SEO_App.html` file with your AMs (or deploy it — see Step 8). When they sign in, their task instances for the current period will auto-generate.

---

## Step 8 — Deploy to Firebase Hosting (Recommended)

Hosting gives your app a shareable URL like `https://88gravity-seo-tracker.web.app` instead of opening a local file.

### Prerequisites
- Install [Node.js](https://nodejs.org) (LTS version)
- A terminal / command prompt

### Steps

1. Open a terminal and install the Firebase CLI:
```bash
npm install -g firebase-tools
```

2. Log in:
```bash
firebase login
```

3. Create a folder for your project and put `88Gravity_SEO_App.html` inside a subfolder called `public`. Rename the file to `index.html`:
```
seo-app/
  public/
    index.html        ← this is 88Gravity_SEO_App.html renamed
```

4. In the `seo-app/` folder, run:
```bash
firebase init hosting
```
- Select your project: `88gravity-seo-tracker`
- Public directory: `public`
- Single-page app: **Yes**
- Don't overwrite `index.html`: **No**

5. Deploy:
```bash
firebase deploy
```

6. Your app is now live at the URL shown in the terminal, e.g.:  
   **`https://88gravity-seo-tracker.web.app`**

7. Share this URL with Birbal, Ajay, and Ambuj. They sign in with their @88gravity.com Google accounts and are automatically routed to their AM dashboard.

---

## Step 9 — Add Additional Authorized Domains (if hosted)

After deploying to Firebase Hosting:

1. Go to **Firebase Console → Authentication → Settings → Authorized domains**.
2. Click **Add domain** and add your hosting URL: `88gravity-seo-tracker.web.app`.
3. Click **Add**.

---

## Role & Access Summary

| Email | Role | Access |
|---|---|---|
| kuldeep@88gravity.com | AVP | Full access — all AMs, all projects, task allocation, reports |
| birbal@88gravity.com | AM | Own tasks only — Today, My Tasks, By Project, Kanban |
| ajay@88gravity.com | AM | Own tasks only — Today, My Tasks, By Project, Kanban |
| ambujsurothiya@88gravity.com | AM | Own tasks only — Today, My Tasks, By Project, Kanban |

Any other @88gravity.com account attempting to sign in will be blocked with an "Access denied" message.

---

## Troubleshooting

| Issue | Fix |
|---|---|
| "This account is not authorized" | The email isn't in the ROLE_MAP in the HTML. Open the file and add the email to `ROLE_MAP`. |
| Sign-in popup blocked | Allow popups for the app's domain in your browser settings. |
| Tasks not appearing for AM | Make sure the AVP logged in first to seed tasks. Check Firestore → taskTemplates collection. |
| "Missing or insufficient permissions" | The Firestore security rules weren't published. Repeat Step 5. |
| Tasks duplicating on refresh | Normal behavior is prevented by period init markers — if duplicating, check that Firestore writes are completing (no network errors in browser console). |
| Kanban drag-and-drop not working | Use Chrome or Edge. Firefox has occasional issues with the HTML5 drag API. |

---

## Firestore Collections Reference

| Collection | Purpose |
|---|---|
| `taskTemplates` | Master list of 33 recurring task definitions (AVP-managed) |
| `taskInstances` | One doc per task per AM per period — holds status, notes, evidence |
| `periodInit` | Markers like `birbal-2026-W21` — prevents duplicate instance creation |
| `userProfiles` | Stores display name and last-seen timestamp per user |

---

*Guide written for 88Gravity SEO Team — May 2026*
