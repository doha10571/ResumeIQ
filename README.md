# ResumeIQ — CV Optimizer

An AI-powered job application assistant built with **n8n**, **Google Gemini**, **Supabase** and a single-file HTML frontend.

Upload a CV and a job description, get a match analysis, receive a tailored ATS-friendly CV as a PDF, then review and send the application email — nothing is sent without your approval.

## How it works

| Step | Screen | Webhook | What happens |
|------|--------|---------|--------------|
| 1 | Upload | `POST /cv-analysis` | CV text is cleaned and stored, Gemini parses the CV and the job description, then scores the match |
| 2 | Analysis | `POST /cv-tailoring` | Gemini rewrites the CV for the job (without inventing experience), the result is validated and rendered to PDF via APITemplate |
| 3 | Optimize | `POST /application` | Gemini drafts the application email and saves it as `pending_approval` |
| 4 | Send | `POST /application-approve` | The PDF is downloaded, attached and sent through Gmail; the application is marked `sent` |

PDF and DOCX files are read in the browser (pdf.js / mammoth), so only the extracted text is sent to the workflow.

## Project structure

```
cv_optimizer_frontend.html   Frontend (no build step, open it in a browser)
ResumeIQ.json                n8n workflow export
```

## Setup

### 1. Supabase

Create these tables (columns inferred from the workflow — adjust types as you like):

```sql
create table cvs (
  cv_id   uuid primary key,
  job_id  uuid,
  cv_text text
);

create table analyses (
  id              uuid primary key default gen_random_uuid(),
  cv_id           uuid,
  job_id          uuid,
  match_score     numeric,
  matched_skills  jsonb,
  missing_skills  jsonb,
  recommendations jsonb,
  analysis_data   jsonb
);

create table tailored_cvs (
  id                uuid primary key default gen_random_uuid(),
  cv_id             uuid,
  job_id            uuid,
  tailored_cv_text  text,
  cv_html           text,
  validation_status boolean,
  resume_url        text
);

create table applications (
  id               uuid primary key default gen_random_uuid(),
  cv_id            uuid,
  job_id           uuid,
  tailored_cv_id   uuid,
  hr_email         text,
  email_subject    text,
  email_body       text,
  resume_url       text,
  status           text,
  sent_at          timestamptz,
  gmail_message_id text
);
```

### 2. n8n

1. Import `ResumeIQ.json` into n8n (**Workflows → Import from file**).
2. Create the credentials and attach them to the nodes:

   | Credential | Type | Used by |
   |------------|------|---------|
   | Supabase account | Supabase API | all Supabase nodes |
   | Header Auth account | Header Auth (`x-goog-api-key: <Gemini API key>`) | Gemini HTTP nodes |
   | Header Auth account 2 | Header Auth (`X-API-KEY: <APITemplate key>`) | Generate PDF |
   | Gmail account | Gmail OAuth2 | Send Email |

3. In **Generate PDF (APITemplate)**, replace `template_id` with your own APITemplate template.
4. Activate the workflow.

### 3. Frontend

Set your n8n webhook base URL in `cv_optimizer_frontend.html`:

```js
const CONFIG = {
  N8N_BASE_URL: "https://<your-n8n-host>/webhook"
};
```

Use `/webhook-test` while testing in the n8n editor and `/webhook` once the workflow is active. Then open the file in a browser.

## Security note

The webhooks are unauthenticated and allow any origin. Anyone who knows the base URL can trigger the workflow, including the email step. Before exposing it publicly, add webhook authentication (e.g. Header Auth) and restrict `allowedOrigins`.

## Tech stack

n8n · Google Gemini · Supabase · APITemplate.io · Gmail API · HTML/CSS/JS · pdf.js · mammoth.js
