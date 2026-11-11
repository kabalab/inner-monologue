# Inside Your Head

An educational site about how people experience thinking: inner speech, mental imagery, thinking without words, memory and dreams. Visitors can read verified articles, take a questionnaire, and retrieve a personal breakdown with an email address and a private PIN.

**Live site:** https://kabalab.github.io/inner-monologue/

This is an educational self-reflection tool. It is not a medical or psychological test, and it cannot diagnose anything.

## What is in this repository

| Path | Published? | Role |
| --- | --- | --- |
| `index.html` | Yes (GitHub Pages) | The whole website |
| `README.md` | Yes | This file |
| `.gitignore` | Yes | Keeps the backend and tests off GitHub |
| `apps-script/` | No (local only) | Google Apps Script backend |
| `tests/` | No (local only) | Automated checks |

The questionnaire structure (questions, branching and scoring) is defined once in `apps-script/Code.gs` between `// BEGIN SHARED SPEC` and `// END SHARED SPEC`, and copied character-for-character into `index.html`. A test fails if the two copies differ.

## Architecture

```
Visitor browser  →  GitHub Pages (index.html)
                 →  Apps Script web app (/exec)  →  private Google Sheet (Submissions tab)
                                                 →  Script Properties (secrets)
```

- **Frontend:** one HTML file, hash routing (`#/`, `#/learn`, `#/quiz`, `#/results`, `#/retrieve`, `#/admin`). No CDNs, web fonts, analytics or third-party scripts.
- **Backend:** one Apps Script web app, Execute as Me, access Anyone. The browser POSTs JSON as `text/plain` (Apps Script cannot answer CORS preflights). Fixed actions only: `submit`, `checkEmail`, `retrieve`, `publicStats`, `adminLogin`, `adminLogout`, `adminStats`, `adminList`, `adminGet`, `adminDelete`.
- **Data:** one private Sheet tab named `Submissions`. Stats are computed from the raw rows. No Statistics or Config tab.
- **Secrets (Script Properties only):** `SPREADSHEET_ID`, `ADMIN_PASSWORD`, `SESSION_SECRET`. The script also keeps `ADMIN_SESSIONS` (ids of open admin sign-ins; not a secret).

## How the administrator signs in

There is no admin link on the site.

1. Open https://kabalab.github.io/inner-monologue/#/retrieve
2. Type `ADMIN` as the email (any capitals).
3. Type the admin password in the PIN box (it shows dots).
4. Press **Show my results**.

A wrong password shows the same message as a failed retrieval. **Sign out** is on the dashboard. The sign-in stays open until you sign out or close the tab (up to 10 open sign-ins; an 11th closes the oldest). Changing `ADMIN_PASSWORD` or `SESSION_SECRET` ends every open sign-in at once.

## Deploying a backend change

1. Open the bound Apps Script project from the private Sheet (**Extensions → Apps Script**).
2. Select all of `Code.gs`, paste the local `apps-script/Code.gs`, **Save**.
3. **Deploy → Manage deployments →** pencil → **New version → Deploy**.
4. Do **not** run `setup()` unless the sheet columns changed. Do **not** clear the Sheet.

Deploy the backend **before** pushing a page that depends on a new reply shape. The live page and the backend must stay compatible.

To change the admin password: edit Script Property `ADMIN_PASSWORD`. It takes effect immediately. No redeploy.

## Limits you should know

These are accepted design limits, not surprises:

- The `/exec` URL is public (it is in the page). The Sheet stays private because the script runs as you and exposes only the fixed actions.
- Apps Script cannot see a visitor's IP address. Rate limits are site-wide or per email. A stranger can cause short lock-outs by guessing (admin: 8 wrong → 15 minutes site-wide; retrieval: 5 wrong → 15 minutes per email). Running `clearLockouts()` in the Apps Script editor lifts them.
- Email addresses are not verified. Someone can type another person's address and block that person until you delete the row.
- PINs are stored readably so you can help people who lose them. Anyone with edit access to the Sheet can also open its script and read Script Properties. Do not share the Sheet.
- Deleting a row removes it from the live data. Google's version history of the sheet may still hold older copies.
- There is no limit on how many new questionnaires can be saved. Spam rows are possible; delete them in the admin area. A site-wide save limit was considered and rejected, because a stranger could use it to stop everyone from saving.
- The early email check reveals whether an address has already finished (required). Retrieval errors stay generic.
- GitHub Pages cannot send the usual clickjacking header. The admin area refuses to run inside another page with a script check.
- Comparisons on the results page are only against this site's volunteers, shown only when enough people answered, and never invent population statistics.

## Honest test report

Report date: 10 November 2026. Build under test: the live page at commit `3134ce9` (byte-identical to the local file when checked) and the deployed Apps Script that includes the admin sign-in list.

### Automated checks I ran

| Check | Result |
| --- | --- |
| Full local suite (`node --test`) | 191 pass, 0 fail, 0 skipped |
| Deliberate breaks of the admin and sign-in logic | 22 of 22 caught |
| Live backend check (`tests/live-check.js`) against the real `/exec` URL | 28 of 28, including two open sign-ins and sign-out; test rows removed |
| Admin browser scenarios (sign-in, framing guard, empty states, six people, 55 people, twenty days, layouts at 320 / 375 / 1100 px) | all pass |
| Questionnaire, results and Retrieve browser scenarios | pass on the final build |
| Live admin run (seed temporary people, dashboard with percentages, search, detail, two-step delete, sign out, cleanup) | pass; Sheet left empty |

### Step 4 functions (your original list)

| Function | How it was tested |
| --- | --- |
| Homepage navigation | Router tests + browser navigation |
| Educational article navigation | Content tests + browser |
| Search | Browser search on the Learn page |
| Questionnaire branching | Shared-spec tests + browser quizzes |
| Main questionnaire completion | Browser + live end-to-end + live-check |
| Optional Memories & Dreams | Browser longest path + live-check |
| Skipping the optional section | Browser |
| PIN generation | Live-check (shape and uniqueness) |
| Submission storage | Live-check + Sheet row checks |
| Duplicate email prevention | Live-check + early email check |
| Result generation | Server scoring tests + browser results |
| Result retrieval | Live-check + Retrieve browser scenario |
| Incorrect PIN handling | Same message for wrong PIN and unknown email; lock-out |
| Admin authentication | Live-check + hidden Retrieve login |
| Dashboard statistics | Admin audit + live admin run (counts and percentages) |
| Individual submission viewing | Admin audit + live admin run |
| Submission deletion | Two-step confirm + live delete + email free again |

### Security review (read-only)

Two independent reviews of `Code.gs` and `index.html` found **no high-severity** issues and **no XSS path**, including stored XSS into the admin dashboard.

Fixes applied from that review:

- Privacy notice wording about what the browser tab keeps.
- Admin **Sign out** clears the tab first, then tells the server (so a slow reply cannot leave the dashboard open).
- Admin overview shows percentages next to counts.

Left as documented limits (owner decisions): no site-wide save throttle; readable PINs; hidden admin login; long-lived admin sign-ins with a server-side list.

### What I have not tested

- A real phone in your hand
- A screen reader with a real keyboard
- Tab-key walking through every control by hand
- Browsers other than the Chromium built into this environment
- The 15-minute admin lock-out message on the live site (testing it would lock you out)

### What you should still check yourself

1. Sign in on the live Retrieve page with `ADMIN` and your password.
2. On a phone, open home, Learn, one article, the questionnaire, Retrieve and (after signing in) the admin area.
3. Confirm the Google Sheet sharing is still **Restricted**.
4. Read the privacy notice on `#/quiz` and confirm you are happy with the wording.
5. When Phase 8 is approved, **replace the throwaway `ADMIN_PASSWORD`** in Script Properties with a long random password. Changing it ends every open sign-in immediately.

## Local checks (optional)

The `tests/` folder stays on this computer. From the project folder:

```powershell
node --test --test-reporter=tap | Select-String '^\s*not ok|^# (tests|pass|fail|skipped)'
```

Against the live backend (set the password in the shell; do not commit it):

```powershell
$env:BACKEND_URL = 'https://script.google.com/macros/s/<your-deployment-id>/exec'
$env:ADMIN_PASSWORD = 'your-admin-password'
node tests/live-check.js
Remove-Item Env:ADMIN_PASSWORD, Env:BACKEND_URL
```

## Expert read (optional)

Some article claims rest on background knowledge rather than a retrieved abstract. A human expert should check: Baddeley 2003, Marks 1973, Galton 1880, Nisbett & Wilson 1977, Schwitzgebel 2008, Brown & McNeill 1966, Ross et al. 1977, and the four books listed in the references.
