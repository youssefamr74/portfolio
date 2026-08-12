# Jōb

A job search companion app. It reads your Gmail, figures out what happened with each application, and tracks everything automatically instead of you updating a spreadsheet by hand.

Live on Google Play: play.google.com/store/apps/details?id=com.youssefibrahim.job

Built solo. React Native/Expo on the frontend, FastAPI and PostgreSQL on the backend, Anthropic's Claude API doing the classification work.

## Screenshots

### Home
<img src="screenshots/01-home.png" width="280">

The main dashboard. Shows today's application count, response rate, and a rotating "companion" card that reacts to whatever happened most recently in your inbox (a rejection, an interview invite, etc). This is the first thing you see, so it's designed to answer "where do I actually stand right now" in one glance, without needing to dig through a list.

### Applications list
<img src="screenshots/02-applications-list.png" width="280">

Every tracked application in one scrollable list, each with its current status (Applied, Interview, Assessment, Offer, Rejected). This is the core of what the app replaces: a manually updated spreadsheet. Every entry here got there automatically from a Gmail sync, not a manual add.

### Application detail
<img src="screenshots/03-application-detail.png" width="280">

A single application's full timeline: when you applied, every status change since, and the underlying email history that led to each change. This solves a specific problem I had myself: once you've applied to 50+ places, you forget the details of any one of them by the time something happens. This screen exists so you never have to reconstruct that context from memory.

### Insights
<img src="screenshots/04-insights.png" width="280">

Aggregate stats across your whole search: response rate, best/worst days to apply, which roles are actually converting versus which ones you're applying to most. This is the difference between a tracker and something that actually helps you improve your approach. A spreadsheet tells you what happened. This tells you what's working.

### AI companion
<img src="screenshots/05-job-ai.png" width="280">

A chat interface with access to your actual application data, not generic career advice. You can ask it things like "why am I not getting interviews" and it answers using your real response rate, role performance, and history, not a canned response.

### Weekly review
<img src="screenshots/06-weekly-review.png" width="280">

A weekly summary: applications sent, responses received, and an honest note about momentum (if your volume dropped compared to the week before, it says so). This exists because job searching is a long, low-feedback process, and it's easy to lose track of whether you're actually keeping pace week to week.

## The interesting part: how it reads your inbox

Most tools like this would throw every email at an LLM and let it sort things out. That's slow, it's expensive at any real scale, and it doesn't actually prove you understand the problem you're solving.

I built it differently. Before anything touches a model, most emails get resolved by a layered, deterministic system:

1. **A cheap filter kills the obvious noise first.** Newsletters, spam, anything clearly unrelated to a job application never gets any further processing. This costs nothing and runs instantly.

2. **A rule engine handles the majority of what's left.** I wrote pattern-matching logic for 12+ ATS platforms (Greenhouse, Lever, Workday, SmartRecruiters, and others), extracting company, role, and status from how each platform actually structures its emails. Status detection uses multilingual regex, since rejection and interview language doesn't only show up in English. Every extraction gets a confidence score, weighted across company, role, status, and completeness. There's also a template cache so a recurring email format from the same ATS gets classified instantly on repeat.

3. **Claude only gets called when the rule engine genuinely can't decide.** If confidence doesn't clear a threshold, the email goes to the model with the thread history as context. Most emails never reach this step at all.

<img src="screenshots/10-classifier-pipeline.png" width="700">

The diagram above shows the full flow, including the entity-matching cascade and the business rules that come after classification.

Why this matters more to me than it might sound: it's the part of the project where I actually had to think, not just wire an API together. Deciding when a model is worth calling, and building something reliable enough that it usually doesn't need to be, was the real engineering problem.

## What else is in there

- **Automated tracking.** Applications get logged, matched, and updated without manual entry, using a matching cascade (thread ID, then job ID, then a company+role+date fingerprint, then a recency-based company match) so the same application doesn't get duplicated across a long email thread.
- **Status protection.** Once something is marked Rejected or Offer, the system won't silently flip it back on a low-confidence re-read. If it's going to affect how someone feels about their own job search, it has to be right.
- **Interview prep and a companion chat** that has access to your actual application data, not generic advice.

None of this was a spec I was handed. I built it because I was job hunting myself and got tired of an inbox that told me nothing back.

## Stack

- **Mobile:** React Native, Expo
- **Backend:** FastAPI, PostgreSQL, async SQLAlchemy
- **AI:** Anthropic API (Claude Haiku for classification and coaching)
- **Auth:** Google OAuth, Gmail API integration
- **Infra:** Railway, GitHub Actions

## Running it locally

See `.env.example` for the required environment variables (Gmail OAuth credentials, Anthropic API key, database URL). Real credentials are not included in this repo.

\`\`\`
# backend
cd backend
pip install -r requirements.txt
uvicorn main:app --reload

# mobile
cd mobile
npm install
npx expo start
\`\`\`