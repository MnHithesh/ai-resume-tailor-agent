# JobFit AI — AI Resume Tailor Agent

An automated pipeline that watches a Gmail inbox for recruiter emails, extracts the job description, and uses AI to generate a tailored, ATS-optimized version of a master resume — without ever inventing experience or fabricating skills. Sends a Discord notification with the ATS score and match analysis when done.

Built with [n8n](https://n8n.io) for workflow orchestration and the Google Gemini API for analysis and generation.

---

## Why this exists

Manually tailoring a resume for every job application is repetitive and easy to get wrong — either under-tailored and ignored by ATS keyword matching, or over-tailored into something dishonest. This project automates the repetitive part while keeping a hard rule: **the AI can only reorder, select, and rephrase presentation — never invent facts.**

---

## How it works

```
Gmail Trigger (IMAP)
        │
        ▼
Extract Job Description (Gemini)
        │
        ▼
Parse JD → structured JSON
        │
        ▼
Read Master Resume (final_resume.json)
        │
        ▼
Merge JD + Master Resume
        │
        ▼
AI Tailoring Analysis (Gemini)
        │
        ▼
Parse Tailored Resume
        │
        ▼
Generate ATS Score (Gemini)
        │
        ▼
Parse ATS Score
        │
        ▼
Discord Notification (webhook)
```

A recruiter email arrives → the workflow extracts the job description → reads your master resume from `C:/Downloads/final_resume.json` → produces a tailored version (reordered skills, reordered/selected experience bullets, a new role-relevant summary) → scores how well it matches the job → sends a Discord notification with the ATS score, matched skills, missing skills, and improvement suggestions.

---

## Discord Notification Preview

When a recruiter email is processed, you receive a rich embed in Discord:

```
✅ Tailored Resume Ready

🎯 Role           🏢 Company         📊 ATS Score
React Developer   Not mentioned      78/100

✅ Matched Skills
React.js, JavaScript (ES6+), TypeScript, RxJS, HTML5, CSS3

⚠️ Missing Skills
Next.js, Jest, React Testing Library

💡 Suggestions
Consider adding Next.js to your learning roadmap
Highlight testing experience if any exists in projects
```

---

## Current status

| Stage | Status |
|---|---|
| Gmail IMAP trigger | ✅ Complete |
| AI job description extraction | ✅ Complete |
| Master resume loading | ✅ Complete |
| AI resume tailoring | ✅ Complete |
| ATS scoring | ✅ Complete |
| Discord notification | ✅ Complete |
| PDF generation | 🚧 Planned (v2) |
| Resume caching (SHA256) | 🚧 Planned (v2) |

---

## Master Resume File

Your resume lives at a single, fixed location on your machine:

```
C:/Downloads/final_resume.json
```

This path is configured in the **"Read/Write Files from Disk"** node in n8n. If you switch machines or change the location, update that one field in the node — nothing else needs to change.

The file follows this JSON structure:

```json
{
  "name": "Your Name",
  "title": "Your Title",
  "contact": {
    "phone": "+xx-xxxxxxxxxx",
    "email": "you@email.com",
    "linkedin": "linkedin.com/in/yourprofile",
    "location": "City, Country"
  },
  "summary": "Your professional summary...",
  "skills": {
    "frontend": ["React", "TypeScript"],
    "backend": ["Node.js", "Express.js"],
    "devops_cloud": ["Docker", "AWS"],
    "tools": ["Git", "Figma"]
  },
  "experience": [
    {
      "company": "Company Name",
      "role": "Your Role",
      "duration": "Jan 2022 – Present",
      "bullets": [
        "Achievement or responsibility bullet point.",
        "Another bullet point."
      ],
      "tech": ["Angular", "Node.js"]
    }
  ],
  "projects": [
    {
      "title": "Project Name",
      "tech": ["React", "Node.js"],
      "bullets": [
        "What you built and why it matters."
      ]
    }
  ],
  "education": [
    {
      "degree": "MCA",
      "institution": "College Name",
      "year": "2018"
    }
  ]
}
```

Use `master_resume.example.json` in this repo as a starting template — copy it to `C:/Downloads/final_resume.json` and fill in your real details.

---

## Design decisions

**Resume stored as JSON at a fixed path, not PDF or DOCX.**
Keeping the source resume as structured JSON at `C:/Downloads/final_resume.json` lets the AI reorder and select content cleanly without ever touching formatting or risking corrupted output. Tailoring happens at the data level; presentation (PDF rendering) happens afterward in v2. The fixed path means zero configuration — just drop the file there and the workflow works on any Windows machine.

**Strict truthfulness rules enforced via prompt.**
The tailoring AI is explicitly instructed to:
- Never invent experience, skills, projects, or achievements
- Never add fake skills not in the master resume
- Never change company names, dates, or years of experience
- Never reword achievement bullets — only reorder and select
- Move less relevant skills to "additional_skills" rather than deleting them

**Three Gemini calls per run — all on the free tier.**
The pipeline uses `gemini-2.5-flash` which has a free tier with per-minute rate limits. For real use (one recruiter email at a time) this is more than sufficient. Rapid repeated testing can hit the per-minute cap — just wait 60 seconds and retry.

---

## Tech stack

- **n8n** — workflow automation and orchestration
- **Gmail IMAP** — recruiter email detection (app-password auth)
- **Google Gemini API** (`gemini-2.5-flash`) — JD extraction, resume tailoring, ATS scoring
- **Discord Webhook** — completion notifications with ATS score summary
- **Node.js** (planned) — PDF generation from the tailored JSON output

---

## Project structure

```
resume-agent/
  master_resume.example.json   # copy this to C:/Downloads/final_resume.json and fill in your details
  n8n/
    resume-agent.json          # exported n8n workflow — import this to get started
  node/
    pdf-generator.js           # PDF generation script (planned for v2)
  generated/                   # output tailored resumes / PDFs (gitignored)
  reports/                     # ATS score reports (gitignored)
  README.md
  .gitignore
```

---

## Setup

### 1. Install n8n locally
```
npx n8n
```
Open `http://localhost:5678` and create a local owner account.

### 2. Import the workflow
In n8n → `...` menu → **Import from File** → select `n8n/resume-agent.json`.

### 3. Prepare your master resume
- Copy `master_resume.example.json` from this repo.
- Fill in your real details following the JSON structure above.
- Save it as `C:/Downloads/final_resume.json`.
- This exact path is already configured in the workflow's **"Read/Write Files from Disk"** node.

### 4. Set up Gmail IMAP access
- Enable 2-Step Verification on your Google account.
- Generate an [App Password](https://myaccount.google.com/apppasswords) — name it `n8n resume agent`.
- In the **Email Trigger (IMAP)** node, create a new IMAP credential:
  - Host: `imap.gmail.com`
  - Port: `993`
  - SSL: ON
  - Username: your Gmail address
  - Password: the 16-character app password

### 5. Set up Gemini API
- Get a free API key from [Google AI Studio](https://aistudio.google.com/apikey).
- In any of the three **"Message a model"** nodes, create a new Google Gemini credential and paste the key.
- The same credential will be reused across all three Gemini nodes.

### 6. Set up Discord notifications
- In Discord, go to any channel → gear icon → **Integrations** → **Webhooks** → **New Webhook**.
- Copy the webhook URL.
- In the **HTTP Request** node, paste your webhook URL into the URL field.
- Keep this URL private — treat it like a password, never commit it to git.

### 7. Activate the workflow
- Toggle the **"Inactive"** switch in the top-right of the n8n editor to **"Active"**.
- n8n will now listen for new emails in the background even when the browser is closed.

---

## Updating your resume

When your resume changes:
1. Open `C:/Downloads/final_resume.json`.
2. Edit the relevant fields (new job, new skills, new project).
3. Save the file.

That's it — the next recruiter email that arrives will automatically use the updated resume. No workflow changes needed.

---

## Roadmap (v2)

**PDF generation.**
A Node.js script using Puppeteer to render the tailored JSON resume into a professionally formatted PDF, saved to `generated/` and attached to the Discord notification.

**Resume caching (SHA256).**
Hash the cleaned job description text before running the tailoring AI call. If an identical JD has been processed before, reuse the cached tailored resume instead of calling the AI again — saving API quota and demonstrating optimization-minded design.

**Full interview prep kit.**
Beyond the tailored resume: auto-generate a matching cover letter, tailored LinkedIn summary, likely HR/behavioral questions, likely technical questions for the role, and salary negotiation tips — all from the same job description, packaged into one Discord message.

---

## A note on honesty

This project is built around one non-negotiable constraint: **the AI never fabricates anything.** Every fact in the tailored resume output traces back to the `final_resume.json` ground truth. The AI reorders, selects, and refocuses — it does not invent. A resume tailoring tool that hallucinates experience is worse than useless, and this was a deliberate architectural decision from day one.

---

## Screenshots

*Add screenshots of the n8n canvas and a Discord notification here once live.*
