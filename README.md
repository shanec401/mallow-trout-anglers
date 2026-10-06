# Mallow Trout Anglers 🎣

A shared catch log, leaderboard, and competition manager for Mallow Trout Anglers Association (founded 1919) — built as a single self-contained web app (`index.html`), backed by Firebase, hosted free on GitHub Pages.

---

## Features

- **Google or email/password sign-in** — no separate club login system to manage
- **Catch log** — species, weight (g), length (cm), date, location, notes, and a phone-camera photo
- **Leaderboard** — ranked by total length, with overall and per-competition / per-year views; exportable to CSV
- **Competitions** — admins create them (with an optional Cup/Trophy name), members request to join, admins approve or deny; only approved members see that competition's results
- **Anglers roster** (admin-only) — contact details, membership, and fee status, tracked per year, with CSV export and a one-click "copy to next year" tool
- **In-app admin roles** — separate from GitHub/Firebase access; admins are managed from the Settings tab
- **Auto sign-out** after 20 minutes of inactivity
- **Full data backup** — one-click JSON export of everything, from Settings
- Home-screen icon support on mobile (iOS "Add to Home Screen" shows the club crest)

---

## How it's built

One file, `index.html`, containing all HTML/CSS/JS. It uses:

| Need | Service |
|---|---|
| Hosting | GitHub Pages (free) |
| Sign-in | Firebase Authentication (Google + Email/Password) |
| Shared data | Firebase Firestore |
| Photos | Firebase Storage |

All on Firebase's free **Spark plan** — no credit card required, generous limits for a small club.

---

## One-time setup

### 1. Create a Firebase project
1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project**
2. Skip Google Analytics (not needed)

### 2. Register a web app
1. On the project overview page, click **+ Add app** → choose the **Web** option
2. Give it any nickname, skip Firebase Hosting (we're using GitHub Pages instead)
3. Copy the `firebaseConfig` object it shows you

### 3. Paste the config into `index.html`
Open `index.html`, find this block near the top of the `<script>` section, and replace it with your real values:

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

This is safe to leave visible — Firebase web API keys aren't secret. Security comes from the rules below, not from hiding this.

### 4. Enable Authentication
Firebase console → **Authentication** → **Sign-in method** → enable:
- **Google**
- **Email/Password**

Also check **Authentication → Settings → Authorized domains** and add your GitHub Pages domain (e.g. `yourusername.github.io`), or sign-in will fail on the live site.

### 5. Create Firestore Database
Firebase console → **Firestore Database** → **Create database** → start in **production mode**.

Then go to the **Rules** tab and replace the default rules with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isSignedIn() {
      return request.auth != null;
    }
    function isAdmin() {
      return isSignedIn() &&
        exists(/databases/$(database)/documents/admins/$(request.auth.uid));
    }
    function isApprovedFor(compId) {
      return isSignedIn() &&
        exists(/databases/$(database)/documents/registrations/$(compId + '__' + request.auth.uid)) &&
        get(/databases/$(database)/documents/registrations/$(compId + '__' + request.auth.uid)).data.status == 'approved';
    }

    match /catches/{catchId} {
      allow read: if isSignedIn();
      allow create: if isSignedIn()
        && request.resource.data.anglerId == request.auth.uid
        && (request.resource.data.competitionId == null || isApprovedFor(request.resource.data.competitionId));
      allow update, delete: if isSignedIn() &&
        (isAdmin() || resource.data.anglerId == request.auth.uid);
    }

    match /competitions/{compId} {
      allow read: if isSignedIn();
      allow create, update, delete: if isAdmin();
    }

    match /registrations/{regId} {
      allow read: if isSignedIn();
      allow create: if isSignedIn()
        && regId == request.resource.data.competitionId + '__' + request.auth.uid
        && request.resource.data.anglerId == request.auth.uid
        && request.resource.data.status == 'pending';
      allow update: if isAdmin() ||
        (isSignedIn() && resource.data.anglerId == request.auth.uid
          && request.resource.data.anglerId == request.auth.uid
          && request.resource.data.status == 'pending');
      allow delete: if isAdmin() ||
        (isSignedIn() && resource.data.anglerId == request.auth.uid);
    }

    match /admins/{uid} {
      allow read: if isSignedIn();
      allow write: if isAdmin();
    }

    match /roster/{entryId} {
      allow read, write: if isAdmin();
    }
  }
}
```

Click **Publish**.

### 6. Set up Storage (for catch photos)
Firebase console → **Storage** → **Get started** (any default location is fine).

Go to the **Rules** tab and set:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /catch-photos/{fileName} {
      allow read: if request.auth != null;
      allow write: if request.auth != null
        && request.resource.size < 10 * 1024 * 1024
        && request.resource.contentType.matches('image/.*');
    }
  }
}
```

Click **Publish**.

### 7. Deploy to GitHub Pages
1. Create a public GitHub repo
2. Upload `index.html` to it (overwrite each time you get an updated version)
3. Repo → **Settings → Pages** → Source: **Deploy from a branch** → `main` / root
4. Your site will be live at `https://yourusername.github.io/your-repo-name/`

### 8. Make yourself the first admin
Nobody is an admin until you add one by hand (after that, admins can promote others from the app itself):

1. Sign in once on the live site (Google or email) so Firebase creates your account
2. Firebase console → **Authentication → Users** → copy your **User UID**
3. Firebase console → **Firestore Database → Data** → **Start collection** → ID: `admins`
4. Document ID: paste your User UID → add any field (e.g. `addedAt: 1`) → **Save**
5. Refresh the live site — you should now see **Settings** and **Anglers** tabs, and "New competition" on the Competitions tab

---

## Day-to-day admin tasks

| Task | Where |
|---|---|
| Approve someone to join a competition | Competitions tab → that competition's admin panel |
| Create / edit / delete a competition | Competitions tab |
| Add / edit / remove a club member's roster entry | Anglers tab |
| Roll fee records over to a new year | Anglers tab → pick a year → "Copy to next year →" |
| Promote another member to admin | Settings tab |
| Download a full data backup | Settings tab |
| Export a leaderboard or the roster as a spreadsheet | "Export CSV" buttons on the Leaderboard / Anglers tabs |

---

## Known limitations

- **No automatic backups.** Firebase's free tier doesn't back anything up for you — use the Settings → backup export occasionally, especially before deleting things in bulk.
- **Read-side privacy is app-level, not database-level.** The Firestore rules above properly lock down *writes* (nobody can fake another angler's catches, self-approve a competition, or self-promote to admin). But *reading* is only filtered by the app's own code — a technically determined person with developer tools could still query the raw database. For a small trusted club this is normally an acceptable tradeoff.
- **No full PWA install prompt.** The crest shows up as a home-screen icon on iOS Safari, but a true "Install app" experience on Android would need a separate `manifest.json` and service worker file, which isn't included here (single-file app by design).
- **Push notifications aren't included** (e.g. "your registration was approved") — that needs Firebase Cloud Functions, which sits outside the free Spark plan.

---

## Support

This app has no external dependency beyond the Firebase SDK (loaded from Google's CDN) and Google Fonts. If something breaks, the browser's developer console (F12 → Console tab) will usually show a clear error — most issues trace back to either a missing Firestore/Storage rule, or a browser caching an old version of `index.html` (hard refresh with Ctrl+Shift+R / Cmd+Shift+R after any update).
