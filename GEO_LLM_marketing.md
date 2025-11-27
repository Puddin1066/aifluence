# Generative Engine Optimization & LLM Influencer Marketing Playbook

This document distills how the current repo enables fully-automated influencer personas that can be tuned for generative-engine optimization (GEO) and LLM-driven marketing while still giving operators precise control over sponsors, demographics, and campaign arcs.

## Platform Snapshot

- **Backend**: FastAPI + SQLite (`backend/`). Handles onboarding (`/sorcerer/init`), scheduling (`/schedule`, `/schedule/interval`, `/schedule/bulk`), sponsor CRUD/matching, Gemini image generation, and divine interventions for lifestyle personas.
- **AI Core**: `managers/ai_generator.py` uses Anthropic for life stories, reel/story planning, scene prompts, and captions. All functions gracefully fall back to deterministic text when API keys are missing, making mocks explicit.
- **Schedulers**: APScheduler (`managers/scheduler.py`) executes `Schedule` jobs, moving videos from pending → processing → posted. Background workers (`utils/background_tasks.py`) generate prompts/captions before inserting `Video` rows.
- **Frontend**: Next.js App Router (`frontend/`). Provides the onboarding wizard, influencer dashboards, sponsor directory, and internal API helpers (OpenAI audience suggestions, prompt stubs).

## Generative Engine Optimization (GEO) Workflow

1. **Persona Fabrication**
   - Wizard collects tone, goals, target demographics, IG credentials.
   - Lifestyle mode automatically calls `generate_life_story`; company mode stores frequency knobs for interval scheduling.
   - Audience targeting lives in `Influencer.audience_targeting`, enabling precise alignment with GEO keyword clusters.

2. **Narrative Planning**
   - Lifestyle personas run `plan_and_schedule_from_life_story`, which:
     - Generates reel tentpoles (`generate_reel_content_plan`).
     - Generates complementary story beats (`generate_story_content_plan`).
     - Produces scene prompts + captions aware of persona goals and audience interests.
   - Company personas run `process_interval_schedule`, creating reels/stories at fixed cadences but still enriched via AI prompts.

3. **Scheduling & Triggering**
   - Each prompt becomes a `Video` + `Schedule`. APScheduler stores job IDs so divine interventions can cancel and regenerate posts safely.
   - Manual overrides use `AddPostModal` → `POST /schedule`, letting operators inject high-priority GEO angles or sponsor campaigns on demand.

4. **Execution**
   - At `run_at`, `VideoScheduler.process_scheduled_video` marks the post as processing/posted (placeholder for real render/upload). The Instagram manager already wraps instagrapi, so swapping in an actual renderer is straightforward.

## Multi-Persona Strategy for LLM Influence Campaigns

| Capability | Implementation Detail |
| --- | --- |
| Distinct personas per vertical | Run `/sorcerer/init` multiple times. Each influencer row carries unique persona JSON, demographics, and scheduling mode. |
| Shared sponsors / cross-campaign arcs | Sponsors live globally. Use `POST /sponsor/match` to document relationships, and attach `sponsor_id` on each scheduled video. |
| Inter-persona storytelling | Divine intervention + manual scheduling let you reference other personas in prompts/captions, enabling collaborative narratives around a single sponsor reveal. |
| Audience tuning | `audience_targeting` flows into AI prompts, ensuring each persona speaks to its vertical-specific interests/regions while promoting the same sponsor. |
| Rapid replans | Lifestyle workflows (process_lifestyle_post_update + divine intervention) delete future posts and regenerate 30-day calendars in minutes, so coordinated campaigns stay in sync. |

## Operating Several Personas in Parallel

1. **Provision** each persona via the wizard (or direct API call) with distinct tone/goals/demographics.
2. **Assign sponsors** by calling `/sponsor/match` for each persona → sponsor pair.
3. **Schedule campaign beats**:
   - Use interval scheduling for evergreen content.
   - Inject synchronized hero moments via manual `/schedule` calls that reference the common sponsor story.
4. **Trigger cross-talk** by:
   - Embedding other personas’ events inside `generation_prompt.description` for manual posts.
   - Running divine intervention on multiple personas simultaneously to regenerate future posts that acknowledge the shared sponsor milestone.
5. **Monitor & iterate** by inspecting `/influencer/{id}/videos` in the dashboard; cancel/rewrite via divine intervention if timelines drift.

## Sponsor Lifecycle

1. `POST /sponsors` → store metadata (tier, industry, targeting tags).
2. `POST /sponsor/match` → compute a simple score (shared interests) and persist in `SponsorMatch`.
3. `AddPostModal` → attach sponsor during scheduling.
4. `POST /video/{video_id}/add-sponsor` → retrofit sponsorship on previously scheduled content.
5. Extend background planners to pass sponsor context into `generate_scene_prompt` for automated product mentions.

## Cost Model (Automation-Only)

- **Anthropic API**: ~$120/mo per persona for life story + ~40 prompts/captions.
- **Gemini image generation**: ~$60/mo if generating 4–6 avatars/frames.
- **OpenAI helper route**: <$5/mo during onboarding tweaks.
- **Infra**: ~$40/mo for a small VPS running FastAPI + Next.js + APScheduler, plus ~$10 for media storage.
- **Contingency**: budget $50–$80 for reruns/spikes/logging.
- **Total**: ≈ **$350–$500 per persona per month** when your human time is free.

Costs shrink if you reuse life stories, disable Gemini, or minimize divine interventions; they rise with heavier video rendering or premium AI tiers.

## Runbook (Full Operation)

```bash
# Backend
cd backend
pip install -r requirements.txt
cp .env.example .env  # add ANTHROPIC_API_KEY, GEMINI_API_KEY
uvicorn app:app --reload

# Frontend
cd frontend
npm install
npm run dev  # or bun/pnpm equivalents
```

Keep both servers online; FastAPI exposes `http://localhost:8000`, Next.js runs on `http://localhost:3000`.

## GEO & LLM Best Practices

- **Seed rich personas**: The more detailed the goals/background, the better the AI planner can craft search-friendly narratives.
- **Use sponsor tags in prompts**: Pass `{"sponsor": {...}}` into `generate_scene_prompt` (simple code change) so captions naturally mention the brand.
- **Cadence variation**: Mix interval scheduling with lifestyle planning to avoid uniform posting that algorithms might demote.
- **Inter-persona amplification**: Schedule “collab” posts where personas reference each other to compound reach around sponsor launches.
- **Audit AI outputs**: Even with automation, add lightweight review hooks (e.g., pre-flight caption check) before high-value sponsor slots.

Adopting this playbook lets you treat personas as programmable agents tuned for generative engines while coordinating them like a virtual creator collective that advances sponsor goals in lockstep.
