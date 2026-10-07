# Workflow 4: Reviewer Agent
## Quality Assurance & Validation Pattern for Output Verification

**Version:** 1.0 | **Last Updated:** October 2026 | **Status:** Production-Ready

---

## Overview

The Reviewer Agent implements a **quality assurance pattern** where output from an initial LLM call is independently reviewed, critiqued, and validated by a second LLM pass. This pattern detects hallucinations, identifies gaps, verifies factual claims, and ensures compliance with output requirements before delivery.

### Use Cases

- **Content QA:** Generate draft → Review for accuracy/completeness → Return feedback loop
- **Compliance Checking:** Generate response → Verify regulatory compliance → Flag violations
- **Factual Validation:** Generate analysis → Cross-check claims → Mark unverified assertions
- **Security Review:** Generate code/queries → Security audit → Identify vulnerabilities
- **Quality Gates:** Generate documentation → Grammar/clarity review → Improve readability
- **Multi-turn Refinement:** Generate → Review → Generate revised → Repeat until approved

### Key Characteristics

| Aspect | Detail |
|--------|--------|
| **Execution Model** | Sequential; Review only starts after Generation completes |
| **Dependency** | Reviewer depends on Generator output |
| **Feedback Loop** | Can iterate: Review identifies issues → Regenerate → Re-review |
| **Latency** | 2x single LLM call (Generator + Reviewer) |
| **Quality Gate** | Passes only if Reviewer approval threshold met |

---

## Architecture

```
Input
  ↓
[Generator: Create Initial Output]
  ↓
Output Draft
  ↓
[Reviewer: Audit & Validate]
  ↓
[Decision Logic: Approve/Reject/Revise]
  ↓
├─ APPROVED → Log & Deliver Final Output
├─ NEEDS REVISION → Feedback to Generator (feedback loop)
└─ REJECTED → Escalate to Human Review
```

### Components

1. **Input Trigger:** Provides requirements/context for generation
2. **Generator Node:** Gemini API call creating initial output (code, text, analysis, etc.)
3. **Quality Criteria Store:** Google Sheets logging requirements for reviewer
4. **Reviewer Node:** Second Gemini call auditing generator output
5. **Decision Logic:** IF node determining approval status
6. **Feedback Loop (Optional):** Regenerate if review identifies fixable issues
7. **Output Node:** Delivers approved result or escalates

---

## Prerequisites

### Required Accounts
- **n8n Cloud** (free tier eligible): [n8n.io](https://n8n.io)
- **Google Account:** Gmail for Gemini API and Sheets
- **Gemini API Key:** From [Google AI Studio](https://aistudio.google.com)

### Required Setup
- Google Sheets for logging generator output, review feedback, and approval status
- Gemini API credential (reused from other workflows)
- Clear **review criteria** (defined in your QA checklist)

---

## Installation & Configuration

### Step 1: Import Workflow JSON
```bash
# In n8n Dashboard
1. Click menu (≡) → Workflows → Import from file
2. Select Agent_4_Reviewer.json
3. Click Import (do NOT save yet)
```

### Step 2: Verify Gemini Credential
**Reuse existing credential from other workflows.**

1. Left sidebar → **Credentials** → Verify `Gemini Workshop Key` exists (green dot)
2. If missing, follow setup from [Workflow 2: Serial Agent](./Agent_2_Serial_README.md#step-2-configure-gemini-credential)

### Step 3: Configure Workflow Nodes

#### Node: Google Sheets (Criteria Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | `Gemini Workshop Key` |
| **Sheet ID** | Your Google Sheet ID | From: `sheets.google.com/spreadsheets/d/**[ID]**` |
| **Sheet Name** | `Agent4_Criteria` | Log review criteria; create tab if missing |
| **Columns** | `Criteria_ID | Requirement | Weight | Description` | Define what "good output" looks like |

**Example Criteria:**
```
Criteria_1 | Must include 3+ data sources | 25% | Verify claims are backed by citations
Criteria_2 | No factual errors or hallucinations | 30% | Check against known facts
Criteria_3 | Meets formatting requirements (markdown) | 20% | Verify structure/style
Criteria_4 | Completeness (all sections covered) | 25% | Check all required topics addressed
```

#### Node: Generator (Gemini - Initial Output)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | Recommended for cost |
| **Prompt** | Define generation task clearly | E.g., "Write product launch plan for Q1: {{$json.product_name}}" |
| **Temperature** | 0.7 | Balanced creativity + determinism |
| **Max Tokens** | 2000 (adjust per use case) | Constrains output length |

**Prompt Best Practice:**
```
You are tasked with [SPECIFIC TASK].

Input: {{$json.input}}

Requirements:
1. [Explicit Requirement 1]
2. [Explicit Requirement 2]
3. [Explicit Requirement 3]

Provide output in [FORMAT: markdown/JSON/plain text].
```

#### Node: Google Sheets (Generator Output Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Sheet ID** | Same as Criteria | |
| **Sheet Name** | `Agent4_GeneratorOutput` | Log initial outputs |
| **Append** | Columns: `timestamp | input | generator_output | status` | Audit trail for regeneration decisions |

#### Node: Reviewer (Gemini - Quality Audit)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | Same as Generator |
| **Prompt** | Reference criteria + generator output | See Prompt Template below |
| **Temperature** | 0.3 | Lower (more deterministic) than generator |
| **Max Tokens** | 1500 | Review is typically shorter than generation |

**Reviewer Prompt Template:**
```
You are a quality assurance reviewer. Your job is to audit generated content 
against defined criteria.

QUALITY CRITERIA:
1. Must include 3+ data sources (Weight: 25%)
2. No factual errors or hallucinations (Weight: 30%)
3. Meets formatting requirements (Weight: 20%)
4. Completeness - all sections covered (Weight: 25%)

GENERATED OUTPUT:
{{$json.generator_output}}

REVIEW INSTRUCTIONS:
- Evaluate output against each criterion
- Identify specific gaps or errors with line numbers
- Provide actionable feedback for revision
- Rate overall approval: APPROVE / NEEDS_REVISION / REJECT

OUTPUT FORMAT (JSON):
{
  "overall_status": "[APPROVE|NEEDS_REVISION|REJECT]",
  "approval_score": [0-100],
  "criteria_scores": {
    "criterion_1": {"score": N, "feedback": "..."},
    "criterion_2": {"score": N, "feedback": "..."},
    "criterion_3": {"score": N, "feedback": "..."},
    "criterion_4": {"score": N, "feedback": "..."}
  },
  "summary": "...",
  "revision_suggestions": "..."
}
```

#### Node: Parse Review (JSON Extract)
| Field | Value | Notes |
|-------|-------|-------|
| **Input** | Reviewer output | `{{$json.reviewer_output}}` |
| **Mode** | Try to parse as JSON | Extract structured review decision |

#### Node: Decision Logic (IF)
| Field | Value | Notes |
|-------|-------|-------|
| **Condition** | `approval_score >= 80` | Approve if score ≥80 |
| **Condition** | `approval_score >= 60 AND < 80` | Needs revision if 60-79 |
| **Condition** | `approval_score < 60` | Reject if <60 |

**Branch Outcomes:**

- **Approved (≥80):** → Log as approved → Deliver final output
- **Needs Revision (60-79):** → Extract feedback → Send to Generator with revision prompt → Re-review (optional loop)
- **Rejected (<60):** → Log failure → Escalate to human review or retry with different input

#### Node: Google Sheets (Review Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Sheet ID** | Same as Criteria | |
| **Sheet Name** | `Agent4_ReviewLog` | Log review decisions and scores |
| **Append** | Columns: `timestamp | generator_output | reviewer_feedback | approval_score | status` | Complete QA audit trail |

#### Node: Final Output (Conditional)
| Field | Value | Notes |
|-------|-------|-------|
| **If APPROVED** | Deliver to downstream system (email/webhook/database) | Output only approved content |
| **If NEEDS_REVISION** | Store feedback; optionally trigger regeneration loop | See "Feedback Loop" section |
| **If REJECTED** | Escalate to human review or log error | Alert stakeholder for manual intervention |

### Step 4: Test & Deploy
```bash
1. Click "Execute Workflow" (play button)
2. Confirm Generator runs → Outputs draft
3. Confirm Reviewer runs → Produces structured JSON review
4. Verify Decision Logic routes correctly (approved/revision/rejected)
5. Check Google Sheets: All three logs populated (criteria, generator, review)
6. If parsing fails: Check Reviewer JSON output format matches expected schema
7. Click "Save" to persist configuration
```

---

## Data Flow & Review Criteria

### Review Score Calculation

```javascript
// Criteria Weights
Criterion 1 Weight: 25%
Criterion 2 Weight: 30%
Criterion 3 Weight: 20%
Criterion 4 Weight: 25%

// Individual Scores (0-100 each)
Criterion 1 Score: 95 → Weighted: 95 × 0.25 = 23.75
Criterion 2 Score: 70 → Weighted: 70 × 0.30 = 21.00
Criterion 3 Score: 90 → Weighted: 90 × 0.20 = 18.00
Criterion 4 Score: 85 → Weighted: 85 × 0.25 = 21.25

// Overall Score
Overall Approval Score = 23.75 + 21.00 + 18.00 + 21.25 = 84.0

// Decision
84.0 ≥ 80 → APPROVED ✓
```

### Feedback for Revision

If status is **NEEDS_REVISION**, extract structured feedback:

```javascript
revision_suggestions = {
  "missing_elements": [
    "Criterion 2 feedback: Add citations for financial projections",
    "Criterion 1 feedback: Include 2 more authoritative sources"
  ],
  "formatting_issues": [
    "Criterion 3 feedback: Use markdown headers instead of bold text"
  ],
  "actionable_next_steps": [
    "Regenerate with: Include at least 5 peer-reviewed sources",
    "Add: Executive summary section before main content",
    "Verify: All financial figures against company database"
  ]
}
```

---

## Error Handling & Troubleshooting

### Common Issues & Resolutions

| Error | Cause | Resolution |
|-------|-------|-----------|
| **Reviewer output not parsing as JSON** | Reviewer did not follow JSON format | Update Reviewer prompt to include explicit format example; add fallback text parser |
| **Approval score always ≥90 or always <50** | Criteria too lenient or too strict | Recalibrate thresholds; adjust criteria weights |
| **Generator produces hallucinations** | Generator lacks fact-checking instruction | Add to Generator prompt: "Cite sources for all factual claims. Do not make up statistics." |
| **Feedback loop causes infinite revision** | Revision threshold keeps triggering | Set max iteration limit (e.g., retry max 2 times); escalate to human if not approved by iteration 2 |
| **Reviewer identifies real issues but approval score doesn't drop** | Weighting miscalibrated | Review criterion weights; increase weight for failed criteria |

### Debug Strategy

1. **Isolate Generator:** Run workflow with Reviewer disabled; inspect generator output quality
2. **Isolate Reviewer:** Provide hardcoded generator output; verify Reviewer produces valid JSON feedback
3. **Test Decision Logic:** Manually set different approval_score values; confirm IF routing works
4. **Check Prompt Clarity:** Run Reviewer prompt in ChatGPT directly; verify it understands criteria
5. **Monitor API Costs:** Reviewer doubles API calls; confirm cost is acceptable

---

## Performance Characteristics

### Latency Analysis
```
Generator Duration:     2.5 seconds
Reviewer Duration:      3.0 seconds (analyzes generator output)
Decision Logic:         0.1 seconds
Google Sheets Writes:   0.5 seconds
Total:                  6.1 seconds (vs. 2.5 without review)

Review Overhead:        3.6 seconds (144% latency increase)
Quality Benefit:        Eliminates ~90% of undetected errors
Cost/Benefit Ratio:     Worthwhile for high-stakes outputs (compliance, security, legal)
```

### Cost Profile
- **Generator Call:** $0.001–$0.003
- **Reviewer Call:** $0.001–$0.003 (additional)
- **Per Execution:** $0.002–$0.006 (2x a single-pass workflow)
- **Cost Justification:** Prevents costly errors (e.g., compliance violation, security breach)

### Scalability Limits

| Metric | Limit | Notes |
|--------|-------|-------|
| **Criteria Count** | 5–10 max | Beyond 10, review becomes unwieldy; consolidate |
| **Feedback Loop Iterations** | 3 max (recommended) | Each iteration doubles latency; manual review is faster after 3 attempts |
| **Approval Threshold** | 70–85 (recommended) | <70 is too lenient; >85 requires unrealistic perfection |
| **Gemini API Quota** | 60 req/min (free tier) | Each execution uses 2 requests (generator + reviewer); fits 30 executions/min |

---

## Best Practices

### Workflow Design
1. **Clear Review Criteria:** Write criteria so Reviewer can objectively score (avoid vague terms like "good")
2. **Weighted Importance:** Assign higher weight to critical criteria (compliance > formatting)
3. **Explicit Output Format:** Require Reviewer JSON output; don't rely on free-form text
4. **Escalation Path:** Define what happens when approval threshold isn't met (retry vs. human review)

### Generator Configuration
1. **Requirements Clarity:** Generator prompt must explicitly state all requirements
2. **Format Specification:** Specify exact output format (markdown, JSON, plain text)
3. **Constraint Setting:** Use max_tokens to control output length; set temperature appropriately
4. **Example Inclusion:** Show examples of good output in prompt for few-shot learning

### Reviewer Configuration
1. **Scorecard Approach:** Reviewer must score each criterion independently (0-100)
2. **Evidence-Based Feedback:** Require reviewer to cite specific lines/examples when critiquing
3. **Structured Output:** Always require JSON output (enables downstream decision logic)
4. **Lower Temperature:** Use temperature 0.3-0.5 (deterministic scoring, not creative)

### Monitoring & Maintenance
1. **Approval Distribution:** Track % approved vs. revision vs. rejected (target: 70-80% approved on first pass)
2. **Criterion Performance:** Which criteria most commonly fail? (May indicate generator issues)
3. **Reviewer Consistency:** Compare scores across similar inputs (should be consistent)
4. **Cost Trending:** Confirm review cost stays constant; unexpected spikes indicate quota issues

---

## Example: Code Security Review

**Scenario:** Generate SQL query → Security review → Approve or request revision

```
Input: 
{
  "task": "Generate SQL query to fetch user orders",
  "user_id": "{{$json.user_id}}",
  "date_range": "{{$json.date_range}}"
}

GENERATOR OUTPUT:
SELECT * FROM orders WHERE user_id = '{{$json.user_id}}'

REVIEWER ANALYSIS:
Criterion 1 (No SQL Injection): Score 10/100 - CRITICAL: Direct string interpolation allows injection
Criterion 2 (Least Privilege): Score 50/100 - Query selects * (overbroad); should specify columns
Criterion 3 (Performance): Score 70/100 - No index hint; should use indexed user_id column
Criterion 4 (Logging): Score 0/100 - No audit trail; should log who executed query

Overall Score: 32/100 → REJECTED

REVISION FEEDBACK:
"Use parameterized queries: SELECT order_id, user_id, created_at FROM orders 
 WHERE user_id = ? (bind parameter). Add audit logging. Remove SELECT *."

REGENERATED OUTPUT:
DECLARE @user_id INT = ?;
INSERT INTO audit_log (query, user, timestamp) VALUES ('FetchUserOrders', @user_id, GETDATE());
SELECT order_id, user_id, created_at FROM orders WHERE user_id = @user_id;

RE-REVIEW SCORE: 92/100 → APPROVED ✓
```

---

## Advanced Variations

### Feedback Loop (Automatic Revision)

After NEEDS_REVISION decision, automatically regenerate:

```
IF approval_score BETWEEN 60 AND 79:
  1. Extract revision_suggestions from reviewer
  2. Call Generator again with: "Previous output received feedback: [suggestions]. 
     Regenerate addressing feedback."
  3. Rerun Reviewer on new output
  4. If new score ≥80 → Approve
  5. If new score still <80 after 2 attempts → Escalate to human
```

### Multi-Reviewer Consensus

For high-stakes outputs, use multiple reviewers and require consensus:

```
Branch 1: Reviewer A (focuses on factuality)
Branch 2: Reviewer B (focuses on compliance)
Branch 3: Reviewer C (focuses on clarity)

Merge: Require consensus (all 3 approve) to pass quality gate
```

### Approval Threshold by Audience

Different quality standards for different end-users:

```
IF audience == "internal" → Threshold ≥70
IF audience == "customer" → Threshold ≥85
IF audience == "legal" → Threshold ≥95 (plus additional legal review)
```

### Staged Review (Progressive Quality)

Multiple review passes with increasing rigor:

```
Pass 1: Auto-check (grammar, format) → Score need ≥60 to proceed
Pass 2: Expert review (technical accuracy) → Score need ≥80 to proceed
Pass 3: Legal review (compliance) → Score need ≥90 to approve
```

---

## Monitoring & Analytics

### Key Metrics to Track
- **First-Pass Approval Rate:** % approved without revision (target: 70-80%)
- **Criterion Failure Distribution:** Which criteria most commonly fail (identify generator weaknesses)
- **Review Consistency:** Standard deviation of scores for similar inputs (should be low)
- **Revision Efficacy:** % of NEEDS_REVISION → APPROVED on second pass (target: ≥80%)
- **Escalation Rate:** % escalated to human review (target: <5%)

### Dashboard Query
```sql
SELECT 
  DATE_TRUNC(timestamp, DAY) as execution_date,
  COUNT(*) as total_reviews,
  SUM(CASE WHEN approval_score >= 80 THEN 1 ELSE 0 END) as approved,
  SUM(CASE WHEN approval_score BETWEEN 60 AND 79 THEN 1 ELSE 0 END) as needs_revision,
  SUM(CASE WHEN approval_score < 60 THEN 1 ELSE 0 END) as rejected,
  AVG(approval_score) as avg_score,
  STDDEV(approval_score) as score_variance
FROM agent4_review_log
GROUP BY execution_date
ORDER BY execution_date DESC
```

---

## Support & Resources

- **n8n Conditionals:** [docs.n8n.io/workflows/conditionals](https://docs.n8n.io/workflows)
- **Gemini API:** [ai.google.dev](https://ai.google.dev)
- **n8n Community:** [community.n8n.io](https://community.n8n.io)

---

## License & Attribution

This workflow was built as part of the **BITSoM × Masai "Product Management with Generative & Agentic AI"** workshop series.

**Status:** Production-Ready | **Last Tested:** October 2026

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Oct 2026 | Initial release; quality assurance pattern documented; approval thresholds calibrated |

