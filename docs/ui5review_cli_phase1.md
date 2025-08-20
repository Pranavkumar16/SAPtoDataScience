# ui5review CLI Tool – Phase 1 Design

## 1. Goals and Scope
- Desktop command line tool that reviews SAPUI5 application archives against the rules in `KBN-SAPUI5-CodeReview-Guidelines.txt`.
- Generates a human readable report and structured JSON for feedback and training.
- Supports an online learning loop that updates a lightweight classifier after each review.
- Enforces guardrails: no blanket refactors, no secrets or PII in output, no API contract changes without explicit approval.

## 2. Command Line Experience
```
ui5review review <app.zip> [--rules KBN-SAPUI5-CodeReview-Guidelines.txt] \
                      [--context story.md] [--ml] \
                      [--report out/report.html]
ui5review export-feedback out/<review_id>_feedback.json
ui5review train --from out/<review_id>_feedback.json
ui5review model status
```
- `review` analyses the archive and emits the report and a findings JSON.
- `export-feedback` opens the interactive findings JSON, allowing reviewers to mark **Accept / Reject / Out of Scope** and add missed issues. The result is exported as feedback JSON.
- `train` ingests feedback JSON into the model.
- `model status` lists model versions, precision, recall and allows rollback.

## 3. Processing Pipeline
1. **Unpack input** – unzip archive and read optional context.
2. **Rule loader** – parse KBN guidelines; each rule has an ID, description, severity and detection logic.
3. **Static analysis** – traverse controllers, views, `manifest.json`, `Component.js`, i18n files and CSS.
4. **Findings aggregator** – capture each issue with:
   - Severity
   - File and line number
   - KBN section ID or `Std` (for best practice)
   - Issue text and suggested fix
5. **Auto-fix proposals** – minimal patches plus notes on side‑effects.
6. **Test recommendations** – list QUnit/OPA tests to add or update.
7. **Follow‑ups** – optional improvements outside the review scope.
8. **Report generator** – output HTML summary, risk score, confidence and findings table.

## 4. Feedback Loop
- The findings JSON stores every issue with a `status` field (`pending`, `accepted`, `rejected`, `out_of_scope`).
- Reviewers update statuses via a minimal TUI (textual user interface).
- Additional missed issues can be appended.
- `export-feedback` writes a JSON structure used for training.

### Feedback JSON Skeleton
```json
{
  "review_id": "<UUID>",
  "model_version": "v1.0.0",
  "feedback": [
    {"rule_id": "KBN-1.2", "status": "accepted"},
    {"rule_id": "Std-View-01", "status": "rejected"},
    {"rule_id": "KBN-3.4", "status": "missed"}
  ]
}
```
- `accepted` and `missed` contribute positive examples.
- `rejected` contributes negative examples.
- `out_of_scope` is ignored.

## 5. Machine Learning Integration
- Code segments are embedded using a frozen model (e.g. CodeBERT) and cached.
- A lightweight incremental classifier (e.g. `sklearn.linear_model.SGDClassifier`) consumes embeddings and labels.
- `train` performs online updates for single JSON files and supports batch retraining from a directory.
- Metrics (precision, recall, F1) are computed on a hold‑out set after each update.

## 6. Model Management
- Each model version is stored under `.ui5review/models/<semver>/` with:
  - `model.pkl` – classifier weights
  - `metadata.json` – version, training date, data sources and metrics
- `model status` lists versions and metrics and supports `--rollback <version>`.
- Training history allows comparison and restoration of previous models.

## 7. Safety Considerations
- Strip secrets/PII from analysis results and logs.
- Provide `--no-ml` switch to skip model updates in sensitive environments.
- Record precision and recall for audit; abort training if metrics fall below thresholds.
- All artefacts include the model version to guarantee traceability.

## 8. Future Work (Phase 2)
- Extend the feedback loop to learn from developer code fixes and generate patch suggestions.
- Introduce generation-aware tests and reinforcement learning with human feedback.

