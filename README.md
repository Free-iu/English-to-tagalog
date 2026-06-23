# Tagalog Translator Agent — Setup Guide

This is a single static HTML file (`index.html`). It has no server of its
own — login and "save my progress" are handled by **Firebase** (a free
backend service from Google), and translation is done by calling the
**Gemini** API directly from your browser.

You must do a few setup steps yourself before this works. I can't do these
for you — they require your own Google account and your own GitHub account.

---

## Part 1 — Create your free Firebase project (for login + saved progress)

1. Go to https://console.firebase.google.com and sign in with a Google account.
2. Click **"Add project"**, give it any name, finish the wizard (you can
   skip Google Analytics).
3. In the left sidebar: **Build → Authentication → Get started**.
   - Click the **Email/Password** provider, enable it, save.
4. In the left sidebar: **Build → Firestore Database → Create database**.
   - Choose any region close to you.
   - Start in **test mode** for now (we'll lock it down in step 6).
5. In the left sidebar: **Project settings (gear icon) → General**.
   - Scroll to "Your apps" → click the **</> (Web)** icon → register an app
     (any nickname).
   - Firebase will show you a `firebaseConfig` object that looks like:
     ```js
     const firebaseConfig = {
       apiKey: "AIza...",
       authDomain: "yourproject.firebaseapp.com",
       projectId: "yourproject",
       storageBucket: "yourproject.appspot.com",
       appId: "1:..."
     };
     ```
   - Copy that whole block.
6. Open `index.html` in any text editor (or directly in GitHub, see Part 3)
   and find this section near the top of the `<script>` tag:
   ```js
   const firebaseConfig = {
     apiKey: "YOUR_FIREBASE_API_KEY",
     ...
   };
   ```
   Replace it with the real values Firebase gave you. Save the file.

7. **Lock down your database** so only each user can see their own data.
   Back in Firebase: **Build → Firestore Database → Rules**, replace the
   default rules with:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
         match /projects/{projectId}/chunks/{chunkId} {
           allow read, write: if request.auth != null && request.auth.uid == userId;
         }
       }
     }
   }
   ```
   Click **Publish**.

   This Firebase API key/config above is **not secret** in the way an AI
   API key is — it just tells your browser which Firebase project to talk
   to. The Firestore security rules above are what actually protect your
   data, so don't skip step 7.

---

## Part 2 — Get a free Gemini API key (for the actual translation)

1. Go to https://aistudio.google.com/apikey
2. Sign in, click **Create API key**, copy it.
3. You'll paste this into the website itself (in the "AI API key" box),
   not into the code. It gets saved to your Firebase account once you
   click "Save key to my account."

Perplexity and Grok (xAI) keys can technically be entered too, but those
providers usually block direct browser requests (CORS) — without a real
backend server they likely will fail. Gemini is the one this is built to
actually work with.

---

## Part 3 — Put it on GitHub Pages (so you can open it anytime, anywhere)

1. Create a new repository on GitHub (e.g., `tagalog-translator`).
2. Upload `index.html` into the repo (GitHub's web UI lets you drag-and-drop
   — "Add file → Upload files").
   - If you haven't edited in your `firebaseConfig` yet, you can also click
     the file in GitHub, click the pencil/edit icon, paste your real
     values in directly, and commit.
3. Go to repo **Settings → Pages**.
   - Under "Build and deployment," set Source = **Deploy from a branch**,
     Branch = **main**, folder = **/ (root)**. Save.
4. GitHub will give you a URL like:
   `https://yourusername.github.io/tagalog-translator/`
   Wait a minute or two after the first deploy, then open it.
5. Bookmark that URL. Every time you open it and log in, your saved API
   key and translation progress will be there waiting.

---

## How resuming works

- When you upload the same PDF again, the app recomputes the same text
  chunks from it (deterministic) and checks Firestore for which chunk
  indexes are already translated for that exact file. It only calls the
  API for the chunks still missing.
- This means: if you hit a quota limit halfway through, just close the
  tab. Come back later (even on a different computer, since it's tied to
  your login, not your device), log in, re-upload the same file, hit
  "Start / Resume," and it picks up where it left off.
- "Reset progress for this file" wipes saved chunks for that file only, in
  case you want to start over.

## Known limitations (being upfront)

- Large PDFs (your 300-page collection) will take a long time and will
  likely hit Gemini's free-tier daily quota before finishing in one sitting
  — that's expected, not a bug. Resuming the next day works fine.
- This extracts text from the PDF using layout-based text extraction in
  the browser; very unusually formatted PDFs may extract text in a
  slightly different order than a human reader would expect. Worth a
  skim of the output before you trust it fully.
- The generated .docx is built from a minimal but valid OOXML structure
  (no styles.xml, no headers/footers) — it opens correctly in Word/Google
  Docs/LibreOffice with your Times New Roman / size / single-spacing /
  black-text formatting, but it's intentionally bare-bones beyond that.
