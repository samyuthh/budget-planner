# Budget Planner

A finance dashboard that tracks income, expenses and savings goals.
Built with **Vue 3**, **Chart.js** and **Firebase** (Auth + Firestore, free tier).

## Features
- Email/password sign-in (Firebase Auth)
- Income and expense entry forms
- Pie (doughnut) chart of spending by category, bar chart of income vs expenses by month
- Filters by month and category
- Savings goals with progress bars
- Per-user data stored in Firestore, with real-time sync

## Setup
1. Create a project at https://console.firebase.google.com (free Spark plan).
2. **Authentication** → Sign-in method → enable **Email/Password**.
3. **Firestore Database** → create a database (production mode), then paste the contents of `firestore.rules` into the Rules tab and publish.
4. **Project settings** → Your apps → add a **Web app** and copy its config.
5. Copy `.env.example` to `.env` and fill in the values from that config.
6. Install and run:
   ```bash
   npm install
   npm run dev
   ```

## Scripts
- `npm run dev` – start the dev server
- `npm run build` – production build into `dist/`
- `npm run preview` – preview the production build

## Data model
```
users/{uid}/transactions/{id}  { type, amount, cat, date, note }
users/{uid}/goals/{id}         { name, target, saved }
```
Security rules restrict each user to their own `users/{uid}` subtree.
