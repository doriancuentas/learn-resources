Comprehensive dietary planning protocol (single-document, app-phase structure, with equations and sources)

Assumptions
- Healthy adults and most teens; exclude pregnancy/lactation, insulin‑dependent diabetes, stage ≥3 CKD, active eating disorder, or clinician‑directed diets (screen in Phase 0).
- Outputs in user’s preferred units (metric or US customary); internal calcs in SI. Conversions: 1 in = 2.54 cm; 1 lb = 0.45359237 kg; 1 kcal = 4.184 kJ.
- Energy equations are estimates with typical ±10–20% error; calibration via 14–21 days of outcomes is required.
- Defaults align to DGA 2020–2025 dietary patterns for general health; athlete specifics from ACSM/AND/DC joint position. ([dietaryguidelines.gov](https://www.dietaryguidelines.gov/resources/2020-2025-dietary-guidelines-online-materials?utm_source=openai))

Phase 0 — Eligibility and safety gates (auto-stop if triggered)
- Stop and refer if: pregnancy/lactation; BMI <18.5 with recent loss; T1D without medical oversight; CKD stage ≥3; cirrhosis/heart failure; uncontrolled HTN; syncope/unexplained GI symptoms; active eating disorder; minors with growth concerns.

Phase 1 — Smart intake (ask once, branch to minimize questions)

1) Minimal required fields (single screen)
- Age; sex at birth; height; weight; country/region; goal [A fat loss, B muscle gain, C recomposition, D maintenance, E performance]; average steps/day; job activity [sedentary/light/moderate/heavy]; training days/week and minutes/session; dietary pattern (omnivore/vegetarian/vegan/pescatarian/other); allergies/intolerances; time zone/email.

2) Conditional follow-ups (only if applicable)
- Body composition known? method/date; if yes, use FFM‑based RMR.
- Sports/performance? sport(s), event date, long-session duration, typical intensity (RPE or HR zones).
- Cultural/religious rules; cuisine likes/dislikes; budget/week; prep minutes/day; cooking skill [low/medium/high].
- Tracking mode preference: [A macros, B protein+calories, C plate method, D none].
- Optional medical/lab context (if provided, reflected in targets): A1C, lipids (apoB/LDL‑C), BP, TSH, ferritin, B12, vitamin D, creatinine/eGFR.

3) Decision tree (branching)
```mermaid
flowchart TD
  A[Start] --> B{Age <14?}
  B -->|Yes| B1[Stop: pediatric clinician]
  B -->|No| C{Pregnant/lactating? T1D? CKD3+? ED hx?}
  C -->|Yes| C1[Stop: clinician-directed plan]
  C -->|No| D{FFM or BF% available?}
  D -->|Yes| D1[Use Cunningham (FFM-based) for RMR]
  D -->|No| D2[Use Mifflin–St Jeor for RMR]
  D1 --> E{Training ≥3 d/wk? Event?}
  D2 --> E
  E -->|Yes| E1[Collect sport specifics]
  E -->|No| E2[Skip]
  E1 --> F{Diet rules/allergies?}
  E2 --> F
  F --> G{Budget/time constraints?}
  G --> H{Tracking mode?}
  H --> I[Finish intake]
```

JSON schema (abridged)
```json
{
  "age": 16,
  "sex_at_birth": "female",
  "units": "metric|us",
  "height": {"value": 165, "unit": "cm"},
  "weight": {"value": 62.0, "unit": "kg"},
  "body_fat_pct": null,
  "ffm_kg": null,
  "goal": "fat_loss|muscle_gain|recomp|maintenance|performance",
  "event_date": null,
  "steps_per_day": 6500,
  "job_activity": "light",
  "training": [{"type":"lifting","days":3,"minutes":60,"intensity":"RPE7"}],
  "diet": {"pattern":"omnivore","allergies":["peanut"],"rules":["halal"]},
  "budget_usd_week": 120,
  "prep_minutes_per_day": 30,
  "tracking_mode": "macros|protein_calories|plate|none",
  "time_zone": "America/Los_Angeles"
}
```

Phase 2 — Build the numbers (equations, targets, and defaults)

2.1 Resting metabolic rate (RMR)
- If fat‑free mass (FFM) known: Cunningham (validated in active populations):
  \[ \mathrm{RMR} = 500 + 22 \times \mathrm{FFM}_{kg} \]
  Variant commonly used: \(\mathrm{RMR} = 370 + 21.6 \times \mathrm{FFM}_{kg}\). ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/1957828/?utm_source=openai))
- Else (adults): Mifflin–St Jeor:
  - Male: \(\mathrm{RMR} = 10W + 6.25H - 5A + 5\)
  - Female: \(\mathrm{RMR} = 10W + 6.25H - 5A - 161\)
  where \(W\) kg, \(H\) cm, \(A\) y. ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/2305711/?utm_source=openai))
- Teens (3–18 y): use IOM EER equations (PA coefficients required). For girls 9–18:
  \[ \mathrm{EER} = 135.3 - 30.8A + \mathrm{PA}\big(10.0W + 934H_m\big) + 25 \]
  For boys 9–18:
  \[ \mathrm{EER} = 88.5 - 61.9A + \mathrm{PA}\big(26.7W + 903H_m\big) + 25 \]
  Use PA per IOM tables. For adults: EER equations with PA also available. ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/11537/chapter/8?utm_source=openai))

2.2 Physical activity and total daily energy expenditure (TDEE)
- Exercise energy per session (MET method):
  \[ \mathrm{EEE}_{kcal} = 0.0175 \times \mathrm{MET} \times W_{kg} \times \mathrm{minutes}\,. \] ([journals.lww.com](https://journals.lww.com/acsm-msse/Fulltext/2011/08000/2011_Compendium_of_Physical_Activities__A_Second.25.aspx?utm_source=openai))
- Physical Activity Level (PAL) categories (factor on RMR/EER): sedentary 1.0–<1.4; low‑active 1.4–<1.6; active 1.6–<1.9; very active 1.9–<2.5. Heuristic mapping to steps (assumes ~2,000 steps/mi and DRI walking distances): <5k ≈1.2–1.3; 5–7.5k ≈1.35–1.45; 7.5–10k ≈1.5–1.6; 10–12.5k ≈1.6–1.75; ≥12.5k ≈1.75–1.9. ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/10490/chapter/7?utm_source=openai))
- Compute:
  \[ \mathrm{TDEE} = \mathrm{RMR} \times \mathrm{PAL} + \overline{\mathrm{EEE}}_{daily} \]
  (Adults without FFM can alternatively use IOM adult EER with PA as primary TDEE; both converge when calibrated.) ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/11537/chapter/8?utm_source=openai))

2.3 Energy target by goal
- Fat loss: 10–25% below TDEE; aim −0.5–1.0% body mass/week (athletes closer to 0.5–0.7% to preserve lean mass). ([jissn.biomedcentral.com](https://jissn.biomedcentral.com/articles/10.1186/1550-2783-11-20?utm_source=openai))
- Muscle gain: +5–15% above TDEE; aim +0.25–0.5%/week.
- Recomposition: near TDEE with high protein and progressive resistance training.

2.4 Macronutrients (daily)
- Protein: 1.6–2.2 g/kg/day; distribute ~0.4 g/kg/meal across ≥4 meals to reach ≥1.6 g/kg/day; upper per‑meal ~0.55 g/kg if targeting ~2.2 g/kg/day. ([pmc.ncbi.nlm.nih.gov](https://pmc.ncbi.nlm.nih.gov/articles/PMC5867436/?utm_source=openai))
- Fat: 0.6–1.0 g/kg/day (≥20% kcal most days).
- Carbohydrate: remainder of kcal after P and F; periodize 3–10+ g/kg/day with higher intakes on high‑volume/intensity days per ACSM/AND/DC. ([journals.lww.com](https://journals.lww.com/acsm-msse/fulltext/2016/03000/nutrition_and_athletic_performance.25.aspx?utm_source=openai))

2.5 Micronutrients and other daily targets
- Fiber ≈14 g per 1,000 kcal (typical 25–38 g/day; titrate to GI comfort). ([download.nap.edu](https://download.nap.edu/skim.php?chap=1-20&record_id=10490&utm_source=openai))
- Sodium <2,300 mg/day (age ≥14), unless otherwise directed. ([dietaryguidelines.gov](https://www.dietaryguidelines.gov/resources/2020-2025-dietary-guidelines-online-materials?utm_source=openai))
- Water (all sources): Adequate Intake ≈3.7 L/day men, 2.7 L/day women; add with heat/exercise. ([download.nap.edu](https://download.nap.edu/skim.php?chap=449-464&record_id=10925&utm_source=openai))
- Pattern: favor vegetables/fruits, whole grains, legumes, nuts, olive oil, fish 1–2×/week; limit added sugars and saturated fat (DGA <10% energy; AHA suggests aiming even lower for high‑risk). ([dietaryguidelines.gov](https://www.dietaryguidelines.gov/resources/2020-2025-dietary-guidelines-online-materials?utm_source=openai))
- Optional ergogenic: creatine monohydrate 3–5 g/day has strong efficacy/safety data for strength/power. ([jissn.biomedcentral.com](https://jissn.biomedcentral.com/articles/10.1186/S12970-017-0173-Z?utm_source=openai))

2.6 Macro-to‑plate translation (fast meal construction)
- P‑block ≈25 g protein; C‑block ≈25 g carbs; F‑block ≈10 g fat; V‑block ≈150 g non‑starchy veg.
- Templates: training meal = 1 P + 1–2 C + 0.5 F + 1–2 V; rest meal = 1 P + 0.5–1 C + 1 F + 2 V.

Phase 3 — Periodized plan with dated “epics” and today’s prescription

3.1 Timeline (relative to t0 = start date)
- Week 0–1 Baseline: collect 7 days of ad‑lib intake, steps, training, morning weights; compute RMR/EER, PAL, TDEE; set initial kcal and macros; stock pantry.
- Weeks 2–3 Calibration: target adherence ≥80%; protein daily. Day‑21 gate: compare 7‑day avg weight vs Day‑1 avg; adjust kcal ±5–10% if off‑target.
- Weeks 4–11 Main Phase I: maintain target rate; add modest carb cycling if training ≥3 d/wk.
- Week 8 Checkpoint α: waist/hip, photos, performance KPIs; apply stall rules if needed.
- Week 12 Consolidation: hold bodyweight within ±0.25% for 7 days; deload if fatigued.
- Weeks 13–20 Main Phase II: recompute TDEE; re‑target; allow 1–2 “social‑flex” meals/wk.
- Week 16 Checkpoint β; Weeks 21–24 Transition to maintenance (step to new TDEE).
- Month 6+: choose next macrocycle (maintain / cut / lean gain).

3.2 Today’s prescription (auto‑generated daily/on‑demand)
- Kcal and macros: display daily kcal; P/C/F grams; fiber and sodium caps.
- Training‑day timing: pre/post‑training carbs 30–60 g; 0.3 g/kg protein within 3 h post‑session. ([journals.lww.com](https://journals.lww.com/acsm-msse/fulltext/2016/03000/nutrition_and_athletic_performance.25.aspx?utm_source=openai))
- Hydration: baseline AI plus +0.5–1.0 L per hour of sweaty exercise; include sodium around long/hot sessions. ([download.nap.edu](https://download.nap.edu/skim.php?chap=449-464&record_id=10925&utm_source=openai))
- SMART activity tasks (example library, scaled to user):
  - Steps: set to baseline +2,000 until 8,000–10,000/day typical; step intensity less critical than total steps (JAMA/NIH). ([jamanetwork.com](https://jamanetwork.com/journals/jama/articlepdf/2763292/jama_saintmaurice_2020_oi_200016.pdf?utm_source=openai))
  - Cardio: accumulate 150–300 min/wk moderate or 75–150 min/wk vigorous. ([odphp.health.gov](https://odphp.health.gov/healthypeople/tools-action/browse-evidence-based-resources/physical-activity-guidelines-americans-2nd-edition?utm_source=openai))
  - Strength: 2–4 sessions/wk; 6–12 reps main lifts; progress load when top‑of‑range achieved with RPE ≤8; last set RPE 7–9. ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/11828249/?utm_source=openai))
- Quantified examples:
  - “Walk 4.0–5.0 km in 40 min (brisk) today” or “Add 2.5 kg to squat if last week’s 3×8 @ RPE ≤8.”

Phase 4 — Tracking and analytics

4.1 Daily logs
- Morning weight (after voiding), steps, kcal, P/C/F, fiber, sodium, sleep hours, hunger 1–10, energy/mood 1–10, session type/duration, adherence flag.

4.2 Weekly and monthly
- Weekly: 7‑day moving average weight; waist/hip; progress photos; top set load×reps.
- Monthly: optional BF% (consistent method), resting BP/HR.

4.3 Derived metrics and formulas
- 7‑ and 14‑day moving averages; loss/gain rate %/wk = 100 × (Δ weight over 14 d / current weight) ÷ 2.
- Kcal–weight slope (simple linear regression) to back‑estimate TDEE and guide adjustments.

Phase 5 — Readjustment rules (evaluate every 14 days unless events warrant earlier)

- Fat loss slow (<0.25%/wk, adherence ≥80%): reduce kcal 5–10% or add +2–3k steps/day; keep protein constant.
- Fat loss too fast (>1.25%/wk for 7–10 d or fatigue high): add +150–300 kcal/day (carb‑first) and/or 1–2 rest days.
- Gain slow (<0.15%/wk): +150–250 kcal/day; gain too fast (>0.6%/wk): −150–300 kcal/day.
- Performance drop ≥2 weeks (sleep adequate): shift +30–60 g carbs pre/post; consider 1‑week maintenance. ([journals.lww.com](https://journals.lww.com/acsm-msse/fulltext/2016/03000/nutrition_and_athletic_performance.25.aspx?utm_source=openai))
- Hunger >7/10 for 4+ days in deficit: +5–10 g fiber; +300–500 g high‑volume veg/day; swap fats→carbs to increase food volume; ensure 25–35 g protein/meal.
- GI distress: reduce sugar alcohols; trial low‑FODMAP swaps 2 weeks; distribute fiber/protein evenly; cap fat 20–30 g/meal.
- Adherence <70% for 2 weeks: simplify to 2–3 repeatable meals; switch to protein+calories tracking; add one social‑flex meal.
- Biomarkers trending adverse (↑BP or ↑LDL‑C/apoB): reduce sat‑fat; add soluble fiber 10 g/d; evaluate sodium/alcohol; consider clinician follow‑up per AHA dietary guidance. ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/34724806/?utm_source=openai))

Phase 6 — Worked examples (illustrative; app will compute precisely)

Example A: adult male, fat‑loss goal
- Inputs: 34 y male; 178 cm; 86 kg; BF 22% (FFM ≈ 67.1 kg); ~7,000 steps/day; 3×/wk lifting, 60 min, MET≈6.
- RMR:
  - Cunningham: RMR = 500 + 22×FFM = 500 + 22×67.1 ≈ 1,976 kcal/d.
  - Mifflin–St Jeor: RMR = 10×86 + 6.25×178 − 5×34 + 5 ≈ 1,808 kcal/d. ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/1957828/?utm_source=openai))
- EEE per session: 0.0175×6×86×60 ≈ 542 kcal; weekly ≈1,626; per‑day avg ≈232 kcal. ([journals.lww.com](https://journals.lww.com/acsm-msse/Fulltext/2011/08000/2011_Compendium_of_Physical_Activities__A_Second.25.aspx?utm_source=openai))
- TDEE (PAL 1.5 heuristic): 
  - Using Cunningham: 1,976×1.5 + 232 ≈ 3,196 kcal/d;
  - Using MSJ: 1,808×1.5 + 232 ≈ 2,933 kcal/d.
- Target energy (−20%): ≈2,560 kcal (Cunningham path) or ≈2,350 kcal (MSJ path). Calibrate by 14–21 d outcome.
- Macros (choose fixed protein/fat; carbs remainder):
  - Protein 2.0 g/kg → 172 g; Fat 0.8 g/kg → 69 g; Carbs ≈260–310 g depending on chosen target kcal.
- Per‑meal across 4 feedings: P ≈ 40–45 g; C ≈ 60–75 g; F ≈ 15–20 g.
- SMART activity: steps baseline+2,000 targeting ≥8,000; 2–3 cardio sessions/week to reach 150–300 min moderate; 3 lifting sessions 6–12 reps main lifts; diet break optional around week 8 if fatigue/hunger high. ([odphp.health.gov](https://odphp.health.gov/healthypeople/tools-action/browse-evidence-based-resources/physical-activity-guidelines-americans-2nd-edition?utm_source=openai))

Example B: active teen (recomp/maintenance bias)
- Inputs: 16 y female; 165 cm; 62 kg; ~6,500 steps; 3×/wk lifting 60 min; goal recomposition/performance; PA (girls active) ≈1.31 per IOM.
- EER (girls 9–18): 135.3 − 30.8×16 + 1.31×(10×62 + 934×1.65) + 25 ≈ 2,499 kcal/d (maintenance baseline). ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/11537/chapter/8?utm_source=openai))
- Macros: Protein 1.8 g/kg → 112 g; Fat 0.8 g/kg → 50 g; Carbs remainder ≈400 g (periodize with training).
- Activity: aim for daily activity (≥60 min moderate‑to‑vigorous) plus 2–4 structured strength sessions/wk. ([odphp.health.gov](https://odphp.health.gov/healthypeople/tools-action/browse-evidence-based-resources/physical-activity-guidelines-americans-2nd-edition?utm_source=openai))
- Note: avoid aggressive deficits/surpluses; prioritize calcium/iron/vitamin D adequacy per DGA life‑stage guidance. ([dietaryguidelines.gov](https://www.dietaryguidelines.gov/resources/2020-2025-dietary-guidelines-online-materials?utm_source=openai))

Phase 7 — Data model and unit handling (for developers)
- Canonical storage in SI; display per user locale. Double precision for height/mass.
- PAL estimation: combine steps/day heuristic with job category; bound to IOM PAL bands; override if user schedules high‑volume sport blocks. ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/10490/chapter/7?utm_source=openai))
- Meal generator: map macro blocks to selected cuisines within sodium/fiber guardrails.

Phase 8 — App views
- Dashboard: today’s macros; hydration; two tasks (movement + meal prep); adherence bar.
- Trends: 7‑day weight average; kcal vs weight scatter with slope; steps histogram with 8,000‑step line.
- Check‑ins: auto‑schedule Day 21, 56, 112; prompt for waist/hip, photos, KPI lifts, rate targets.

Formulas appendix (for professional review)
- Conversions: lb→kg = ×0.45359237; in→cm = ×2.54.
- RMR (adult, no FFM): Mifflin–St Jeor (sex‑specific) as above. ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/2305711/?utm_source=openai))
- RMR (with FFM): Cunningham \(500+22\cdot FFM\) or \(370+21.6\cdot FFM\). ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/1957828/?utm_source=openai))
- EEE: \(0.0175 \cdot MET \cdot W_{kg} \cdot \text{minutes}\). ([journals.lww.com](https://journals.lww.com/acsm-msse/Fulltext/2011/08000/2011_Compendium_of_Physical_Activities__A_Second.25.aspx?utm_source=openai))
- TDEE: \(RMR \cdot PAL + \overline{EEE}_{daily}\). PAL bands per IOM; step mapping heuristic derived from DRI walking distances (2, 7, 17 mi/day). ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/10490/chapter/7?utm_source=openai))
- Protein/day: 1.6–2.2 g/kg; per‑meal 0.4–0.55 g/kg. ([pmc.ncbi.nlm.nih.gov](https://pmc.ncbi.nlm.nih.gov/articles/PMC5867436/?utm_source=openai))
- Carbohydrate/day (athletes): 3–10+ g/kg based on training volume/intensity. ([journals.lww.com](https://journals.lww.com/acsm-msse/fulltext/2016/03000/nutrition_and_athletic_performance.25.aspx?utm_source=openai))
- Fiber: 14 g/1,000 kcal. Water AI: men 3.7 L, women 2.7 L. Sodium: <2,300 mg/day (≥14 y). ([download.nap.edu](https://download.nap.edu/skim.php?chap=1-20&record_id=10490&utm_source=openai))
- Physical activity: 150–300 min/wk moderate or 75–150 min/wk vigorous, plus 2+ strength days. Steps ≥8,000/day associated with lower mortality independent of intensity. ([odphp.health.gov](https://odphp.health.gov/healthypeople/tools-action/browse-evidence-based-resources/physical-activity-guidelines-americans-2nd-edition?utm_source=openai))
- Strength training progression: 6–12 reps for hypertrophy, multi‑joint first, 2–4 days/week; periodize load/volume; RPE‑guided progression acceptable. ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/11828249/?utm_source=openai))

Criticism and limitations
- RMR/EER and PAL introduce model error; therefore the calibration phase is non‑optional.
- Device step counts are biased between brands; use trends.
- High protein/fiber may cause GI issues—titrate.
- Heuristic mapping from steps to PAL is approximate (DRI distances to steps assumption); recalibrate with outcomes. ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/10490/chapter/7?utm_source=openai))

Plan to proceed
- Step 1 Intake: implement the Phase‑1 form + decision tree; store in canonical SI; require only minimal fields; branch for optional detail.
- Step 2 Compute: apply equations above; choose Cunningham if recent reliable FFM present; otherwise Mifflin (adults) or IOM EER (teens). Output kcal and P/C/F, fiber, sodium, hydration, plus per‑meal targets.
- Step 3 Prescribe today: generate meals from macro blocks and user cuisines; add training‑day timing, hydration, and two SMART activity tasks sized to user baseline.
- Step 4 Log: capture daily fields; compute 7/14‑day trends; visualize.
- Step 5 Readjust: run 14‑day rules; auto‑tune kcal or steps; schedule diet breaks if needed; flag biomarker concerns.

Questions
- Q1 Do you want FFM‑based RMR used whenever any FFM is provided, or only when method = DEXA/BIA (≤90 days old)?
- Q2 Default tracking mode: [A macros, B protein+calories, C plate method]?
- Q3 Include creatine prompts by default for resistance‑training users [Y/N]?
- Q4 Preload additional cuisine/rule sets beyond standard tags (e.g., Jain, low‑FODMAP)?
- Q5 Citations format for your GitHub MD: inline links, reference list, or both? (I used inline source citations above; can switch to footnotes on request.)

Primary sources (quick links)
- Mifflin–St Jeor (Am J Clin Nutr, 1990). https://pubmed.ncbi.nlm.nih.gov/2305711/ ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/2305711/?utm_source=openai))
- Cunningham FFM‑based RMR (AJCN, 1991) and athlete validation. https://pubmed.ncbi.nlm.nih.gov/1957828/; https://pubmed.ncbi.nlm.nih.gov/8537566/ ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/1957828/?utm_source=openai))
- Ainsworth Compendium/MET method. https://journals.lww.com/acsm-msse/Fulltext/2011/08000/2011_Compendium_of_Physical_Activities__A_Second.25.aspx ([journals.lww.com](https://journals.lww.com/acsm-msse/Fulltext/2011/08000/2011_Compendium_of_Physical_Activities__A_Second.25.aspx?utm_source=openai))
- IOM DRIs (energy/fiber/water; PAL categories; EER equations). https://nap.nationalacademies.org/read/10490/chapter/7; https://nap.nationalacademies.org/read/10925/chapter/6; https://nap.nationalacademies.org/read/11537/chapter/8 ([nap.nationalacademies.org](https://nap.nationalacademies.org/read/10490/chapter/7?utm_source=openai))
- DGA 2020–2025 (patterns; sodium <2,300 mg). https://www.dietaryguidelines.gov/resources/2020-2025-dietary-guidelines-online-materials ([dietaryguidelines.gov](https://www.dietaryguidelines.gov/resources/2020-2025-dietary-guidelines-online-materials?utm_source=openai))
- ACSM/AND/DC 2016 joint position (athlete carb/protein/timing). https://journals.lww.com/acsm-msse/fulltext/2016/03000/nutrition_and_athletic_performance.25.aspx ([journals.lww.com](https://journals.lww.com/acsm-msse/fulltext/2016/03000/nutrition_and_athletic_performance.25.aspx?utm_source=openai))
- Protein meta‑analysis and per‑meal distribution. https://pmc.ncbi.nlm.nih.gov/articles/PMC5867436/; https://academicworks.cuny.edu/le_pubs/212/ ([pmc.ncbi.nlm.nih.gov](https://pmc.ncbi.nlm.nih.gov/articles/PMC5867436/?utm_source=openai))
- Physical Activity Guidelines (150–300 min/wk; strength 2+ days). https://odphp.health.gov/healthypeople/tools-action/browse-evidence-based-resources/physical-activity-guidelines-americans-2nd-edition ([odphp.health.gov](https://odphp.health.gov/healthypeople/tools-action/browse-evidence-based-resources/physical-activity-guidelines-americans-2nd-edition?utm_source=openai))
- Steps and mortality (JAMA 2020; NIH). https://jamanetwork.com/journals/jama/articlepdf/2763292/jama_saintmaurice_2020_oi_200016.pdf; https://www.nih.gov/news-events/nih-research-matters/number-steps-day-more-important-step-intensity ([jamanetwork.com](https://jamanetwork.com/journals/jama/articlepdf/2763292/jama_saintmaurice_2020_oi_200016.pdf?utm_source=openai))
- Resistance training progression (ACSM position). https://pubmed.ncbi.nlm.nih.gov/11828249/ ([pubmed.ncbi.nlm.nih.gov](https://pubmed.ncbi.nlm.nih.gov/11828249/?utm_source=openai))
- Creatine ISSN position stand. https://jissn.biomedcentral.com/articles/10.1186/S12970-017-0173-Z ([jissn.biomedcentral.com](https://jissn.biomedcentral.com/articles/10.1186/S12970-017-0173-Z?utm_source=openai))

End of document.