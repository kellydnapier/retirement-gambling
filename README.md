# Retirement Gambling

A daily stock-gain goal tracker. Only counts NYSE market days. Installs as a PWA on iOS / Android / desktop. Syncs across devices via Firebase.

## Quick start

1. Fill in `firebase-config.js` with your Firebase project's web config.
2. Set up Firebase (auth providers + Firestore rules — see below).
3. Push to a GitHub repo, enable GitHub Pages.
4. Open the deployed URL in Safari on your iPhone → Share → Add to Home Screen.

---

## 1. Firebase setup

### Create the project

1. Go to [console.firebase.google.com](https://console.firebase.google.com), click **Add project**.
2. Name it whatever (e.g. `retirement-gambling`). Disable Analytics — not needed.
3. Once created, click the web icon `</>` to register a web app. Skip Firebase Hosting (you're using GitHub Pages).
4. Copy the `firebaseConfig` object Firebase shows you. Paste those values into `firebase-config.js`.

### Enable Auth providers

In Firebase Console → **Authentication** → **Sign-in method**:

- Enable **Email/Password**.
- Enable **Google**. Set a public-facing name and pick a support email.

### Add your GitHub Pages domain as an authorized domain

Firebase Console → **Authentication** → **Settings** → **Authorized domains** → Add:

- `<your-username>.github.io`

(`localhost` is already authorized for local testing.)

### Enable Firestore

Firebase Console → **Firestore Database** → **Create database** → Start in **production mode** → pick a region close to you (e.g. `us-west2`).

### Set Firestore security rules

In Firestore → **Rules**, replace the rules with this and click **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;

      match /balances/{date} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }
    }
  }
}
```

This makes each user's data readable/writable only by them.

---

## 2. Deploy to GitHub Pages

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

In the repo on GitHub → **Settings** → **Pages**:

- Source: **Deploy from a branch**
- Branch: `main` / root
- Save.

After a minute the site will be live at `https://<your-username>.github.io/<repo-name>/`.

---

## 3. Install on iPhone

1. Open the GitHub Pages URL in **Safari** (must be Safari, not Chrome).
2. Tap the Share icon → **Add to Home Screen**.
3. Confirm. The app icon appears on your home screen and launches in standalone mode (no Safari chrome).

For desktop install (Chrome/Edge): there'll be an install icon in the address bar.

---

## How it works

### Math model

- You enter a **starting balance** on your **start date**. That's your baseline.
- Each market day adds your **daily goal** to a **cumulative target gain**.
- Each day you log your **current account balance**. The app computes your **actual gain** as `current_balance − starting_balance`.
- **% ahead/behind** = `(actual_gain − cumulative_target_gain) / cumulative_target_gain × 100`.

### Market days

NYSE holidays are computed algorithmically — no yearly updates needed. Includes:

- All standard fixed-date holidays (New Year's, Independence Day, etc.) with observed-on-weekday rules.
- Floating holidays (MLK, Presidents', Memorial, Labor, Thanksgiving).
- Good Friday (Easter calculated via Meeus/Jones/Butcher).
- Juneteenth (from 2022 onward).

Half-days (Black Friday, Christmas Eve, July 3) count as full trading days per your spec.

### Goal changes

Change your daily goal at any time. The change applies **from the effective date forward** — previously accrued cumulative target is unchanged. The full goal history is visible in Settings.

### Missing entries

If a past market day has no balance entry, the app flags it (amber badge on dashboard + amber outline on calendar). Computations treat the missing day as $0 toward your gain until you backfill, so "behind by X%" is real.

### Backfilling

Tap any past market day in the calendar to enter or edit that day's balance. Same for the start date — edit it in Settings.

---

## File layout

```
/
├── index.html           ← whole app: HTML + CSS + JS
├── firebase-config.js   ← your keys go here
├── manifest.json        ← PWA install metadata
├── sw.js                ← service worker (offline shell)
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   └── apple-touch-icon.png
└── README.md
```

---

## Data model (Firestore)

```
users/{uid}
  startDate: "YYYY-MM-DD"
  startingBalance: number
  goalHistory: [
    { effectiveDate: "YYYY-MM-DD", dailyGoal: number },
    ...
  ]
  createdAt: timestamp

users/{uid}/balances/{YYYY-MM-DD}
  balance: number
  updatedAt: timestamp
```

One Firestore document per balance entry. With 20 years of trading data (~5,040 entries) you're nowhere near any Firestore limits.

---

## Local development

Open `index.html` in a local server (the service worker and ES modules require one, not `file://`):

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

---

## Troubleshooting

- **"Sign in with Google" popup gets blocked**: allow popups for the domain.
- **Auth/unauthorized-domain error**: add `<your-username>.github.io` to authorized domains in Firebase Console.
- **PWA icon doesn't update on iOS**: iOS aggressively caches Home Screen icons. Delete the icon, hard-reload Safari, re-add to Home Screen.
- **Sync not happening across devices**: confirm both devices are signed in with the same account. Firestore syncs in real-time when online.
