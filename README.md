# Adult Asthma Management Decision Support Tool

Guideline-mapped browser tool for adult asthma diagnosis support, treatment selection, severe-asthma escalation, and prevention prompts based primarily on GINA 2025.

## Overview

This repository contains a static single-page clinical decision support application for clinicians managing adults with suspected or established asthma. It mirrors the structure of the related COPD calculator with a lightweight front end:

- `index.html` for the interface
- `styles.css` for styling
- `app.js` for all rules-driven logic and note generation

The goal is to make guideline-aligned decision support easier to use at the point of care while keeping the logic transparent and easy to review.

## Features

- Initial and follow-up asthma workflows
- Diagnostic confirmation support using bronchodilator response, ICS response, and bronchoprovocation context
- ATS/ERS 2021 bronchodilator criterion support using `>10%` change relative to predicted `FEV1` or `FVC`
- GINA symptom-control classification
- GINA Track 1 treatment recommendations with explicit `budesonide-formoterol` AIR and MART dosing
- Step 5 and severe-asthma branching with non-biologic and biologic add-on guidance
- Tailored biologic guidance based on eosinophils, FeNO, allergic phenotype, maintenance oral steroids, nasal polyps, atopic dermatitis, and EGPA concern
- Vaccine and non-pharmacologic care prompts for adult asthma management
- Copy/paste-ready clinical note output

## Running locally

No build step is required.

1. Open `index.html` directly in a modern browser.

Or serve it locally:

```bash
python3 -m http.server 8765
```

Then open [http://127.0.0.1:8765/](http://127.0.0.1:8765/).

## Repository contents

- `index.html`: application UI
- `styles.css`: application styling
- `app.js`: clinical rules engine, rendering, and note generation
- `docs/`: design notes and seed rules created during development
- `LICENSE`: MIT license

## Evidence basis

The current implementation is mapped primarily to:

- GINA 2025 Global Strategy for Asthma Management and Prevention
- GINA 2025 Difficult-to-Treat and Severe Asthma Guide
- Current CDC adult vaccine guidance for the United States

Reference links:

- [GINA reports](https://ginasthma.org/reports/)
- [CDC adult immunization schedule](https://www.cdc.gov/vaccines/hcp/imz-schedules/adult-age.html)

Reference PDFs used during development are intentionally not distributed in this repository.

## Clinical disclaimer

This project is for clinician support only. It does not replace clinical judgment, objective testing, contraindication review, pregnancy-specific management, payer criteria, formulary constraints, or local and national policy.

## License

Released under the MIT License. See `LICENSE`.
