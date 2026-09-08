# Problem → Solution

A privacy-minded, full-stack problem discovery platform. People can anonymously submit genuine everyday problems; the protected admin workspace turns those reports into structured, comparable evidence.

## Architecture

- **Client:** React + TypeScript + Vite with responsive, conversation-first public pages.
- **Server:** Express API with Helmet headers, request-size limits, submission rate limiting, Zod validation, and JWT-protected admin routes.
- **Database:** SQLite (`better-sqlite3`) initialized automatically in `data/problem-solution.db`.
- **Analysis:** server-side deterministic development analyzer that creates an AI-estimate summary, category, tags, severity, frequency, solution/market potential, opportunity score, and a related-problem cluster. Replace `analyze()` with an OpenAI server-side call when `OPENAI_API_KEY` is configured; never send that key to the browser.

## Data model

- `problems`: anonymous submissions, their user-provided evidence, analysis, status, and cluster reference.
- `clusters`: related problem groups and pipeline status.
- `solution_projects`: reserved linked solution-project records for validation/build evidence.

## API

- `POST /api/submit` — validates and stores an anonymous submission.
- `POST /api/admin/login` — issues an 8-hour admin JWT.
- `GET /api/admin/dashboard` — aggregate evidence.
- `GET /api/admin/problems` — explorer data; supports `?sort=score` plus category, frequency, status, and age range filtering.
- `GET /api/admin/clusters`, `GET /api/admin/clusters/:id`, `PATCH /api/admin/clusters/:id` — cluster research and pipeline status.

### Google Sheets sync

Set `GOOGLE_SHEETS_ID`, `GOOGLE_SERVICE_ACCOUNT_EMAIL`, and `GOOGLE_PRIVATE_KEY` in `.env`, then share the spreadsheet with the service account email as an Editor. Each submission is saved locally first and then appended to the `Submissions` tab (or `GOOGLE_SHEETS_TAB`). Google Sheets is optional; if it is unavailable, the local submission still succeeds.

If your organization blocks service-account keys, use a Google Apps Script web app instead. Create a script attached to the spreadsheet with this code:

```js
function doPost(e) {
	const body = JSON.parse(e.postData.contents);
	const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(body.sheet) || SpreadsheetApp.getActiveSpreadsheet().insertSheet(body.sheet);
	if (sheet.getLastRow() === 0) sheet.appendRow(body.headers);
	sheet.appendRow(body.row);
	return ContentService.createTextOutput(JSON.stringify({ ok: true })).setMimeType(ContentService.MimeType.JSON);
}
```

Deploy it as a web app with **Execute as: Me** and **Who has access: Anyone**, then set `GOOGLE_SHEETS_WEBHOOK_URL` to the deployment URL. The app will POST each new submission to that URL.

## Run locally

1. Copy `.env.example` to `.env`, set a long `JWT_SECRET`, and set a unique admin email/password.
2. `npm install --cache .\\work\\npm-cache`
3. In one terminal: `npm run dev`
4. In another: `npm run build && npm start`

For development, open the Vite URL and it proxies API calls to port 3001. For production, run the built server and it serves the compiled React app.

## Deploy

Use a persistent volume for `DATABASE_PATH`, set all secrets as host environment variables, run `npm run build`, then `npm start`. Put the app behind HTTPS at the platform/load-balancer layer. For multi-instance or high-volume deployments, migrate the three SQLite tables to managed PostgreSQL and run AI analysis asynchronously in a job queue.

## Product safeguards

No accounts, exact birth dates, or contact details are collected. Public inputs are validated server-side and stored as text, never rendered as HTML. Admin data is server-authorized. Opportunity figures in the dashboard are intentionally labelled **AI estimates**, not objective facts.
