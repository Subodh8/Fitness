# 7-Month Fitness Transformation System — Master Blueprint

---

## 1. Project Structure (Claude Pro)

### Recommended Project: "Fitness OS"

Create a single Claude Project with **custom instructions** in the system prompt that persist across all chats. This is your control center.

#### System Prompt (Project Instructions)
Include: your stats (height, weight, age, activity level, goal), your schedule, dietary constraints, injury history, and the decision rules from Section 3 below. Every chat in the project inherits this context automatically — no re-explaining.

#### Chat Architecture (5 Chats)

| # | Chat Name | Purpose | Mode | Why This Mode |
|---|-----------|---------|------|---------------|
| 1 | **Intake & Baseline** | One-time: collect all personal data, set targets, generate initial plan | Chat | Conversational back-and-forth needed to dial in details |
| 2 | **Training Program** | Design mesocycles, exercise selection, progressive overload plan | Chat | Iterative discussion; Claude references your constraints from project instructions |
| 3 | **Nutrition Engine** | Calorie/macro targets, meal frameworks, adjustment protocols | Chat | Needs reasoning about trade-offs (satiety vs. deficit, meal timing vs. schedule) |
| 4 | **Weekly Review Agent** | Paste weekly data dump → get analysis + recommendations | Chat | Structured input/output; this becomes your recurring ritual |
| 5 | **Automation Build** | Build n8n workflows, Supabase schemas, Slack integrations | Chat + Code Execution | Code generation, testing SQL, building JSON for n8n nodes |

#### Why NOT Cowork for This
Cowork is best for file/folder management tasks on your desktop. Your fitness system lives in cloud services (Slack, Supabase, n8n, Notion) — Chat and Code Execution cover everything you need.

#### Master Workflow Connection

```
Daily (you):   Slack message → n8n webhook → Supabase insert
Weekly (auto): n8n cron → pulls 7 days from Supabase → formats summary
Weekly (you):  Paste summary into Chat #4 → Claude analyzes → outputs decisions
Monthly (you): Update project instructions if targets change
```

---

## 2. Automation Architecture

### Stack

| Layer | Tool | Role |
|-------|------|------|
| Input | Slack (`#fitness-log`) | You send weight + optional notes each morning |
| Orchestration | n8n | Webhook listener, cron jobs, data routing |
| Storage | Supabase (PostgreSQL) | Structured table for all metrics |
| Dashboard | Notion | Optional: weekly summary page, habit tracker |
| Calendar | Google Calendar | Workout schedule, meal prep blocks, review reminders |
| Intelligence | Claude (Chat #4) | Weekly analysis and plan adjustments |

### Supabase Schema

```sql
-- Table: daily_logs
CREATE TABLE daily_logs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  logged_at DATE NOT NULL DEFAULT CURRENT_DATE,
  weight_lbs NUMERIC(5,1) NOT NULL,
  sleep_hours NUMERIC(3,1),
  energy_level INT CHECK (energy_level BETWEEN 1 AND 5),
  training_completed BOOLEAN DEFAULT FALSE,
  training_type TEXT, -- 'upper', 'lower', 'cardio', 'rest'
  nutrition_adherence INT CHECK (nutrition_adherence BETWEEN 1 AND 5),
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Table: weekly_summaries (populated by n8n cron)
CREATE TABLE weekly_summaries (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  week_start DATE NOT NULL,
  week_end DATE NOT NULL,
  avg_weight NUMERIC(5,1),
  weight_delta NUMERIC(4,1),
  avg_sleep NUMERIC(3,1),
  avg_energy NUMERIC(3,1),
  training_adherence_pct NUMERIC(5,1),
  nutrition_adherence_avg NUMERIC(3,1),
  trend_direction TEXT, -- 'losing', 'gaining', 'plateau'
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### n8n Workflow #1: Daily Weight Logger

```
Trigger: Slack message in #fitness-log channel
  ↓
Parse message (e.g., "185.2 sleep:7 energy:4 trained:upper adherence:4")
  ↓
Supabase INSERT into daily_logs
  ↓
Slack reply: "✓ Logged: 185.2 lbs | Sleep: 7h | Energy: 4/5"
```

**Slack Message Format (what you type each morning):**
```
185.2 sleep:7 energy:4 trained:upper adherence:4
```
Or minimal version:
```
185.2
```
The n8n workflow should handle both — full data or weight-only, with nulls for missing fields.

### n8n Workflow #2: Weekly Summary Generator (Cron)

```
Trigger: Cron — every Sunday at 10:00 AM NST
  ↓
Supabase query: SELECT * FROM daily_logs WHERE logged_at >= (NOW() - INTERVAL '7 days')
  ↓
Compute: avg weight, weight delta from last week, avg sleep, avg energy,
         training adherence %, nutrition adherence avg
  ↓
INSERT into weekly_summaries
  ↓
Format a clean text summary
  ↓
Post to Slack #fitness-log: "📊 WEEKLY SUMMARY: [formatted data]"
  ↓
(Optional) Update Notion page with weekly row
```

### n8n Workflow #3: Weekly Review Prompt (Optional Advanced)

```
Trigger: Cron — every Sunday at 10:30 AM NST (30 min after summary)
  ↓
Pull the weekly_summary text
  ↓
Call Claude API with a structured prompt:
    "Given this weekly data: [summary]. Given this plan: [current targets].
     Evaluate: Is rate of change on track? Any red flags in sleep/energy?
     Output: Keep plan / Adjust calories by X / Adjust training volume by Y"
  ↓
Post Claude's analysis to Slack #fitness-log
```

This makes the entire feedback loop **hands-free** after initial setup. You wake up, send one Slack message, and the system handles the rest.

### Architecture Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│  You (Slack) │────▶│  n8n webhook │────▶│  Supabase       │
│  #fitness-log│     │  parse msg   │     │  daily_logs     │
└─────────────┘     └─────────────┘     └────────┬────────┘
                                                  │
                    ┌─────────────┐               │ Sunday cron
                    │  n8n cron   │◀──────────────┘
                    │  weekly agg │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Supabase │ │  Slack   │ │ Claude   │
        │ weekly_  │ │ summary  │ │ API call │
        │ summaries│ │ post     │ │ (optional│
        └──────────┘ └──────────┘ │ analysis)│
                                  └──────────┘
```

---

## 3. Fitness System Overview

### 3A. Training Program Structure

**Methodology:** Linear-to-undulating periodization over 7 months, broken into 4-week mesocycles.

**Research basis:** Meta-analyses consistently show that periodized training produces superior strength and hypertrophy outcomes compared to non-periodized approaches (Rhea & Alderman, 2004; Harries et al., 2015). A 4-week mesocycle with a deload every 4th week balances stimulus and recovery.

**Split:** Upper/Lower, 4 days per week
- Fits a cook's variable schedule better than a rigid PPL split
- Each muscle hit 2x/week (optimal frequency per Schoenfeld et al., 2016)

**Mesocycle Progression (7 months = ~7 mesocycles):**

| Block | Weeks | Focus | Rep Range | RIR Target |
|-------|-------|-------|-----------|------------|
| 1 | 1–4 | Foundation / movement quality | 8–12 | 3–4 RIR |
| 2 | 5–8 | Hypertrophy accumulation | 8–15 | 2–3 RIR |
| 3 | 9–12 | Hypertrophy + progressive overload | 6–12 | 1–2 RIR |
| 4 | 13–16 | Strength emphasis | 4–8 | 1–2 RIR |
| 5 | 17–20 | Hypertrophy volume push | 8–15 | 1–2 RIR |
| 6 | 21–24 | Peak intensity | 3–6 | 0–1 RIR |
| 7 | 25–28 | Realization + assessment | Mixed | Autoregulated |

**Volume:** Start at ~10 hard sets per muscle group per week. Increase by 1–2 sets per mesocycle up to ~20 sets if recovery allows (based on Schoenfeld & Krieger, 2017 dose-response meta-analysis).

**Cardio:** 2–3 sessions/week of zone 2 (120–140 BPM range), 20–40 minutes. Walking counts. This supports cardiovascular health, recovery, and caloric expenditure without impairing resistance training (Murach & Bagley, 2016).

### 3B. Nutrition Strategy

**Approach:** Flexible dieting (IIFYM) with protein-first meal architecture.

**Calorie setting:**
- Estimate TDEE using Mifflin-St Jeor equation (most accurate for most populations per Frankenfield et al., 2005)
- If cutting: deficit of 300–500 kcal/day (0.5–1% body weight loss per week to preserve lean mass)
- If lean bulking: surplus of 200–300 kcal/day
- If recomp: eat at maintenance with high protein

**Macronutrient targets:**
| Macro | Target | Rationale |
|-------|--------|-----------|
| Protein | 1.6–2.2 g/kg body weight/day | Upper end of evidence-based range for muscle retention during deficit (Morton et al., 2018) |
| Fat | 0.7–1.0 g/kg body weight/day | Minimum for hormonal health; don't go below 20% of calories (Helms et al., 2014) |
| Carbs | Remainder of calories | Fuel training performance; prioritize around workouts |

**Practical framework:**
- Protein at every meal (4 servings of 30–50g)
- Pre-workout meal 1.5–3 hours before training
- Post-workout meal within 2 hours (the "anabolic window" is wider than bro-science suggests, but proximity still matters for glycogen; Aragon & Schoenfeld, 2013)
- One weekly high-calorie day at maintenance (psychological sustainability, not metabolic magic)

**Adjustment protocol:**
- Track 7-day weight average, not daily fluctuations
- If average weight hasn't moved in the intended direction for 2 consecutive weeks → adjust calories by 100–150 kcal
- Never drop below BMR estimate

### 3C. Recovery Habits

| Habit | Target | Evidence |
|-------|--------|----------|
| Sleep | 7–9 hours/night | Sleep restriction impairs muscle protein synthesis and increases cortisol (Dattilo et al., 2011) |
| Sleep consistency | ±30 min same bed/wake time | Circadian regularity index correlates with better body composition outcomes |
| Hydration | ~1 mL per kcal consumed (minimum) | Even 2% dehydration impairs exercise performance (Cheuvront & Kenefick, 2014) |
| Step count | 7,000–10,000/day | Low-grade NEAT is the largest variable in non-exercise energy expenditure |
| Stress management | Any daily practice: 5–10 min | Chronic stress elevates cortisol, impairs recovery, increases visceral fat storage |
| Deload week | Every 4th week: 50% volume, same intensity | Prevents overreaching, allows connective tissue recovery |

### 3D. Supplement Guidance (Evidence-Supported Only)

Only including supplements with strong, replicated evidence. Everything else is noise.

| Supplement | Dose | Evidence Level | Notes |
|------------|------|----------------|-------|
| Creatine monohydrate | 3–5 g/day | Very strong | Most researched ergogenic aid. Improves strength, power, and lean mass (Kreider et al., 2017). No loading phase needed. |
| Caffeine | 3–6 mg/kg, 30–60 min pre-workout | Strong | Improves endurance and strength performance (Grgic et al., 2020). Skip if it disrupts sleep. |
| Vitamin D | 1000–2000 IU/day (or based on blood test) | Moderate-strong | Especially relevant in St. John's — limited sun exposure Oct–Apr. Deficiency impairs muscle function. |
| Omega-3 (EPA/DHA) | 1–2 g combined EPA+DHA/day | Moderate | Anti-inflammatory; may support recovery. Not a game-changer alone. |
| Whey protein | As needed to hit protein target | Strong (as a food) | Not magic — just convenient protein. Use if whole food is impractical. |

**Skip:** BCAAs (redundant if protein is adequate), fat burners, testosterone boosters, most pre-workout blends with proprietary doses.

### 3E. Progress Tracking Metrics

| Metric | Frequency | Method | Decision Relevance |
|--------|-----------|--------|--------------------|
| Body weight (morning, fasted) | Daily | Slack → Supabase | Primary trend indicator |
| 7-day weight average | Weekly (auto-calculated) | n8n cron | Used for all adjustment decisions |
| Training performance (weight × reps) | Each session | Manual log or app | Progressive overload verification |
| Sleep hours | Daily | Slack log | Recovery quality flag |
| Energy rating (1–5) | Daily | Slack log | Fatigue/overtraining early warning |
| Nutrition adherence (1–5) | Daily | Slack log | Compliance tracking |
| Progress photos | Every 4 weeks | Phone | Visual reference (scale lies sometimes) |
| Body measurements | Every 4 weeks | Tape measure: waist, chest, arms, thighs | Body recomposition evidence |
| Strength benchmarks | End of each mesocycle | Estimated 1RM on 4 key lifts | Objective performance tracking |

### 3F. Decision Rules (When to Change the Plan)

These are the rules the weekly review agent (or you in Chat #4) applies:

**Weight loss goal:**
| Condition | Action |
|-----------|--------|
| Losing 0.5–1% BW/week for 2+ weeks | Stay the course |
| Losing >1.2% BW/week for 2+ weeks | Increase calories by 100–150 kcal (too aggressive = muscle loss risk) |
| Weight stable or gaining for 2+ weeks | Decrease calories by 100–150 kcal OR add 1 cardio session |
| Weight stable but strength increasing + visual change | Likely recomping — hold steady, reassess in 2 more weeks |
| Energy avg <2.5/5 for a full week | Possible under-recovery. Check: sleep? calories too low? training volume too high? |
| Sleep avg <6h for a full week | Prioritize sleep interventions before any other changes |
| Nutrition adherence avg <3/5 for 2+ weeks | Plan is too restrictive. Simplify meals, increase flexibility |
| Training adherence <75% for 2+ weeks | Reduce sessions from 4 to 3, or restructure schedule |
| Completed 4-week block | Run deload, reassess volume/intensity for next block |

**Muscle gain goal:**
| Condition | Action |
|-----------|--------|
| Gaining 0.25–0.5% BW/month | On track for lean gain |
| Gaining >1% BW/month | Too fast — reduce surplus by 100–150 kcal (excess is fat) |
| Weight flat for 3+ weeks | Increase calories by 100–150 kcal |
| Strength stalling across multiple lifts for 2+ weeks | Check recovery first, then consider volume/intensity adjustment |

---

## 4. Tools & AI Capabilities

### What Claude Can Do in This System

| Capability | How It Helps |
|------------|-------------|
| Structured analysis | Feed it weekly numbers → get trend analysis, flag anomalies, suggest specific calorie/volume changes |
| Meal planning | Generate meal frameworks that hit macro targets within food preferences and budget |
| Workout programming | Design full mesocycles with exercise selection, sets, reps, RIR targets, and progression schemes |
| Research lookup | Web search for latest meta-analyses or position stands on specific questions |
| n8n workflow generation | Generate complete n8n workflow JSON for import |
| Supabase SQL | Write and test schema, queries, views, and functions |
| Decision automation | Encode the decision rules above into a structured prompt that Claude API can evaluate weekly |

### Research-Backed Methods to Integrate

| Method | Application | Evidence |
|--------|-------------|----------|
| Rate of perceived exertion (RPE) / RIR tracking | Autoregulate training intensity | Stronger than fixed percentages for intermediate lifters (Helms et al., 2016) |
| Exponential moving average for weight | Smoother trend than simple 7-day average | Reduces noise from water/sodium fluctuations |
| Adaptive TDEE estimation | Recalculate TDEE from actual intake vs. weight change data over 3+ weeks | More accurate than any calculator (uses YOUR data, not population averages) |
| Minimum effective volume (MEV) → maximum recoverable volume (MRV) | Start training at MEV, titrate up toward MRV across a mesocycle | Israetel et al. framework; prevents both undertraining and overtraining |
| Habit stacking | Attach fitness behaviors to existing daily anchors | Clear evidence from behavioral psychology (Wood & Neal, 2007) |

### AI Tools Worth Considering

| Tool | Use Case | Worth It? |
|------|----------|-----------|
| Claude API (via n8n) | Automated weekly analysis posts to Slack | Yes — this is the centerpiece |
| MacroFactor app | Adaptive TDEE tracking, food logging | Yes — best consumer app for evidence-based nutrition tracking; worth the $6/mo |
| Strong app (or Hevy) | Workout logging with progressive overload charts | Yes — structured data for training performance |
| Notion (already connected) | Central dashboard: weekly reviews, mesocycle plans, habit tracker | Yes — you already have it |
| Whoop/Apple Watch/Garmin | HRV, sleep quality, recovery score | Optional — useful but not essential if sleep hours + energy ratings are logged |

---

## 5. Step-by-Step Setup Plan

### Week 0: Foundation (Before Any Training)

**Day 1–2: Data Collection**
1. Collect all personal metrics (see Section 7 below)
2. Calculate baseline TDEE using Mifflin-St Jeor
3. Set initial calorie and macro targets
4. Define primary goal (cut / bulk / recomp) and 7-month target

**Day 3–4: Infrastructure**
5. Create Supabase tables (use schema from Section 2)
6. Create Slack channel `#fitness-log`
7. Build n8n Workflow #1 (daily weight logger)
8. Build n8n Workflow #2 (weekly summary cron)
9. Test end-to-end: send Slack message → verify Supabase row → verify Sunday summary

**Day 5–6: Claude Project Setup**
10. Create Claude Project "Fitness OS"
11. Write project instructions with all your stats, goals, decision rules
12. Open Chat #1 (Intake) — confirm all numbers, finalize plan
13. Open Chat #2 (Training) — generate Mesocycle 1 program
14. Open Chat #3 (Nutrition) — generate meal framework + grocery list

**Day 7: Soft Launch**
15. Do first workout from the program
16. Send first Slack log message
17. Set Google Calendar recurring events: workouts, meal prep, weekly review
18. Optional: build n8n Workflow #3 (Claude API weekly review)

### Weeks 1–4: Mesocycle 1

- Follow program
- Log daily in Slack (takes <30 seconds)
- Sunday: review weekly summary in Chat #4
- End of Week 4: deload + progress photos + measurements

### Ongoing (Months 2–7)

- Each 4-week block: new mesocycle parameters from Chat #2
- Each week: automated summary → Chat #4 analysis
- Each month: update project instructions if targets shift
- Month 4 checkpoint: major reassessment (halfway) — are we on track for the 7-month goal?

---

## 6. To-Do List for You

### Before We Start Building

- [ ] Decide primary goal: fat loss, muscle gain, or body recomposition
- [ ] Set a specific 7-month target (e.g., "185 → 165 lbs" or "add visible muscle while staying under 180")
- [ ] Weigh yourself tomorrow morning (fasted, after bathroom) and report
- [ ] Provide all data from Section 7 below
- [ ] Decide: are you willing to use a food tracking app (MacroFactor recommended), or do you want a simpler meal framework approach?
- [ ] Confirm your weekly schedule: which days/times can you train? (account for cook shifts)
- [ ] Confirm gym access: home gym, commercial gym, equipment available?

### Infrastructure Setup

- [ ] Create `#fitness-log` Slack channel
- [ ] Create Supabase table using provided schema (I can generate the SQL)
- [ ] Build n8n Workflow #1 (I will provide the full workflow JSON)
- [ ] Build n8n Workflow #2 (I will provide the full workflow JSON)
- [ ] Test the pipeline end-to-end
- [ ] Create Claude Project "Fitness OS" with system instructions
- [ ] Set Google Calendar events for workouts and weekly review

### Ongoing Weekly

- [ ] Log weight + data in Slack every morning (~30 seconds)
- [ ] Follow training program
- [ ] Hit macro targets (or at minimum, protein target)
- [ ] Sunday: review weekly summary, paste into Chat #4 if adjustments needed
- [ ] Every 4 weeks: progress photos + body measurements

---

## 7. Data I Need From You

### Required (to build your plan)

| Data Point | Why I Need It |
|------------|---------------|
| Age | TDEE calculation, recovery expectations |
| Height | TDEE calculation, body composition context |
| Current weight (fasted morning) | Baseline for all tracking |
| Primary goal (cut / bulk / recomp) | Determines calorie direction and training emphasis |
| Target weight or physique description | Defines the 7-month endpoint |
| Training experience (years, consistency) | Determines starting volume, exercise complexity |
| Current training (if any) | Know what we're building from |
| Injuries or movement limitations | Exercise selection constraints |
| Work schedule (typical shifts) | Training day/time planning |
| Gym access + equipment available | Exercise programming |
| Food preferences / restrictions / allergies | Meal framework design |
| Cooking skill level | You're a cook — but confirming for meal planning |
| Whether you'll use a food tracking app | Determines nutrition tracking approach |
| Daily step count estimate | NEAT baseline for TDEE |
| Current sleep pattern (hours, consistency) | Recovery baseline |
| Stress level (1–10) | Recovery and cortisol context |
| Any medications | Some affect metabolism, water retention, or recovery |
| Budget for supplements (if any) | Determines supplement recommendations |

### Nice to Have

| Data Point | Why It Helps |
|------------|-------------|
| Recent blood work (vitamin D, testosterone, thyroid) | Rule out deficiencies or hormonal issues |
| Previous dieting history | Metabolic adaptation context |
| Body measurements (waist, chest, arms, thighs) | Baseline for recomp tracking |
| Current estimated daily calorie intake | Calibrates starting point |
| Resting heart rate | Cardiovascular fitness baseline |

---

## Appendix: How Each Chat Runs

### Chat #1 — Intake & Baseline (One-time)
**Input:** All data from Section 7
**Output:** Confirmed TDEE, calorie target, macro split, 7-month milestone roadmap

### Chat #2 — Training Program (Every 4 weeks)
**Input:** "Generate Mesocycle [X]. Here's how last block went: [performance notes]"
**Output:** Full 4-week program with exercises, sets, reps, RIR, progression rules

### Chat #3 — Nutrition Engine (As needed)
**Input:** "I need a new meal framework" or "I'm struggling with [specific issue]"
**Output:** Updated meal templates, grocery lists, or troubleshooting adjustments

### Chat #4 — Weekly Review (Every Sunday)
**Input:** Paste weekly summary from Slack/n8n
**Output:** Trend analysis, adherence evaluation, specific adjustment recommendations based on decision rules

### Chat #5 — Automation Build (Setup phase + maintenance)
**Input:** "Build n8n workflow for [X]" or "Fix this Supabase query"
**Output:** Working code, JSON exports, SQL queries, debugging

---

*This blueprint is version 1.0. It will evolve as we collect your baseline data and begin execution.*
