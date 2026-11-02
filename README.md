# Learning Lab

A web portal where invited learners sign in with an email link, build a profile, and work through interactive workbooks. Every answer autosaves, progress is tracked per workbook, and each workbook compiles into a downloadable brief. An admin uploads new workbooks as JSON, and sees progress percentages but never learners' answers.

**Live:** https://learning-lab-yezi.web.app

This is built from the Learning Lab PRD v1.0. Where the PRD and this code disagree, the PRD describes the target and this README describes what is actually built.

## What's in it

- **Passwordless sign-in** with Firebase Auth email links. Open sign-up: anyone with the link can enter their email and create an account.
- **Learner profile**, module catalogue, enrolment, module player (guidance, key terms, 12 field types, live computed columns, autosave), dashboard, journey view.
- **Briefs** per workbook: copy as text, download Markdown, print to PDF.
- **Admin console** (one admin email only): upload and validate module JSON, save as draft or publish, unpublish, hide built-ins, download JSON, delete, view learner progress.
- **Six seed workbooks**: Experimental Leader, Product Discovery, Product-led, RPT Vision, Buying PX Software, Leadership Exercises.
- **Analytics** with PostHog (EU), cookieless, events carry ids and counts only.

## Architecture

A single static page, no build step.

```
public/index.html      the whole app: HTML, CSS, JS, seed modules, Firebase + PostHog wiring
firestore.rules        server-side access control
firebase.json          hosting config (security headers) and rules path
.firebaserc            default project: learning-lab-yezi
```

| Concern | Choice |
| --- | --- |
| Hosting | Firebase Hosting |
| Auth | Firebase Auth, email link (passwordless) |
| Data | Cloud Firestore |
| Analytics | PostHog EU (`posthog-js` snippet) |

### Data model (Firestore)

Documents are stored as one JSON string in a field `j`, because Firestore rejects nested arrays and the module and answer formats use them.

| Path | Contents | Read | Write |
| --- | --- | --- | --- |
| `users/{uid}/data/profile` | profile, enrolments, recent activity | owner | owner |
| `users/{uid}/data/m-{moduleId}` | answers for one module | owner | owner |
| `progress/{uid}` | name, last active, percent per module | owner, admin | owner |
| `catalog/{moduleId}` | uploaded module spec and status | any signed-in user | admin |
| `config/portal` | hidden built-ins, portal settings | any signed-in user | admin |

### Security model

Enforced by `firestore.rules`, not by the UI:

- A learner can read and write only their own profile and answers. **No one else can read them, including the admin.**
- The admin can read progress percentages and write modules and settings.
- Admin means the verified email `yezi.lilyz@gmail.com`. Change it in `firestore.rules` and in `public/index.html` (`ADMIN_EMAIL`) if ownership changes. The rule is the real gate; the constant only decides whether the Admin tab is shown.
- Modules are data only. Computed columns are evaluated by a whitelisted expression check, and all rendered text is escaped.

The Firebase web config and the PostHog project token in `index.html` are public by design. They identify the project; they do not grant access. Access is controlled by the Firestore rules and by Auth.

## Run locally

```bash
cd public
python3 -m http.server 8000
# open http://localhost:8000
```

`localhost` is an authorised domain for Auth, so email-link sign-in works locally and talks to the live Firebase project. Use a throwaway email, because it creates real accounts and data.

## Deploy

Requires the [Firebase CLI](https://firebase.google.com/docs/cli) and access to the `learning-lab-yezi` project.

```bash
firebase login
firebase deploy --only firestore   # security rules
firebase deploy --only hosting     # the app
```

To use a different Firebase project: create it, enable Firestore and Authentication, turn on **Email/Password → Email link (passwordless sign-in)**, replace the `initializeApp({...})` config in `public/index.html`, update `.firebaserc`, then deploy.

## Module format

Admins upload workbooks as JSON (max 200 KB). Download the template from Admin → Upload a module. Field types: `text`, `textarea`, `preview`, `checklist`, `table`, `list`, `choice`, `ratelist`, `score`, `ab`, `sliders`, `roi`. Computed table columns use declarative `formula`, `sum`, `avg` or `lookup` operations. The validator lists every error with its location before anything can be saved.

## Analytics

PostHog EU, configured for privacy: memory-only persistence (no cookies), no autocapture, no session replay, no performance capture, Do Not Track respected, query strings stripped from all URLs (they can carry one-time sign-in codes), learners identified by Firebase uid, never by email.

| Event | Properties |
| --- | --- |
| `signin_link_requested` | method |
| `signin_completed` | method |
| `profile_saved` | first_time, fields_filled_count |
| `module_enrolled` | module_id, source (catalog, preview) |
| `tool_answered` | module_id, tool_id, status (started, done) |
| `module_completed` | module_id, days_since_enrolled |
| `brief_exported` | module_id, format (copy, md, print) |
| `glossary_opened` | module_id, from (overview, tool) |
| `module_uploaded` | module_id, valid, error_count |
| `module_published` | module_id |
| `$pageview` | view, module_id, tool_id |

Events never contain answer text.

## Known limitations

This is a working prototype, not the full PRD.

- Sign-in emails come from Firebase's shared default sender and can land in spam. A custom sender domain with SPF and DKIM is the fix.
- No server-side API: the browser talks to Firestore directly, protected by rules. There are no rate limits beyond Firebase's own, no GDPR export or delete, no audit log, and no module versioning or rollback.
- Admin reads progress only for learners who have saved a profile, and learner names come from the progress document.
- Offline queueing and conflict handling from the PRD are not built. Autosave is last-write-wins.
- Documents are JSON strings, so Firestore cannot query inside them.
- Invitations are not needed while sign-up is open, so the Invite tab only offers the portal link.
