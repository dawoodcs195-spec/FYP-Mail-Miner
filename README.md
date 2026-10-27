<div align="center">

# Mail Miner: Scenario-Based Gmail Filtering

**A MERN application that syncs a Gmail inbox through the Gmail API and sorts mail into Important, Remaining and Spam using user-defined scenarios, including keyword, GPA and PDF-attachment matching.**

*Final Year Project, BS Computer Science, UET Mardan (three-member team)*

![React](https://img.shields.io/badge/React-19-149ECA?logo=react&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-7-646CFF?logo=vite&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind%20CSS-4-06B6D4?logo=tailwindcss&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-Express%205-339933?logo=nodedotjs&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose-47A248?logo=mongodb&logoColor=white)
![Gmail API](https://img.shields.io/badge/Gmail%20API-OAuth%202.0-EA4335?logo=gmail&logoColor=white)
![JWT](https://img.shields.io/badge/Auth-JWT-000000?logo=jsonwebtokens&logoColor=white)

</div>

---

## Contents

- [The problem](#the-problem)
- [Features](#features)
- [How matching works](#how-matching-works)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [API overview](#api-overview)
- [Roadmap](#roadmap)
- [Team](#team)

## The problem

Recruiters, coordinators and admissions staff receive hundreds of emails and attachments, and the ones that matter (for example "internship applications with GPA above 3.0") are buried. Mail Miner lets the user describe what they are looking for in plain words and pulls those emails, including ones where the match is only inside an attached PDF CV, into a single Important view.

## Features

- **Sign in with Google** using the OAuth 2.0 authorization-code flow (`@react-oauth/google` on the client, `googleapis` on the server) with read-only Gmail scope
- **Background Gmail sync** of up to 2,500 messages (paged 500 at a time, spam included), skipping messages already stored
- **Attachment download**: attachments are fetched through the Gmail API and saved on the server so they can be opened from the UI
- **Spam detection** from Gmail's own `SPAM` label
- **Scenario Manager**: the user defines a scenario (name + keywords); saving it clears the previous results and re-scans the whole mailbox in the background
- **Scenario matching** over subject, snippet, body, attachment file names and **text extracted from PDF attachments** (`pdf-parse`)
- **GPA-aware filtering**: a scenario such as "CGPA above 3.0" only matches emails that mention a GPA at or above that value
- **Views** for All, Important (scenario matches), Remaining and Spam emails, plus a dashboard, profile and settings
- **Session security**: JWT stored in an HTTP-only cookie; protected API routes

## How matching works

1. The scenario text is normalised (lower-case, common misspellings fixed, `cgpa` → `gpa`) and stop words are removed.
2. A GPA requirement, if present, is extracted with regular expressions and handled as a numeric threshold.
3. For each non-spam email, the subject, body, attachment names and PDF text are combined into one token set. Tokens like registration numbers (e.g. `22MDBCS199`) are kept.
4. An email matches when it meets the GPA threshold (if any) and contains enough scenario keywords: all of them for one or two keywords, otherwise at least 60% (minimum two).
5. Matching emails are tagged with the scenario and moved to **Important**; everything else that is not spam stays in **Remaining**.

## Tech stack

| Layer | Technology |
| :--- | :--- |
| Frontend | React 19, Vite 7, React Router 7, Tailwind CSS 4, Axios, lucide-react, React Icons |
| Auth (client) | `@react-oauth/google` |
| Backend | Node.js, Express 5, Mongoose 9 |
| Google APIs | `googleapis` (Gmail API v1, OAuth 2.0) |
| Auth (server) | JSON Web Tokens, cookie-parser |
| Parsing | `pdf-parse` for PDF attachments |
| Database | MongoDB |

## Architecture

```text
React (Vite) ──Google sign-in──► auth code
     │
     ▼
Express API ──exchange code──► Google OAuth 2.0
     │  ├─ issue JWT (HTTP-only cookie)
     │  └─ start background Gmail sync ──► Gmail API (messages + attachments)
     │                                         │
     ▼                                         ▼
  MongoDB  ◄──────── emails, attachments metadata, scenarios, users
     ▲
     └─ scenario sweep: keywords + GPA + PDF text ──► category = important
```

```text
FYP-Mail-Miner/
├── backend/
│   ├── config/          # MongoDB connection
│   ├── controllers/     # auth (Google OAuth), emails (sync, views), scenarios (matching), users
│   ├── middleware/      # JWT auth guard
│   ├── models/          # User, Email, Scenario
│   ├── routes/          # /api/v1/auth, /users, /emails, /scenarios
│   ├── utils/           # OAuth2 client and scopes
│   ├── app.js
│   └── server.js
├── frontend/
│   └── src/
│       ├── components/  # Sidebar, Topbar, EmailCard
│       ├── pages/       # Dashboard, All/Important/Remaining/Spam, Scenario Manager, Profile, Settings, Login
│       └── services/    # Axios API client
└── images/              # screenshots
```

## Getting started

**Prerequisites:** Node.js 20+, MongoDB running locally, and a Google Cloud project with the **Gmail API** enabled and an **OAuth 2.0 Web client** (authorised JavaScript origin `http://localhost:5173`).

```bash
git clone https://github.com/dawoodcs195-spec/FYP-Mail-Miner.git
cd FYP-Mail-Miner
```

**Backend** (http://localhost:5000)

```bash
cd backend
npm install
cp .env.example .env     # add your Google OAuth client and secrets
npm run dev
```

**Frontend** (http://localhost:5173)

```bash
cd frontend
npm install
npm run dev
```

The frontend currently has the API base URL (`http://localhost:5000/api/v1`) and the Google client ID set in `src/services/api.js` and `src/App.jsx`; change them there if you use your own OAuth client.

## Environment variables

`backend/.env` (see `backend/.env.example`; names only):

| Variable | Purpose |
| :--- | :--- |
| `PORT` | API port (default 5000) |
| `NODE_ENV` | `development` / `production` |
| `MONGO_URI` | MongoDB connection string |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `JWT_SECRET` | Signs session tokens |
| `JWT_TIMEOUT` | Token lifetime, e.g. `7d` |

## API overview

Base path: `/api/v1`

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/auth/google?code=` | Exchange Google auth code, create user, set JWT cookie, start Gmail sync |
| `POST` | `/auth/logout` | Clear session cookie |
| `GET` | `/users/me` | Current user |
| `GET` | `/emails/all` | All synced emails |
| `GET` | `/emails/important` | Emails matched by the active scenario |
| `GET` | `/emails/remaining` | Non-spam emails with no match |
| `GET` | `/emails/spam` | Emails labelled spam by Gmail |
| `GET` | `/emails/scenario/:id` | Emails matched by one scenario |
| `GET` | `/emails/fetch` | Sync status message |
| `POST` | `/scenarios/create` | Save a scenario and re-scan the mailbox |
| `GET` | `/scenarios` | List the user's scenarios |

## Roadmap

- Move the API URL and Google client ID into Vite environment variables
- Incremental sync with Gmail history IDs instead of a full re-scan
- Store attachments in cloud storage rather than on local disk
- Optional ML / LLM-based classification alongside the rule-based matcher
- Automated tests and deployment

## Team

Built as a three-member Final Year Project at UET Mardan.

- **Muhammad Dawood**: [dawood.cs195@gmail.com](mailto:dawood.cs195@gmail.com) · GitHub [@dawoodcs195-spec](https://github.com/dawoodcs195-spec)
- **Mujahid Ullah**: GitHub [@mujahidullah712](https://github.com/mujahidullah712)
- GitHub [@Haroon-Sajjad12](https://github.com/Haroon-Sajjad12)

Developed for educational purposes as part of the BS Computer Science programme.
