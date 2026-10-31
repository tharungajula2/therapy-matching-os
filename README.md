# Therapy Matching OS

### A client-side product prototype for exploring therapist fit

Therapy Matching OS turns a short intake flow into an explainable, ranked shortlist from a **seeded therapist catalog**. It explores how language, preferences, concerns, logistics, and working style might shape a more thoughtful discovery experience for people seeking therapy in an Indian context.

> **Portfolio prototype, not a care service.** The profiles and outcomes in the repository are illustrative. Scores are hand-weighted compatibility indices, **not clinical probabilities or validated recommendations**. No therapist credentials, certification flags, availability, or outcome figures have been independently verified. Do not enter real health information into this demo.

[Explore the implementation](PROJECT_BLUEPRINT.md) · [Matching engine](lib/engine/matching.ts) · [Historical research and design notes](SOURCE_OF_TRUTH.md)

## What you can explore

| Experience | What the code actually does |
| --- | --- |
| Guided intake | A multi-step questionnaire converts selected answers into an in-browser profile; a demo autofill is available. A self-harm selection routes to a support screen. |
| Ranked matches | Hard filters and 27 positive scoring dimensions rank up to three of 16 seeded therapist profiles; a separate anti-match rule can subtract 25 points. The results screen can switch among 22 seeded personas. |
| Explainability | The engine names leading score contributions and assembles rule-based “why matched” text. |
| Journey and feedback | Mood, four-slider check-ins, three-question session ratings, and reflections update React state and browser sessionStorage. A dashboard illustrates an alliance alert from the latest rating. |
| Exploration surfaces | A persona sandbox, preference view, research-story slides, and therapist detail pages expose the concept from different angles. |

The UI also includes a scripted assistant, a booking confirmation mock, and “coming soon” settings and privacy screens. There is **no live booking, clinician communication, AI model, backend, or database** in this source. The feedback weight-update function exists but is not wired into the rating flow; matching uses the default weights each time. See the [implementation map and limitations](PROJECT_BLUEPRINT.md#implementation-status) for details.

## How matching works

~~~mermaid
flowchart LR
  A[Questionnaire or seeded persona] --> B[User profile]
  B --> C[Hard filters]
  D[16 seeded therapists] --> C
  C --> E[Weighted compatibility scores]
  E --> F[Anti-match penalty]
  F --> G[Top three and rule-based explanations]
~~~

Language overlap, selected affirmative-care and trauma flags, high-severity credential flags, and a strong gender preference are non-relaxable gates. Availability, format, and fee gates relax in that order if the first pass finds nobody. Remaining candidates receive a weighted average across dimensions such as concern overlap, style, culture, and logistics. The [blueprint](PROJECT_BLUEPRINT.md#matching-method-in-detail) gives the exact rules, example, and caveats.

## Stack and local exploration

| Layer | Implementation |
| --- | --- |
| App | Next.js 16 App Router, React 19, TypeScript |
| UI | Tailwind CSS 4, Lucide icons, local page components |
| State | React Context plus browser sessionStorage for profile and logs |
| Data and ranking | Checked-in TypeScript arrays and synchronous, client-side functions |

~~~bash
npm ci
npm run dev
~~~

Open http://localhost:3000. Start at /onboarding, use **Demo** autofill, or visit /matches to inspect a seeded persona. /admin shows the persona shortlist matrix. These commands are supplied by [package.json](package.json); this documentation review inspected source and did not run or deploy the app.

## Project status

This is a **high-fidelity concept and interaction demo**, useful for discussing product tradeoffs and explainable ranking. It is not ready for clinical use: intake-to-taxonomy mappings need correction, safety and clinician verification need expert review, and the privacy, booking, feedback-learning, and outcome claims need real implementation and validation. The [project blueprint](PROJECT_BLUEPRINT.md) separates shipped behavior, illustrative assumptions, and possible next steps.
