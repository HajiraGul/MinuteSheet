# Collaborative Minute Sheet

An internal web application for **FFC (Fauji Fertilizer Company)** that digitises the minute-sheet process: employees author minute sheets, route them through a configurable **multi-step review/approval chain**, collaborate through comments and sharing, and use **AI** to summarise attachments, extract action items, draft next-meeting agendas, and dictate content in **English or Urdu**.

Built with **.NET 8 Blazor Web App (Interactive Server)**, **ASP.NET Core Identity**, **Entity Framework Core** and **SQL Server**.

---

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Getting started](#getting-started)
- [Configuration reference](#configuration-reference)
- [Seed data and default accounts](#seed-data-and-default-accounts)
- [Running the tests](#running-the-tests)
- [Project structure](#project-structure)
- [Data model](#data-model)
- [Application routes](#application-routes)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

---

## Features

### Authentication and accounts
- Self-service **registration** with employee details: full name, employee number, designation and department.
- **Email verification by 6-digit OTP**. Unconfirmed accounts cannot sign in.
- **Two-step login**: sign in with **email or employee number** and password, then enter a one-time code sent by email.
- Optional **authenticator-app 2FA** (QR code via QRCoder), recovery codes, password reset and change.
- **Profile settings** with avatar upload.
- Two roles, **Admin** and **Employee**. Accounts without a role are backfilled as Employee on startup.

### Minute sheets (`/dashboard/create`)
- Title, **category** (Financial / Non-Financial), with **amount and currency** for financial sheets.
- **Initiating department** and **intended-for department**.
- **Confidential** flag: confidential sheets are visible only to the creator, approvers, admins and users the sheet was explicitly shared with.
- **Rich-text description** in a self-hosted **Quill** editor.
- **Attachment** upload (`.pdf`, `.doc`, `.docx`, up to 10 MB), stored under `wwwroot/uploads/`.
- **Dynamic approval chain**: add or remove rows, choose an approver and mark each step as **Review** or **Approve**. The last step is always **Approve**.
- Download the sheet's summary and description as **Word (.docx / .doc)** or **PDF**, branded with the FFC logo.

### Hierarchical approval workflow (`/dashboard/sheet/{token}`)
- Steps run **in order**: only the lowest-numbered unapproved step can act. Later approvers wait.
- **Review** needs a comment and sends the sheet back to the creator. **Approve** does not need a comment.
- **Review → resolve loop**: the creator resolves the review, can make limited edits, and the reviewer is notified to look again.
- A sheet becomes **Approved** only when every step is approved. Status is shown as `Pending`, `Review` or `Approved`.
- Every sheet is reached through an unguessable GUID link.

### Collaboration
- **Share** a sheet with any user by email. Sharing gives **read-only** access, and the email includes a **PDF copy** generated with QuestPDF.
- Users a sheet was shared with can send **suggestions** or an **"It's all OK"** acknowledgement.
- A full **comment thread** records reviews, approvals and resolutions.

### Email notifications
Emails are queued and sent by a background service, so workflow actions stay fast. Notifications go out when:
- a sheet is submitted or updated (to every approver),
- a reviewer leaves a review (to the creator),
- the creator resolves a review (to the reviewer),
- a sheet is fully approved (to the creator),
- a sheet is shared (to the recipient, with a PDF attached).

Delivery uses the **Brevo HTTP API** when an API key is set and falls back to **SMTP (MailKit)**.

### AI features (OpenRouter)
All LLM calls go through OpenRouter using `google/gemma-4-26b-a4b-it:free`, with Polly retries (exponential backoff on transient errors and HTTP 429).
- **Document summarisation**: extracts text from PDF (PdfPig) and Word (OpenXml) attachments, combines it with the description, and produces an English summary.
- **Action-item extraction** and **next-meeting agenda** generation.
- **Urdu → English translation**.
- An **LLM safety net** rejects "meta" responses (for example *"please provide…"*) and falls back to the original text.

### Voice dictation (English and Urdu)
The browser records audio with `MediaRecorder` and posts it to `POST /api/transcriptions`, which runs this pipeline:

1. Load the admin-managed **domain vocabulary** (people, products, platforms, technologies, abbreviations).
2. **Transcribe** with **Groq Whisper** (`whisper-large-v3`), using the vocabulary as a prompt. If Groq fails or is not chosen, it falls back to a **local Whisper CLI**.
3. Skip the AI steps for very short transcripts (fewer than 3 words).
4. **Correct** the transcript against the vocabulary with an LLM.
5. Check whether the text is English, using an Urdu-script and Roman-Urdu heuristic.
6. **Translate** Urdu, Roman Urdu or mixed text into professional English.

### Admin panel
- **Dashboard** with an overview chart of sheet statuses.
- **All sheets**: every minute sheet in the system.
- **Users**: list, filter by role, create and edit users.
- **Departments**: create and manage departments.
- **Vocabulary**: manage the domain terms and aliases used by dictation.

---

## Tech stack

| Area | Technology |
|------|------------|
| Framework | .NET 8, Blazor Web App (Interactive Server render mode) |
| Auth | ASP.NET Core Identity with roles, email OTP, TOTP 2FA (QRCoder) |
| Data | Entity Framework Core 8 + SQL Server (LocalDB by default) |
| Email | Brevo HTTP API or SMTP via MailKit, with a background queue |
| Rich text | Quill (self-hosted in `wwwroot/lib/quill`) |
| AI / LLM | OpenRouter API (`google/gemma-4-26b-a4b-it:free`) |
| Speech-to-text | Groq Whisper API (`whisper-large-v3`), with local OpenAI Whisper CLI as fallback |
| Document parsing | UglyToad.PdfPig, DocumentFormat.OpenXml |
| Document generation | QuestPDF (share PDFs), PDFsharp + OpenXml + SkiaSharp (summary exports) |
| Resilience | Polly (`Microsoft.Extensions.Http.Polly`) |
| UI | Bootstrap 5 + custom CSS, Poppins font, brand colour `#000066` |
| Tests | xUnit, Moq, EF Core InMemory |

---

## Architecture

```
Browser (Blazor Server over SignalR)
   │
   ├── Razor components (Pages/Dashboard, Pages/Admin, Account)
   │       │
   │       ├── ApplicationDbContext (EF Core) ──► SQL Server
   │       ├── EmailQueue ──► EmailBackgroundService ──► SmtpEmailSender ──► Brevo API / SMTP
   │       ├── DocumentSummarizationService ──► OpenRouterClient ──► OpenRouter API
   │       ├── SheetPdfService (QuestPDF) / SheetExporter (DOCX, DOC, PDF)
   │       └── ToastService
   │
   └── POST /api/transcriptions (minimal API, requires sign-in)
           └── DictationPipelineService
                 ├── GroqTranscriptionService ──► Groq API
                 ├── LocalWhisperTranscriptionService ──► whisper CLI (fallback)
                 ├── TranscriptCorrectionService ──► OpenRouter
                 └── TranslationService ──► OpenRouter
```

On startup the app **applies pending EF Core migrations automatically**, restores the development seed snapshot, and makes sure the roles and bootstrap admin account exist.

---

## Getting started

### Prerequisites

| Requirement | Needed for |
|-------------|------------|
| [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) | Building and running the app |
| SQL Server **LocalDB** (comes with Visual Studio) or any SQL Server instance | Database |
| [.NET 10 SDK](https://dotnet.microsoft.com/download) | Only for running the test project, which targets `net10.0` |
| Python 3, `ffmpeg` on `PATH`, `pip install -U openai-whisper setuptools-rust` | Only for the **local** Whisper fallback |
| OpenRouter API key | AI summary, action items, agenda, transcript correction, translation |
| Groq API key | Cloud speech-to-text (the default dictation provider) |
| Brevo API key **or** SMTP credentials | Sending OTP and notification emails |

> The app runs without the AI and email keys, but those features will fail or be skipped. **Email is required to log in**, because every login needs an emailed OTP.

### 1. Clone the repository

```bash
git clone https://github.com/HamzaSaeed7/minutesheet.git
cd minutesheet
```

### 2. Database

The default connection string in `minutesheet/appsettings.json` points to LocalDB:

```
Server=(localdb)\mssqllocaldb;Database=aspnet-minutesheet-685136f4-...;Trusted_Connection=True;MultipleActiveResultSets=true
```

You don't need to do anything else. Migrations run automatically the first time the app starts. To apply them by hand instead:

```bash
dotnet tool install --global dotnet-ef
dotnet ef database update --project minutesheet
```

### 3. Secrets

Never commit real keys. Store them with **user-secrets** (run these from the `minutesheet/` project folder):

```bash
cd minutesheet

# OpenRouter — one key per slot (the same key can be used for all four)
dotnet user-secrets set "OpenRouterSettings:ApiKeys:0" "sk-or-..."   # summary
dotnet user-secrets set "OpenRouterSettings:ApiKeys:1" "sk-or-..."   # action items + transcript correction
dotnet user-secrets set "OpenRouterSettings:ApiKeys:2" "sk-or-..."   # next-meeting agenda
dotnet user-secrets set "OpenRouterSettings:ApiKeys:3" "sk-or-..."   # Urdu → English translation

# Groq (speech-to-text)
dotnet user-secrets set "ApiKeys:Groq" "gsk_..."
```

**Email, option A: Brevo HTTP API (recommended, uses HTTPS port 443 so it is rarely blocked)**

```bash
dotnet user-secrets set "EmailSettings:ApiKey" "xkeysib-..."
dotnet user-secrets set "EmailSettings:From"   "verified-sender@yourdomain.com"
```

**Email, option B: SMTP (for example a Gmail app password)**

```bash
dotnet user-secrets set "EmailSettings:Host"     "smtp.gmail.com"
dotnet user-secrets set "EmailSettings:Port"     "587"
dotnet user-secrets set "EmailSettings:User"     "you@example.com"
dotnet user-secrets set "EmailSettings:Password" "your-app-password"
dotnet user-secrets set "EmailSettings:From"     "you@example.com"
```

Instead of user-secrets you can also use a `.env` file in the repo root or in `minutesheet/`. `run.bat` loads it, and it is git-ignored. Supported variables: `EMAIL_HOST`, `EMAIL_USER` / `GMAIL_APP_USER`, `EMAIL_PASSWORD` / `GMAIL_APP_PASSWORD`, or any `Section__Key` form such as `OpenRouterSettings__ApiKeys__0`.

### 4. Run

```bash
dotnet run --project minutesheet --launch-profile http
```

On Windows you can also double-click **`run.bat`**.

Open **http://localhost:5285**. With the `https` profile it is https://localhost:7054.

### 5. Sign in

Log in as `admin@ffc.com` with password `Abcd1234!` (see [Seed data](#seed-data-and-default-accounts)), then enter the OTP sent to that address. **Change this password** on any shared or deployed instance.

---

## Configuration reference

| Key | Description | Default |
|-----|-------------|---------|
| `ConnectionStrings:DefaultConnection` | SQL Server connection string | LocalDB |
| `EmailSettings:ApiKey` | Brevo API key (`xkeysib-…`). If set, Brevo is tried first | empty |
| `EmailSettings:Host` / `Port` / `User` / `Password` / `EnableSsl` | SMTP fallback settings | `smtp.gmail.com`, `587`, SSL on |
| `EmailSettings:From` / `FromName` | Sender address and display name | `Minute Sheet` |
| `OpenRouterSettings:ApiKeys:0..3` | OpenRouter keys per task (see above). `OpenRouterSettings:ApiKey` is used if a slot is empty | empty |
| `ApiKeys:Groq` | Groq API key for Whisper transcription | empty |
| `Dictation:TranscriptionProvider` | `Groq` (falls back to local on error) or `Local` | `Groq` |
| `LocalWhisper:Model` | Local Whisper model (`tiny`, `base`, `small`, …) | `base` |
| `LocalWhisper:Executable` | Path to the `whisper` executable (or env `LOCAL_WHISPER_EXECUTABLE`) | `whisper` |
| `Seed:LoadSnapshot` | Restore `Data/Seed/seed-data.json` on startup | `true` in Development, otherwise `false` |

---

## Seed data and default accounts

In **Development**, the app restores a snapshot from `minutesheet/Data/Seed/seed-data.json` on startup. The snapshot contains departments, users, roles, minute sheets, approval steps, comments, shares, suggestions and vocabulary, so a fresh clone starts with realistic data.

- **Every seeded account uses the password `Abcd1234!`.** The snapshot contains no password hashes.
- A bootstrap admin, **`admin@ffc.com` / `Abcd1234!`**, is always created if it is missing.
- Seeding is **idempotent**: rows that already exist (matched by their original primary key) are skipped.
- To turn it off: `dotnet user-secrets set "Seed:LoadSnapshot" "false"`.
- To refresh the snapshot, export each table with `FOR JSON PATH` into the same JSON shape (`departments`, `users`, `userRoles`, `sheets`, `approvalSteps`, `comments`, `shares`, `suggestions`, `vocabulary`).

---

## Running the tests

The `minutesheet.Tests` project (xUnit + Moq + EF Core InMemory) covers the dictation pipeline (language detection, provider fallback, skipping short transcripts) and the LLM safety net.

```bash
dotnet test minutesheet.Tests
```

> The test project targets **`net10.0`**, so it needs the .NET 10 SDK. It is not included in `minutesheet.slnx`, so run it by path.

---

## Project structure

```
.
├── minutesheet.slnx                    # Solution file
├── run.bat                             # Windows launcher (loads .env, runs http profile)
├── minutesheet/                        # Main Blazor web app
│   ├── Program.cs                      # DI, Identity, EF Core, email, Polly, /api/transcriptions
│   ├── appsettings.json                # Configuration (no secrets)
│   ├── Components/
│   │   ├── App.razor, Routes.razor     # Root document and router
│   │   ├── Layout/                     # MainLayout, NavMenu
│   │   ├── Account/                    # Identity pages, OTP login/verify, email sender and queue
│   │   └── Pages/
│   │       ├── Home.razor              # Dashboard / overview
│   │       ├── Settings.razor          # Profile, avatar, 2FA
│   │       ├── Dashboard/              # CreateSheet, SheetView, History, Actions
│   │       └── Admin/                  # AllSheets, Users, CreateUser, EditUser, Departments, Vocabulary
│   ├── Data/
│   │   ├── *.cs                        # Entities, enums, ApprovalWorkflow rules, DbSeeder
│   │   ├── Migrations/                 # EF Core migrations
│   │   └── Seed/                       # Snapshot seeder + seed-data.json
│   ├── Services/
│   │   ├── OpenRouter/                 # OpenRouter HTTP client
│   │   ├── DocumentSummarizationService.cs
│   │   ├── DictationPipelineService.cs
│   │   ├── GroqTranscriptionService.cs / LocalWhisperTranscriptionService.cs
│   │   ├── TranscriptCorrectionService.cs / TranslationService.cs / LlmSafetyNet.cs
│   │   ├── SheetPdfService.cs / SheetExporter.cs
│   │   └── ToastService.cs
│   └── wwwroot/                        # app.css, Bootstrap, Quill, quill-interop.js, images, uploads/
├── minutesheet.Tests/                  # xUnit tests
├── resources/images/                   # Brand assets
└── Fertilizer company login page/      # Login page design prototype
```

---

## Data model

| Entity | Key fields |
|--------|------------|
| **ApplicationUser** | Identity user + `FullName`, `EmployeeNo`, `Designation`, `DepartmentId`, `AvatarPath` |
| **Department** | `Name`, `EmployeeCount` |
| **MinuteSheet** | `Title`, `Category`, `Amount`, `Currency`, `DescriptionHtml`, `Summary`, `ActionItems` (JSON), `NextMeetingAgenda` (JSON), attachment name/path, `IsConfidential`, `Status`, `Token` (GUID), `DepartmentId`, `IntendedForDepartmentId`, creator, `CreatedAt` |
| **ApprovalStep** | `StepIndex`, `Action` (Review/Approve), `Status` (Pending/Reviewed/Approved), `ApproverEmail`, `ActedAt` |
| **SheetComment** | `Kind` (Review/Approval/Resolution), `Body`, author, optional `ApprovalStepId` |
| **SheetShare** | `SharedWithEmail`, `SharedByUserId`, `SharedAt` (read-only access) |
| **SheetSuggestion** | `Body` or `IsAllOk`, author |
| **DomainVocabularyTerm** | `Category` (Person/Product/Platform/Technology/Abbreviation), `Term`, `Aliases`, `IsActive` |

**Designations:** Executive, Senior Executive, Assistant Manager, Deputy Manager, Section Manager, Unit Manager.

---

## Application routes

| Route | Access | Purpose |
|-------|--------|---------|
| `/` | Signed in | Dashboard overview |
| `/dashboard/create` | Signed in | Create a minute sheet |
| `/dashboard/sheet/{token}` | Signed in, plus visibility rules | View, review, approve, share or comment on a sheet |
| `/dashboard/sheet/{token}/edit` | Creator | Edit a sheet |
| `/dashboard/history` | Signed in | Sheets you created |
| `/dashboard/actions` | Signed in | Items waiting for your approval, and reviews you need to resolve |
| `/settings`, `/settings/two-factor` | Signed in | Profile and 2FA |
| `/admin/sheets` | Admin | All minute sheets |
| `/admin/users`, `/admin/users/create`, `/admin/users/{id}/edit` | Admin | User management |
| `/admin/departments` | Admin | Department management |
| `/admin/vocabulary` | Admin | Dictation vocabulary |
| `/Account/Login`, `/Account/Register`, `/Account/LoginOtp`, `/Account/VerifyOtp` | Public | Authentication |
| `POST /api/transcriptions` | Signed in | Audio → English text (dictation pipeline) |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No OTP email arrives | Check the console for `No email transport configured`. Set the Brevo key or SMTP credentials. With Gmail, use an **app password**, not your account password. |
| `Cannot open database` / LocalDB errors | Run `sqllocaldb start mssqllocaldb`, or point `DefaultConnection` at another SQL Server instance. |
| AI summary returns an error or nothing | Make sure `OpenRouterSettings:ApiKeys:*` is set. Free models are rate-limited, so wait and try again. |
| Dictation returns `503` | Groq key is missing and the local Whisper fallback is not installed. Add `ApiKeys:Groq`, or install Python, ffmpeg and `openai-whisper`. |
| Local Whisper is very slow | Use a smaller model: `LocalWhisper:Model` = `base` or `tiny`. |
| Microphone does not work | Browsers only allow microphone access on `localhost` or HTTPS. |

---

## Security notes

- **Never commit secrets.** Keep keys in user-secrets, environment variables or a git-ignored `.env`.
- Change the default `Abcd1234!` passwords and turn off `Seed:LoadSnapshot` on any non-development deployment.
- Uploaded attachments and avatars (`wwwroot/uploads/`) are runtime data and are git-ignored.
- QuestPDF runs under its **Community licence**. Check that your organisation qualifies before using it in production.

---

*Developed during the FFC internship programme.*
