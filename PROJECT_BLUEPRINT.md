# Therapy Matching OS — Project Blueprint

**Reading guide:** This manual describes the checked-in implementation. Start with the [README](README.md) for a short showcase. [SOURCE_OF_TRUTH.md](SOURCE_OF_TRUTH.md) preserves earlier product research and design intent; code, not that historical narrative, determines the behavior described here.

> **Scope and safety:** This is an illustrative therapy-discovery prototype. The numeric match is a hand-built ranking index, not a measured probability of therapeutic alliance, clinical suitability, diagnosis, or outcome. Seeded clinician names, registration strings, certification flags, fees, and outcome priors are unverified examples. The repository contains no verified provider network or live care workflow. Use invented inputs only.

## Product and audience

The prototype asks a prospective therapy client about concerns, preferred interaction style, identity and cultural context, budget, and session format. It then ranks a small local catalog and explains selected matches. A second set of screens imagines later use: check-ins, session feedback, reflections, rematching, and a clinician-facing alert. Its likely audience is portfolio reviewers and product or engineering collaborators examining an Indian-context mental health discovery concept. It is a **concept demo**, not a deployed clinical matching service.

The implementation includes [22 seeded user personas](data/users.ts), [16 seeded therapist records](data/therapists.ts), and a [64-tag concern taxonomy](data/taxonomy.ts) grouped into 12 numbered categories. Those counts describe source entries, not customers, providers, or validated clinical cohorts. The historical [research document](SOURCE_OF_TRUTH.md) describes broader aspirations and literature claims that this audit did not independently verify.

## Implementation status

| Area | Implemented in source | Boundary |
| --- | --- | --- |
| Intake and safety detour | [/onboarding](app/onboarding/page.tsx) collects answers, can autofill a demo, maps fields to UserProfile, and routes a self-harm choice to [/support](app/support/page.tsx). | Mapping is coarse and has incorrect concern codes; the support page is a static resource list, not a triage service. Resource availability was not verified in this audit. |
| Ranking | [matching.ts](lib/engine/matching.ts) runs hard gates, relaxes three logistics gates, scores surviving therapists, applies an anti-match penalty, and returns three results. | Weights and rules are manually set; no training or clinical validation is shown. |
| Results | [/matches](app/matches/page.tsx) switches between an intake profile and 22 personas; [/match](app/match/page.tsx) renders a top-three therapist detail. | A requested ID outside the shortlist falls back to the first match. Neither score nor explanation proves care quality. |
| Progress and feedback | [/journey](app/journey/page.tsx), [/checkin](app/checkin/page.tsx), [/feedback](app/feedback/page.tsx), and [/reflections](app/reflections/page.tsx) save local logs and show simple trends. | Logs begin with seeded example ORS/SRS entries. No clinician receives a notification. The feedback form does not call the weight-update helper. |
| Clinician / admin views | [/therapist-dashboard](app/therapist-dashboard/page.tsx) shows a seeded clinician and a low-rating alert; [/admin](app/admin/page.tsx) displays each persona's **top-three** matches in a 16-column table. | No authentication or role boundary. Blank/zero cells in the sandbox may simply mean “outside the top three,” not an evaluated zero score. |
| Other surfaces | [/compare](app/compare/page.tsx) tells a research story; [/preferences](app/preferences/page.tsx) displays profile values; [/rematch](app/rematch/page.tsx) links back to alternatives. | Research claims in slides are not runtime validation; rematch does not recalibrate weights. |
| Simulations | [AIAssistant](components/AIAssistant.tsx) returns a fixed delayed reply; [/booked](app/booked/page.tsx) displays a random reference; [/listens](app/listens/page.tsx) describes an offering. | No model call, booking request, email, trained listener, or payment flow. [/settings](app/settings/page.tsx), [/privacy](app/privacy/page.tsx), and [/notifications](app/notifications/page.tsx) are largely placeholders. |

## Architecture and data flow

~~~mermaid
flowchart TD
  UI[Next.js App Router pages] --> CTX[TherapyProvider: React Context]
  UI --> ENGINE[Client-side matching functions]
  UI --> FIXTURES[Seeded users, therapists, taxonomy, prompts]
  ENGINE --> FIXTURES
  ENGINE --> EXPLAIN[Rule-based explanation templates]
  CTX <--> SS[Browser sessionStorage]
  UI --> SS
~~~

The [root layout](app/layout.tsx) wraps pages in [TherapyProvider](contexts/TherapyContext.tsx) and [LayoutClient](components/LayoutClient.tsx). The latter chooses when to show the [bottom navigation](components/BottomNav.tsx) and scripted assistant. Page components marked as client components run browser interactions. The [App Router](app/) contains no app/api route in this clone; ranking reads the checked-in arrays directly in the client. There is no network-backed application data store in source. Next.js may load its configured Google fonts during development/build, so “no backend” does not mean all framework activity is offline.

| Directory / file | Responsibility |
| --- | --- |
| [app/](app/) | Route screens, page-local controls, and navigation. |
| [types/engine.ts](types/engine.ts) | Rich UserProfile, TherapistProfile, CNIP, and result contracts. |
| [types/index.ts](types/index.ts) | Narrow intake and older display schemas. |
| [data/users.ts](data/users.ts), [data/therapists.ts](data/therapists.ts) | Hard-coded research personas and therapist examples. |
| [data/taxonomy.ts](data/taxonomy.ts), [data/reflections.ts](data/reflections.ts) | Concern labels, modality names, and 30 reflection prompts. |
| [lib/engine/matching.ts](lib/engine/matching.ts) | Eligibility, scoring, top-three selection, and an unused feedback-update helper. |
| [lib/engine/explainMatch.ts](lib/engine/explainMatch.ts) | Names and sentence templates for selected dimensions. |
| [lib/clinical.ts](lib/clinical.ts) | ORS interpretation copy and two exported thresholds; the thresholds are not consumed elsewhere. |
| [contexts/TherapyContext.tsx](contexts/TherapyContext.tsx) | Profile and local mood, rating, ORS, and reflection histories. |

### Main user path

~~~mermaid
flowchart LR
  HOME[Home] --> INTAKE[Onboarding]
  INTAKE -->|Self-harm choice| SUPPORT[Support screen]
  INTAKE -->|Listening preference| LISTENS[Listens concept]
  INTAKE -->|Complete| MATCHES[Ranked matches]
  MATCHES --> DETAIL[Therapist detail]
  DETAIL --> BOOKED[Booking mock]
  BOOKED --> JOURNEY[Journey]
  JOURNEY --> CHECKIN[ORS-like check-in]
  JOURNEY --> FEEDBACK[SRS-like rating]
  JOURNEY --> REFLECT[Reflection]
~~~

The default seeded profile is used when there is no stored intake on the results screen. The detail page also defaults to the first persona and re-runs the matcher, so a direct visit can show a demo match. The booked page and journey mention a hard-coded clinician who need not be the selected match.

## Intake and profile construction

[OnboardingData](types/index.ts) models the form's visible answers. [mapToEngineProfile](app/onboarding/page.tsx) translates them to the richer engine contract and supplies defaults for fields the form does not ask. For example, it fixes attachmentStyle to Secure, stageOfChange to Preparation, timePref to Flexible, and sexuality to straight; cultural sensitivities and trigger lists are empty. Thus the full persona schema is richer than the live questionnaire. A selected queer-affirming gender preference is represented as NB in the preference field, not the user sexuality field; the affirmative-care hard gate only reads sexuality and user gender.

The concern map needs special care. The option **“Workplace burnout”** maps to 5.3, whose taxonomy label is **career identity / purpose**; **“Feeling down or depressed”** maps to 1.1, **generalized anxiety**; **“Thoughts of self-harm or suicide”** maps to 1.2, **social anxiety**, although selection also triggers the support route. These are code-level mismatches, not clinical interpretations. The form should not be used to guide real care until intake labels, taxonomy, and safety handling have been reviewed by qualified experts.

## Matching method in detail

The engine is deterministic for a given profile, catalog, and weights. It is a demonstration of an explainable ranking pipeline, **not a machine-learning model**.

### Layer 1 — eligibility gates

[passesAllHardFilters](lib/engine/matching.ts) requires at least one exact shared language. It also gates on qacpCertified for a small hard-coded set of sexuality/gender strings; a trauma specialization or EMDR modality when a 3.* primary concern has severity at least 7; a nonempty rciNumber or psychiatrist flag when severity is at least 8; and a therapist gender match for Strong F or Strong M. These are literal data flags, not verified licenses, certification checks, or clinical safety assurances. The limited exact string lists can miss identities outside their enumerated values.

At first, format overlap, fee at most 110% of maximum budget, and availability overlap are also required. If no candidate survives, the engine retries after relaxing **availability**, then **format**, then **fee**. Language and the other five gates never relax. A final fourth retry duplicates the third relaxation level; if still empty, the matcher returns an empty array. The UI handles empty results on /matches, but other screens assume a first match exists.

### Layer 2 — compatibility index

[DEFAULT_WEIGHTS](lib/engine/matching.ts) has W1–W27 positive weights summing to **111**. Each dimension score is intended to be 0–100. The final number is:

~~~text
weighted_index = sum(score[Wi] * weight[Wi] for i = 1..27) / 111
if W28 anti-match is true: weighted_index -= 25
display_score = max(0, round(weighted_index))
~~~

W28 has a configured weight but is excluded from the average; it triggers the flat 25-point subtraction when a primary concern appears in therapist.strugglesWith. The score is neither calibrated nor a probability, even though some UI copy calls it one. The three highest display scores are returned. The code has no explicit tie breaker beyond array order.

| Dimension group | Actual rule examples | Limits |
| --- | --- | --- |
| Concern and approach | W1 measures exact overlap with a 20-point capped bonus for the top concern in a therapist's first three specializations; W8 gives selected modality/concern combinations 100, otherwise 80. | Taxonomy mapping errors flow into these rules; no evidence of clinical efficacy is computed. |
| Style | W2–W5 use 100 − abs(user − therapist) / 30 × 100 for four CNIP coordinates; W6 uses stage-of-change/directiveness thresholds; W7 uses a short attachment-style switch. | These are hand-set representations and cutoffs, not a validated C-NIP instrument. |
| Identity and culture | W9 rewards shared languages and first-language overlap; W10 counts cultural-sensitivity overlap; W11, W17–W19 use gender, religious, caste, and diaspora flags. | Exact text matching can miss equivalent concepts; defaults can create scores without user input. |
| Practical fit | W13 fee, W14 preferred format, W15 time slots, W16 experience versus severity. | W12 age fit is always 80; W23 pace is always 80; W26 availability capacity is always 100. |
| Seeded priors | W24 maps session5Retention from 0.65–0.95 to 0–100; W25 maps avgSrsBond from 3.2–4.8 to 0–100. | These fixture values are illustrative, not measured outcomes. |

**Worked example from checked-in records.** Persona [u_1](data/users.ts) lists 6.1, 4.5, 11.4 as primary concerns, English among languages, a video preference, Weekday evenings, and a maximum budget of ₹2,500. Therapist [t_001](data/therapists.ts) shares a language, offers video and that time slot, and charges ₹2,200, so the ordinary logistics gates pass. The therapist specializes in 6.1 and 11.4: W1 = 2/3 × 100 ≈ 66.7, with no first-three bonus because 6.1 is fifth in the therapist list. User directiveness is −4 and therapist directiveness is 7, so W2 = 100 − 11/30 × 100 ≈ 63.3. Their weighted contributions to the 111-weight numerator are approximately 66.7 × 18 = 1,200 and 63.3 × 9 = 570. The remaining 25 dimensions and any anti-match rule determine the actual final rank; these two calculations alone are **not** a displayed match score.

### Explanation generation

The scorer sorts dimension contributions and adds synthetic QACP and TRAUMA entries worth 15 **for explanation ranking only** when their fixture flags are true. These entries do not change the numeric score. [generateWhyText](lib/engine/explainMatch.ts) takes the first three entries and joins available templates. Some dimensions have no template, so an explanation can be short or empty. Some text describes client or clinician experience without evidence in the fixture; read it as illustrative copy, not verified testimony.

### Layer 3 — feedback helper versus live flow

[updateWeightsFromFeedback](lib/engine/matching.ts) is an exported but uncalled helper. It blends orsGain / 6 and srsMean / 40, compares the result to 0.75, and adjusts eligible weights by 5% of that difference within ±20% of defaults. The app's actual [/feedback](app/feedback/page.tsx) form gathers **three 1–5 ratings (total 3–15)**, saves the total through Context, and does not invoke the helper. The denominator of 40 belongs to a different SRS scale than the UI's 15-point form. The success copy saying weights were updated is therefore inaccurate. [/rematch](app/rematch/page.tsx) simply opens the existing ranked alternatives; it does not learn from feedback.

## State, persistence, and privacy boundary

[TherapyProvider](contexts/TherapyContext.tsx) keeps an active profile, moods, feedback totals, ORS-like check-ins, and free-text reflections in React state. It reads and writes activeProfile, moodHistory, feedbackLog, orsHistory, and reflectionHistory in browser **sessionStorage**. Onboarding writes activeProfile and an onboardingComplete marker directly; /matches and /match also read activeProfile directly. The app does not implement an account, database, encrypted health record, server synchronization, or data export/deletion UI. sessionStorage persists across reloads in the same tab session and is accessible to scripts running in that origin; closing the tab normally ends that storage lifetime. Browser behavior and backups should not be treated as a health-data retention policy.

The Context starts with three example SRS totals and three example ORS totals. A real visitor therefore sees pre-populated trends unless storage overrides them. The ORS-like check-in has four 0–10 sliders (total 0–40); its interpretation is fixed text from [lib/clinical.ts](lib/clinical.ts). The clinician dashboard flags the latest three-question rating when its total is below 12 and displays an example repair script. There is no clinician delivery, alert service, access control, consent flow, verified crisis handling, or production privacy policy. Avoid entering genuine personal health information. No genuine personal health records or credentials were identified in the checked-in source during this audit; the seeded people and credential strings must still be treated as **unverified examples**, not authenticated practitioners.

## Setup and troubleshooting

Requirements are a current Node.js/npm environment compatible with the locked [Next.js 16 package](package.json) and a browser with sessionStorage. From the repository root:

~~~bash
npm ci
npm run dev
~~~

Visit http://localhost:3000. For a deterministic source tour, open /matches, choose a seeded persona, then compare the outcome with [matching.ts](lib/engine/matching.ts). Use /onboarding with the demo autofill to see the intake path. npm run build, npm run start, and npm run lint are also declared in [package.json](package.json). **This documentation audit did not install dependencies or run a build, lint, or browser test.**

| Symptom | Source-based explanation / next check |
| --- | --- |
| Results change after reloading or switching tabs | The active profile lives in sessionStorage; inspect the current tab's storage. A fresh tab can use the first seeded persona instead. |
| No matches | One or more non-relaxable gates can remove every therapist; inspect exact language and identity/trauma/severity flags in [passesAllHardFilters](lib/engine/matching.ts). |
| Feedback does not change rankings | The rating page writes only a feedback total; [updateWeightsFromFeedback](lib/engine/matching.ts) is not called. |
| Booking shows an unrelated clinician or no email arrives | [/booked](app/booked/page.tsx) is static demo copy with a random display reference. |
| Admin table contains many zero cells | Each user is scored into a **top-three shortlist** before the 16-column table is filled; absent entries render zero. |
| A route or concept claim disagrees with source | Consult the route component and this blueprint; [SOURCE_OF_TRUTH.md](SOURCE_OF_TRUTH.md) is retained as historical research and may describe intended functionality. |

## Limitations and extension path

The most urgent improvement is **correctness and safety**: align every intake option with the taxonomy, represent identity and preferences without silent defaults, review self-harm handling and helpline details with qualified local experts, and remove unsupported clinical claims from UI copy. Before any real care use, verify clinician identity, license, scope of practice, capacity, and consent; design human review and escalation for high-risk situations. This prototype supplies none of those assurances.

For engineering, isolate matching inputs from UI defaults, make eligibility reasons visible, test gate relaxation and no-match paths, calibrate score meanings, and evaluate potential bias across identities and languages with appropriate governance. If feedback learning is pursued, first reconcile instrument scales, design a consented and privacy-preserving data model, and validate whether adaptation improves any real outcome. Booking, notifications, authentication, role separation, and secure storage would require new services and operational policies; they should be built and evaluated as separate capabilities rather than inferred from demo screens.
