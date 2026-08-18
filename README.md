# ApexEducation

![TypeScript](https://img.shields.io/badge/TypeScript-Strict-3178c6?style=for-the-badge&logo=typescript&logoColor=fff)
![Next.js](https://img.shields.io/badge/Next.js-14%20App%20Router-000?style=for-the-badge&logo=nextdotjs&logoColor=fff)
![Supabase](https://img.shields.io/badge/Supabase-Postgres%20%2B%20Auth-3ecf8e?style=for-the-badge&logo=supabase&logoColor=fff)
![OpenAI](https://img.shields.io/badge/OpenAI-AI%20Tutor-111827?style=for-the-badge&logo=openai&logoColor=fff)
![Stripe](https://img.shields.io/badge/Stripe-Subscriptions-635bff?style=for-the-badge&logo=stripe&logoColor=fff)
![Vercel](https://img.shields.io/badge/Vercel-Serverless%20API-000?style=for-the-badge&logo=vercel&logoColor=fff)

ApexEducation is a web app for International Baccalaureate students preparing for their final Mathematics exams. A student picks a topic, level, and difficulty, and the app generates an original IB-style question, accepts a written answer, and returns a graded breakdown from an AI examiner. It also includes a Socratic tutor chat, a placement quiz, and public leaderboards across multiple subjects.

This project was built to explore how large language models can make high-quality exam preparation available to students who cannot afford private tutoring. Instead of relying on leaked past-paper banks or static PDFs, ApexEducation produces fresh questions on demand and grades free-response work against the actual IB mark scheme.

## Project Goals

- Give IB students unlimited exam-style practice without reproducing copyrighted past papers.
- Use structured AI prompts to generate questions and marking that match the IB style at both SL and HL.
- Pair every graded answer with a full worked solution and specific, honest feedback.
- Design the app so the model's limits are visible rather than hidden — confidence, cost tier, and provider are surfaced.
- Build a full-stack prototype that combines a Next.js frontend, a Supabase backend, provider-agnostic AI routing, and a Stripe subscription flow.

## Features

### Task Generation

Students choose a subject (Math AA, Math AI, Physics, Chemistry), a level (SL or HL), a syllabus topic, and a difficulty. A serverless endpoint calls the AI provider with a strict system prompt that returns a JSON payload containing the question, mark allocation, hints, and a target grade. All output is rendered with KaTeX so math notation looks like a real exam.

### Answer Checking

The student writes their answer in plain text or LaTeX. The app sends the question, the total marks, and the student's response to an examiner prompt that returns a score out of the question's marks, an IB grade from 1 to 7, a short summary of what went well, specific improvements, and a full step-by-step worked solution. Method marks are awarded even when the final numeric answer is wrong.

### AI Tutor Chat

A streaming chat page hosts a Socratic tutor that stays inside the IB syllabus. It guides the student toward the answer through questions instead of dumping full solutions, uses LaTeX for math, and politely refuses off-topic requests. Streaming is done token by token so the user sees the response as it is generated.

### Placement Quiz and Leaderboards

A short onboarding quiz seeds each new user's `task_attempts` history so the dashboard has real data on day one. Public leaderboards then rank users across four tabs (all-time, weekly, subject, level) using Postgres views on top of the same table.

### Multi-Provider AI Routing

Every AI call goes through a single provider layer (`lib/ai/provider.ts::callAI`). The app currently runs on OpenAI only because of a temporary billing issue on the Anthropic side, and the Anthropic client is left implemented but unwired behind a `TODO` so the switch is a one-line change. Task routing decides per call whether to use the quality tier (`gpt-5.6-terra`) for grading and long explanations, or the cheap tier (`gpt-5.6-luna`) for hints, classification, and chat.

### Free and Pro Plans

Free users get five graded tasks per day, tracked on the user profile with a daily reset. Pro users get unlimited tasks for a fixed monthly price through Stripe Checkout, with a webhook keeping the `profiles.plan` column in sync with the subscription status.

## Tech Stack

| Area | Technology |
| --- | --- |
| Frontend | Next.js 14 App Router, TypeScript, Tailwind CSS, shadcn/ui |
| Math rendering | KaTeX |
| Auth and database | Supabase (Postgres, Row Level Security, Auth, OAuth) |
| AI providers | OpenAI Responses API (Anthropic client parked behind a feature flag) |
| Payments | Stripe Checkout and webhooks |
| Deployment | Vercel serverless functions |
| Package management | npm |

## How It Works

1. The user signs up through Supabase Auth (email or Google OAuth) and lands on the dashboard.
2. A new user is offered a short placement quiz that writes seed rows into `task_attempts`.
3. On the practice page, the user picks subject, level, topic, and difficulty and clicks **Generate Task**.
4. The `/api/ai/generate` route calls the cheap-tier model with a strict JSON prompt and returns the question.
5. The question is rendered with KaTeX; the student writes an answer and clicks **Submit**.
6. The `/api/ai/check` route calls the quality-tier model with the examiner prompt and returns a graded JSON response.
7. The dashboard writes the attempt to `task_attempts`, and the free-tier counter on `profiles` is incremented for free users.
8. The tutor chat page streams responses from `/api/ai/chat` and stores the conversation in `chat_sessions`.
9. Leaderboards read from Postgres views over `task_attempts` and update in near-real time.
10. Stripe webhooks flip a user's `plan` column between `free` and `pro` as subscriptions start, renew, or cancel.

## Project Structure

```text
ibprep/
├── app/
│   ├── (auth)/              # login and signup pages
│   ├── (dashboard)/         # dashboard, practice, chat, settings, leaderboard
│   └── api/
│       ├── ai/generate      # task generation endpoint
│       ├── ai/check         # answer grading endpoint
│       ├── ai/chat          # streaming tutor chat endpoint
│       └── stripe/          # checkout session and webhook handlers
├── components/
│   ├── ui/                  # shadcn/ui primitives
│   └── shared/              # reusable app components
├── lib/
│   ├── ai/                  # provider layer, routing, prompts, JSON repair
│   ├── supabase/            # server and browser clients
│   ├── stripe/              # Stripe helpers
│   └── subjects.ts          # central subject and topic registry
├── types/                   # shared TypeScript interfaces
├── supabase-full-setup.sql  # consolidated schema, policies, and views
└── README.md                # main project documentation
```

## Getting Started

### Prerequisites

- Node.js 18 or newer
- npm
- A Supabase project
- An OpenAI API key
- A Stripe account with a Pro price configured (only required for the paid flow)

### Installation

Clone the repository and install dependencies:

```bash
git clone <your-repository-url>
cd ibprep
npm install
```

### Database Setup

Open the Supabase SQL editor for your project and run the full setup script:

```text
supabase-full-setup.sql
```

This creates the `profiles`, `task_attempts`, and `chat_sessions` tables, the leaderboard views, and the row-level security policies used throughout the app.

### Environment Variables

Create `.env.local` in the project root and fill in the values below. The server refuses to start without `OPENAI_API_KEY` because there is no local fallback provider.

```bash
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

OPENAI_API_KEY=your_openai_api_key
OPENAI_MODEL_QUALITY=gpt-5.6-terra
OPENAI_MODEL_CHEAP=gpt-5.6-luna

STRIPE_SECRET_KEY=your_stripe_secret_key
STRIPE_WEBHOOK_SECRET=your_stripe_webhook_secret
STRIPE_PRO_PRICE_ID=your_stripe_price_id

NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### Running Locally

Start the dev server:

```bash
npm run dev
```

Then open:

```text
http://localhost:3000
```

For Stripe webhooks during local development, run the Stripe CLI in a second terminal:

```bash
stripe listen --forward-to localhost:3000/api/stripe/webhook
```

Before deploying, always run the production build to catch type errors:

```bash
npm run build
```

## Using the App

1. Sign up with email or Google.
2. Complete the short placement quiz so the dashboard has data.
3. On the practice page, pick subject, level, topic, and difficulty.
4. Click **Generate Task** and read the question.
5. Write your answer in the textarea. Use LaTeX for math if you want.
6. Click **Submit** to get a graded response with a worked solution.
7. Open the tutor chat when you get stuck on a concept.
8. Upgrade to Pro from the settings page for unlimited daily tasks.

## Current Limitations

ApexEducation is an early-stage product, so it is designed to be transparent about what it can and cannot do yet.

- Grading is done by an LLM, not by a certified IB examiner, so scores are indicative rather than official.
- The app is currently locked to a single AI provider (OpenAI) while the Anthropic billing issue is resolved.
- Generated questions are IB-style but not IB-authored, and edge cases in wording still occur.
- Diagram-based questions are limited because the current models cannot yet produce reliable exam-quality figures.
- The free-tier counter is per profile row, so it is not a strong rate limit against a determined abuser.
- Leaderboards are cosmetic and are not tied to any verified identity.

These constraints are surfaced in the interface and in the AI prompts, and are the top of the roadmap below.

## Design Decisions

### Provider-Agnostic AI Layer

Every AI call in the app goes through a single `callAI` function, and every route only names a task (`generate`, `check`, `chat`, `hint`) instead of a model. Routing decides the tier and the provider. This kept the app usable when the Anthropic billing issue hit — the swap back is a one-line change in the routing table.

### Quality vs Cheap Tier Split

Grading long free-response answers and producing worked solutions is done by the higher-quality model, because a wrong mark scheme is worse than no mark scheme. Hints, classification, and chat run on the cheaper model to keep per-user cost sustainable on the free tier.

### JSON Repair in the Provider Layer

LLMs occasionally emit LaTeX inside JSON payloads that breaks strict parsers because of collisions between JSON escapes and TeX escapes (`\frac`, `\beta`, and so on). The provider layer runs a `parseJsonLoose` step that repairs these before the payload reaches the route, which keeps the UI from erroring on otherwise valid answers.

### Row-Level Security by Default

Every user-owned table has RLS enabled, and every policy scopes rows to `auth.uid()`. The service-role key is only used from server routes that need to bypass RLS on purpose, such as the Stripe webhook.

## Future Improvements

- Re-enable Anthropic as an alternate quality-tier provider once billing is restored.
- Add IB Chemistry and Physics Paper 3 style questions with diagrams.
- Add a spaced-repetition mode that re-serves topics the student scored below 4 on.
- Store per-topic mastery scores and drive the dashboard from them.
- Add a coach mode that lets teachers see anonymised class-wide weak areas.
- Add exportable revision packs (PDF) built from a student's weakest topics.
- Improve mobile input for math so students can write answers without leaving the keyboard.

## What I Learned

This project combined several areas of software engineering:

- server-rendered React with the Next.js App Router
- Postgres schema design with row-level security and views
- structured prompt design for both generation and grading
- streaming API responses and incremental UI rendering
- payments integration with Stripe Checkout and webhooks
- building a provider-agnostic AI layer that survives an outage on one vendor
- writing prompts that ask the model to be honest about uncertainty instead of hiding it

The most important lesson was that shipping an AI-powered product is mostly not about the model. It is about the boundary layer — validating inputs, routing calls, parsing outputs safely, and telling the user honestly when the system is unsure.

## Responsible Use

ApexEducation is an educational tool. AI-generated grading is intended as practice feedback and should not be used as an official predicted grade, a university application artifact, or a replacement for a qualified teacher. Students should combine the app's feedback with their school's own materials and their teacher's judgment.

## Author

Built by Kirill Kalinin as a student software project exploring how AI can widen access to high-quality exam preparation for International Baccalaureate students.
