# Adult Asthma Decision Support Application Design

Last updated: April 9, 2026
Primary guideline basis: GINA 2025
U.S. preventive care defaults: CDC vaccine guidance current as of February 2026

## 1. Product Goal

Build a clinician-facing application for adults with suspected or established asthma that:

- confirms or challenges the asthma diagnosis using objective testing before long-term treatment recommendations are made
- recommends guideline-concordant initial and follow-up treatment, favoring GINA Track 1
- gives medication-specific output with dose, route, frequency, key treatment considerations, common adverse effects, and follow-up actions
- surfaces non-pharmacologic actions, including vaccinations
- makes severe asthma pathways unusually explicit so clinicians do not get stuck in repeated steroid escalation without phenotype assessment or referral

## 2. Safety Boundaries

This should be a clinical decision support tool, not an autonomous prescriber.

- The application should require clinician attestation before finalizing a recommendation.
- The application should show a visible banner: `Decision support only. Confirm against current local formulary, contraindications, pregnancy status, and specialist input when indicated.`
- Long-term asthma treatment should not be labeled as "confirmed asthma" unless objective evidence of variable expiratory airflow is present, except in urgent situations where empiric treatment is started and repeat testing is scheduled.
- The application should have an `urgent triage` interrupt before all other logic. If the patient has red-flag features such as severe respiratory distress, cyanosis, inability to speak full sentences, altered mental status, silent chest, or SpO2 instability, the app should stop routine logic and direct immediate emergency care.
- The app should maintain versioned source metadata so every recommendation displays the guideline version used.

## 3. Recommended Product Structure

### Core modules

1. `Diagnostic confirmation`
2. `Initial treatment`
3. `Follow-up / step-up / step-down`
4. `Severe asthma pathway`
5. `Prevention and vaccination`
6. `Recommendation explainer`
7. `Guideline source registry`

### Suggested architecture

- `Mirror the COPD tool implementation pattern`: build this as a static browser application with `index.html`, `styles.css`, and a single rules-driven `app.js`, plus a copy/paste-ready note output. Keep the interaction model analogous to the existing COPD decision support application.
- `Rules engine first`: use deterministic clinical rules for diagnosis, step assignment, severe-asthma branching, and vaccine prompts.
- `Narrative layer second`: if an LLM is used, limit it to rewriting the already computed recommendation into clinician-friendly text. It should not decide the treatment step.
- `Medication library`: store dose, route, max daily dose, common adverse effects, monitoring points, and formulary mappings separately from logic.
- `Source registry`: every rule should point to a source ID, publication date, and last review date.
- `Config layer`: allow local override for formulary availability, payer restrictions, biologic access, and local COVID/flu protocol.

## 4. Clinical Scope

### In scope

- Adults age 18 and older
- Suspected asthma
- Established asthma follow-up
- Outpatient and ambulatory decision support
- Severe and difficult-to-treat asthma escalation
- Non-pharmacologic counseling and vaccines

### Out of scope for v1

- Detailed inpatient acute exacerbation treatment
- Pediatric asthma
- Pregnancy-specific asthma management beyond warning flags
- Full occupational asthma workflow
- Full biologic prior-authorization automation

## 5. Required Inputs

The app should not allow a treatment recommendation until required inputs for that branch are present.

### Demographics

- age
- sex
- pregnancy status
- smoking or vaping status
- body mass index

### Diagnostic inputs

- typical variable symptoms present: wheeze, dyspnea, chest tightness, cough
- nighttime symptoms
- exercise- or allergen-triggered symptoms
- viral-triggered worsening
- spirometry values: pre-BD FEV1, post-BD FEV1, pre-BD FVC, post-BD FVC, pre/post FEV1/FVC, predicted values, LLN
- bronchodilator used and withholding interval
- peak flow diary data if spirometry unavailable
- bronchoprovocation type and result
- 4-week ICS response if used diagnostically
- FeNO and blood eosinophils when available
- differential diagnosis flags: COPD risk, inducible laryngeal obstruction, bronchiectasis, heart failure, GERD, chronic rhinosinusitis, OSA, medication-related cough

### Asthma control and risk inputs

- daytime symptoms more than twice per week in last 4 weeks
- night waking due to asthma
- reliever use
- activity limitation
- exacerbations in last 12 months
- ED visits or hospitalization
- ICU or intubation history
- baseline and current FEV1
- SABA overuse
- adherence estimate
- inhaler technique check
- environmental triggers
- medication adverse effects

### Treatment history

- current controller and reliever
- prior controller failures
- recent oral steroid courses
- maintenance oral steroid use
- LAMA use
- montelukast or LTRA use
- prior azithromycin trial
- biologic history

### Severe asthma phenotype inputs

- blood eosinophils
- FeNO
- total IgE
- allergen sensitization testing
- nasal polyps / CRSwNP
- atopic dermatitis
- aspirin-exacerbated respiratory disease
- OCS dependence
- payer or formulary access to biologics

### Vaccine inputs

- influenza vaccine status
- COVID vaccine status per current local schedule
- pneumococcal vaccine history: PCV15, PCV20, PCV21, PPSV23, PCV13 if relevant
- RSV vaccine history
- zoster vaccine history
- Tdap or Td history and last date

## 6. Diagnostic Logic

### Diagnostic principle

Do not let the app jump directly to chronic asthma treatment unless asthma is objectively confirmed or there is a documented urgent reason to start empiric therapy before confirmation.

### Criteria that support the diagnosis of asthma in adults

The app should treat asthma as objectively supported when there are typical symptoms plus at least one of the following:

- Positive bronchodilator response on spirometry:
  - increase in FEV1 or FVC of `>=12% and >=200 mL` from baseline
  - or increase in FEV1 or FVC of `>10%` relative to the predicted value
  - mark `higher confidence` if increase is `>=15% and >=400 mL`
- Peak flow variability over 2 weeks:
  - average daily diurnal PEF variability `>10%`
- Improvement after 4 weeks of daily ICS-containing treatment:
  - FEV1 increase `>=12% and >=200 mL`
  - or PEF increase `>=20%`
- Positive bronchoprovocation:
  - methacholine: FEV1 fall `>=20%`
  - standardized hyperventilation, hypertonic saline, or mannitol: FEV1 fall `>=15%`
  - standardized exercise challenge: FEV1 fall `>10% and >200 mL`

### Required spirometry safeguards

- Show an alert if bronchodilator withholding periods were not respected.
- If FEV1 falls during a challenge test, require confirmation that FEV1/FVC also fell so poor effort or inducible laryngeal obstruction is not misclassified as asthma.
- If spirometry is unavailable, permit PEF-based confirmation but label confidence as lower.

### If the patient is already on ICS-containing therapy and diagnosis is uncertain

If symptoms are typical but objective variability is absent:

- repeat spirometry after correct bronchodilator withholding or during symptoms
- if FEV1 or PEF is `>70% predicted`, consider supervised step-down of ICS-containing therapy and reassessment in 2-4 weeks, then bronchoprovocation or repeat bronchodilator testing
- if FEV1 or PEF is `<70% predicted`, allow a 3-month ICS-containing treatment optimization period, then reassess; if no response, resume prior dose and refer

### Diagnostic outputs

The diagnosis module should return one of these states:

- `asthma objectively confirmed`
- `asthma likely but not yet confirmed`
- `diagnosis uncertain - repeat objective testing required`
- `alternative diagnosis or overlap should be prioritized`

If not confirmed, the app should recommend next tests instead of long-term step therapy.

## 7. Initial Treatment Logic for Adults

### Default strategy

Use `GINA Track 1` as the preferred pathway.

- Preferred reliever: low-dose ICS-formoterol
- Preferred MART medication when MART is chosen: `budesonide-formoterol`

### Step 1-2: AIR-only

Use `as-needed low-dose budesonide-formoterol` for adults with newly diagnosed or mild asthma when symptoms are infrequent and lung function is normal or mildly reduced.

Recommended adult dosing:

- Medication: `budesonide-formoterol`
- Route: inhaled DPI or pMDI
- Strength: `200/6 mcg metered dose` (`160/4.5 mcg delivered dose`)
- Dose: `1 inhalation as needed`
- May also be used before exercise or expected allergen exposure
- Max total: `12 inhalations in any 24-hour period`

### Start directly at Step 3 MART instead of Step 1-2 if any of these are present

- symptoms most days
- night waking due to asthma more than once a week
- current smoking
- impaired perception of bronchoconstriction, such as low initial lung function with few symptoms
- recent severe exacerbation
- history of life-threatening exacerbation
- severe airway hyperresponsiveness
- active seasonal allergen exposure driving symptoms

### Step 3: Low-dose MART

- Medication: `budesonide-formoterol`
- Route: inhaled DPI or pMDI
- Strength: `200/6 mcg metered` (`160/4.5 mcg delivered`)
- Maintenance dose: `1 inhalation twice daily` or `1 inhalation once daily`
- Reliever dose: `1 inhalation as needed`
- Max total: `12 inhalations in any 24-hour period`

### Step 4: Medium-dose MART

- Medication: `budesonide-formoterol`
- Route: inhaled DPI or pMDI
- Strength: `200/6 mcg metered` (`160/4.5 mcg delivered`)
- Maintenance dose: `2 inhalations twice daily`
- Reliever dose: `1 inhalation as needed`
- Max total: `12 inhalations in any 24-hour period`

### Step 5

If symptoms or exacerbations persist despite correct technique and good adherence on Step 4:

- do not simply keep escalating inhaled therapy without reassessment
- refer promptly for expert assessment, severe-asthma phenotyping, and add-on therapy consideration
- if the patient is already doing well on MART and needs biologic therapy, do not automatically switch from MART to conventional ICS-LABA plus SABA

## 8. Follow-Up Logic

### Use GINA symptom control questions at each visit

In the last 4 weeks, ask:

- daytime symptoms more than twice per week?
- any night waking due to asthma?
- reliever use more than twice per week? 
  - for SABA users, yes counts toward poor control
  - for ICS-formoterol Track 1 users, frequency should still be reviewed but should not be interpreted exactly like SABA overuse
- any activity limitation?

Classify:

- `well controlled`: 0 positive items
- `partly controlled`: 1-2 positive items
- `uncontrolled`: 3-4 positive items

### Before every step-up

The app should force a checklist:

- confirm symptoms are due to asthma
- inspect inhaler technique
- assess adherence
- assess smoking and exposures
- check SABA overuse
- review obesity, GERD, chronic rhinosinusitis, OSA, inducible laryngeal obstruction, anxiety/depression
- review medication adverse effects

### Risk factors that should intensify concern even if symptoms seem mild

- SABA overuse
- inadequate ICS exposure
- low FEV1, especially `<60% predicted`
- raised eosinophils or FeNO
- current smoking
- allergen exposure if sensitized
- severe exacerbation in the last year
- ICU or intubation history

## 9. Severe Asthma Pathway

### Definitions the app should use

- `Difficult-to-treat asthma`: uncontrolled despite medium- or high-dose ICS plus a second controller, or requires maintenance OCS.
- `Severe asthma`: asthma that remains uncontrolled despite optimized treatment and contributory factors being addressed, or worsens when high-intensity treatment is stepped down.

### Required severe-asthma workflow

#### Stage 1: confirm it is really asthma

- re-check objective diagnosis
- evaluate overlap and mimics

#### Stage 2: look for reasons the patient appears uncontrolled

- poor inhaler technique
- poor adherence
- obesity
- GERD
- chronic rhinosinusitis
- OSA
- inducible laryngeal obstruction
- ongoing smoking or irritant exposure
- relevant allergen exposure
- beta-blockers or NSAIDs
- medication adverse effects
- anxiety, depression, or social barriers

#### Stage 3: optimize before labeling as severe

- asthma education
- switch to MART if available
- non-pharmacologic intervention
- treat comorbidities
- consider non-biologic add-ons such as LAMA or LTRA if not already tried
- trial high-dose ICS-LABA for `3-6 months` if not currently used

#### Stage 4: review response after roughly 3-6 months

If still uncontrolled, or if asthma becomes uncontrolled when stepped down, classify as `severe asthma likely` and trigger specialist workflow.

### Type 2-high severe asthma markers

If any of the following are present on high-dose ICS or lowest possible OCS dose, flag probable Type 2-high disease:

- blood eosinophils `>=150/uL`
- FeNO `>=20 ppb`
- sputum eosinophils `>=2%`
- clinically allergen-driven asthma

If eosinophils or FeNO are not elevated, the app should allow repeating them up to 3 times when clinically appropriate.

### Severe asthma phenotype prompts

The app should explicitly check for:

- severe allergic asthma
- severe eosinophilic asthma
- maintenance OCS-dependent asthma
- CRSwNP
- atopic dermatitis
- aspirin-exacerbated respiratory disease
- bronchiectasis
- ABPA
- infection risk or parasitic infection when eosinophils are high

## 10. Severe Asthma Add-On Therapy Library

These should be shown only after severe-asthma gating and specialist referral prompts.

### Non-biologic add-ons

#### LAMA

- Medication examples: `tiotropium` or triple inhaler options
- Route: inhaled
- Use: add-on at Step 5, or Step 4 with weaker evidence
- Considerations: ensure at least medium-dose ICS-LABA is in place before adding
- Common adverse effects: dry mouth, urinary retention

#### Azithromycin

- Route: oral
- Dose: `500 mg three times weekly`
- Duration to judge benefit: `at least 6 months`
- Use: only after specialist referral in adults with persistent symptoms despite high-dose ICS-LABA
- Preconditions: check sputum for atypical mycobacteria, check ECG for QTc before treatment and again after 1 month, weigh antimicrobial resistance risk
- Common adverse effects: diarrhea, QT-related concerns

#### Maintenance oral corticosteroid

- Route: oral
- Use: last resort only
- Suggested ceiling in Step 5 narrative: `<=7.5 mg/day prednisone equivalent`
- Common adverse effects:
  - short course: sleep disturbance, reflux, appetite increase, hyperglycemia, mood changes, infection and thromboembolism risk
  - maintenance: cataract, glaucoma, hypertension, diabetes, adrenal suppression, osteoporosis

### Type 2-targeted biologics

#### Omalizumab

- Route: subcutaneous
- Dose: every `2-4 weeks`, based on weight and serum IgE
- Use: severe allergic asthma with sensitization, dosing range eligibility, and recent exacerbations
- Common adverse effects: injection-site reactions, rare anaphylaxis

#### Mepolizumab

- Route: subcutaneous
- Dose: `100 mg every 4 weeks`
- Use: severe eosinophilic asthma
- Common adverse effects: headache, injection-site reactions

#### Benralizumab

- Route: subcutaneous
- Dose: `30 mg every 4 weeks for 3 doses, then every 8 weeks`
- Use: severe eosinophilic asthma
- Common adverse effects: injection-site reactions, rare anaphylaxis

#### Reslizumab

- Route: intravenous
- Dose: `3 mg/kg every 4 weeks`
- Use: severe eosinophilic asthma
- Common adverse effects: infusion or hypersensitivity reactions

#### Dupilumab

- Route: subcutaneous
- Dose:
  - `200 mg or 300 mg every 2 weeks` for severe eosinophilic/Type 2 asthma
  - `300 mg every 2 weeks` for OCS-dependent severe asthma
- Use: severe eosinophilic/Type 2 asthma or maintenance OCS-dependent disease
- Common adverse effects: injection-site reactions, transient eosinophilia
- Important warning: rare EGPA may be unmasked when OCS is reduced; avoid casual use when baseline eosinophils are `>1500/uL`

#### Tezepelumab

- Route: subcutaneous
- Dose: `210 mg every 4 weeks`
- Use: severe asthma with recent exacerbations, including some patients without elevated Type 2 markers
- Common adverse effects: injection-site reactions, rare anaphylaxis

### Biologic decision prompts

The app should ask:

- Is the patient eligible for anti-IgE?
- Is the patient eligible for anti-IL5 or anti-IL5R?
- Is the patient eligible for anti-IL4Ralpha?
- Is the patient eligible for anti-TSLP?

It should then display:

- matching phenotype
- required biomarkers
- route
- dosing frequency
- payer or formulary gating
- expected markers of response
- `trial at least 4 months`

## 11. Medication Recommendation Format

Every medication recommendation card should show:

- `recommended medication`
- `strength`
- `route`
- `maintenance dose`
- `reliever dose if applicable`
- `maximum daily dose`
- `why this is recommended`
- `common adverse effects`
- `important cautions`
- `when to review`
- `guideline source`

### Example recommendation payload

```json
{
  "status": "asthma objectively confirmed",
  "track": "GINA Track 1",
  "step": "3",
  "recommendation_type": "controller_and_reliever",
  "medication": "budesonide-formoterol",
  "strength": "200/6 mcg metered dose (160/4.5 mcg delivered)",
  "route": "inhaled DPI or pMDI",
  "maintenance_dose": "1 inhalation twice daily",
  "reliever_dose": "1 inhalation as needed",
  "max_daily_dose": "12 inhalations in 24 hours",
  "rationale": "Symptoms most days and nocturnal awakening more than once weekly favor starting at Step 3 MART.",
  "common_adverse_effects": [
    "oropharyngeal candidiasis",
    "dysphonia",
    "tachycardia",
    "headache",
    "cramps"
  ],
  "cautions": [
    "Do not use LABA without ICS in asthma.",
    "Review inhaler technique and adherence before future step-up."
  ],
  "follow_up": "Reassess symptom control, exacerbations, inhaler technique, adherence, and lung function in 4-12 weeks.",
  "source": "GINA 2025 Box 4-8"
}
```

## 12. Non-Pharmacologic Recommendation Engine

Every output should include a prevention block with:

- inhaler technique review
- written asthma action plan
- smoking and vaping cessation counseling
- exercise encouragement
- weight management when appropriate
- trigger mitigation
- respiratory virus exposure mitigation when appropriate
- comorbidity treatment reminders

## 13. Vaccine Logic for Adults with Asthma

Use CDC defaults for the United States, but make these rules date-aware and configurable because vaccine recommendations change.

### Influenza

- Recommend `annual flu vaccination`
- Display as routine for all adults

### COVID-19

- Do not hardcode a fixed dose count in the asthma rule set
- Instead, reference the active local protocol or current CDC schedule for the current season
- The app should show: `COVID-19 vaccination should follow current local/CDC guidance, which is time-sensitive and must remain configurable`

### Pneumococcal

- Adults `>=50 years`: recommend pneumococcal vaccination if no prior PCV or series not complete
- Adults `19-49 years` with asthma: also recommend pneumococcal vaccination because asthma counts as chronic lung disease
- If vaccine-naive, show CDC default options:
  - `PCV20 or PCV21 once`
  - or `PCV15 followed by PPSV23`
- Because prior vaccine history can be complex, the app should defer final series logic to a vaccine sub-engine rather than a simple age rule

### RSV

- As of February 20, 2026, CDC recommends RSV vaccine for:
  - all adults `>=75 years`
  - adults `50-74 years` at increased risk of severe RSV illness
- Asthma qualifies under chronic lung disease, so the app should recommend RSV vaccination for adults with asthma starting at age 50 unless already vaccinated
- Mark clearly: `RSV vaccine is not currently annual`

### Zoster

- Recommend `2 doses recombinant zoster vaccine (Shingrix)` for adults `>=50 years`

### Tdap or Td

- If the patient has never received Tdap as an adult, recommend one dose of `Tdap`
- After that, recommend `Td or Tdap booster every 10 years`

### Vaccine output example

```json
{
  "prevention_recommendations": [
    {
      "vaccine": "Pneumococcal",
      "recommendation": "Recommend vaccination",
      "reason": "Adult age 52 with asthma and no prior PCV recorded",
      "default_options": [
        "PCV20 or PCV21 once",
        "PCV15 followed by PPSV23"
      ]
    },
    {
      "vaccine": "RSV",
      "recommendation": "Recommend vaccination",
      "reason": "Age 52 with chronic lung disease (asthma)",
      "note": "Single-dose vaccine; not currently annual"
    },
    {
      "vaccine": "Zoster",
      "recommendation": "Recommend 2-dose Shingrix series",
      "reason": "Age 52"
    },
    {
      "vaccine": "Tdap/Td",
      "recommendation": "Booster due",
      "reason": "Last tetanus-containing vaccine more than 10 years ago"
    }
  ]
}
```

## 14. UX Recommendations

### Main workflow

1. `Confirm diagnosis`
2. `Assess control and risk`
3. `Choose initial or follow-up pathway`
4. `Return medication recommendation`
5. `Return severe-asthma escalation plan if triggered`
6. `Return prevention and vaccine reminders`

### Recommendation screen sections

- `Diagnosis status`
- `Current control`
- `Preferred treatment`
- `Why this step`
- `What to monitor`
- `Severe asthma warning` if applicable
- `Vaccinations and prevention`
- `Guideline sources`

### Important UX rule

If diagnosis is not confirmed, the first card should say:

`Asthma is not yet objectively confirmed. Recommend repeat spirometry / peak flow / bronchoprovocation before locking in long-term stepped therapy unless immediate empiric treatment is clinically necessary.`

## 15. Suggested MVP Build Order

1. Diagnostic confirmation engine
2. GINA Track 1 adult treatment engine
3. Follow-up control and step-up logic
4. Severe asthma referral and phenotype branch
5. Vaccine engine
6. Medication library and source registry
7. Narrative explanation layer

## 16. Source Set Used for This Design

### Local guideline documents in this workspace

- [GINA 2025 Strategy Report PDF](/Users/abhichandel/Documents/Research/Asthma guideline app/GINA-2025-Update-25_11_08-WMS.pdf)
- [GINA 2025 Severe Asthma Guide PDF](/Users/abhichandel/Documents/Research/Asthma guideline app/GINA-Severe-Asthma-Guide-2025-WEB-FINAL-WMS.pdf)

### Web sources

- [GINA 2025 Global Strategy for Asthma Management and Prevention PDF](https://ginasthma.org/wp-content/uploads/2025/05/GINA-2025_tracked-for-archive-WMSA.pdf)
- [CDC Adult Immunization Schedule by Age](https://www.cdc.gov/vaccines/hcp/imz-schedules/adult-age.html)
- [CDC Adult Immunization Schedule by Medical Condition](https://www.cdc.gov/vaccines/hcp/imz-schedules/adult-medical-condition.html)
- [CDC Pneumococcal Vaccines for Adults](https://www.cdc.gov/pneumococcal/vaccines/adults.html)
- [CDC Pneumococcal Risk Factors](https://www.cdc.gov/pneumococcal/causes/index.html)
- [CDC RSV Vaccines for Adults](https://www.cdc.gov/rsv/vaccines/adults.html)
- [CDC Adult Vaccine Recommendations](https://www.cdc.gov/vaccines-adults/recommended-vaccines/index.html)
- [CDC Clinical Overview of Shingles](https://www.cdc.gov/shingles/hcp/clinical-overview/index.html)
- [CDC Td Vaccine VIS](https://www.cdc.gov/vaccines/hcp/current-vis/td.html)
- [CDC Seasonal Flu Vaccine Basics](https://www.cdc.gov/flu/vaccines/index.html)
- [CDC COVID-19 Vaccination Overview for Healthcare Professionals](https://www.cdc.gov/covid/hcp/vaccine-considerations/overview.html)
