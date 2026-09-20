# Reputation Diagnostic Engine

## 1. Overview

The Reputation Diagnostic Engine is a public-source research and
verification application for producing an evidence-traceable diagnostic
on a public founder, CEO, or fund manager.

The core principle is:

> Never present an unverified claim as an established fact, and
> explicitly record what the available evidence cannot establish.

The application separates research, claim extraction, verification,
deterministic status assignment, gap analysis, mandatory refusal, report
generation, and human approval.

The current application uses a Streamlit reviewer console and an
OpenRouter-based LLM transport with web search enabled through
OpenRouter's web plugin.

### Main capabilities

-   Accept a subject name and optional LinkedIn URL.
-   Use the LinkedIn URL only for identity disambiguation. The LinkedIn
    profile itself is not scraped.
-   Research public, freely accessible sources.
-   Store URL, date, language, source type, tier, support and
    accessibility for sources.
-   Extract discrete, checkable factual claims.
-   Detect superlatives such as "first", "largest", and "only".
-   Verify claims through multiple independent search passes.
-   Apply deterministic rules rather than letting the LLM directly
    assign the final status.
-   Assign `VERIFIED`, `PARTIALLY VERIFIED`, or `UNVERIFIED`.
-   Distinguish core conflicts from minor/compatible differences.
-   Flag load-bearing non-English evidence for bilingual review.
-   Identify exactly three evidence-based public-presence gaps.
-   Select a mandatory refused claim through code/rules.
-   Generate a compact one-page diagnostic.
-   Keep the report in draft state until a named human reviewer signs
    off.
-   Save checkpoints, run artifacts, logs, events, and approval records.
-   Preserve partial state when a pipeline stage fails.
-   Fall back to the original report if Stage 6 presentation compression
    fails.

------------------------------------------------------------------------

## 2. Architecture

``` text
Streamlit UI: app.py
        |
        v
Pipeline: pipeline.py
        |
        +--> Stage 1: s1_profile.py
        |
        +--> Stage 2: s2_research.py
        |
        +--> Stage 3: s3_extract.py
        |
        +--> Stage 4: s4_verify.py
        |          |
        |          v
        |      rules.py
        |
        +--> Stage 5: s5_gaps.py
        |
        +--> Mandatory refusal: rules.py
        |
        +--> Stage 6: s6_report.py
                   |
                   v
              Human sign-off

LLM boundary:
stages --> llm_client.py --> OpenRouter --> web plugin
```

------------------------------------------------------------------------

## 3. Repository Structure

``` text
Reputation-Diagnostic-Engine/
|
├── app.py
├── pipeline.py
├── config.py
├── llm_client.py
├── models.py
├── rules.py
├── storage.py
├── logging_setup.py
├── house_style.py
|
├── stages/
│   ├── s1_profile.py
│   ├── s2_research.py
│   ├── s3_extract.py
│   ├── s4_verify.py
│   ├── s5_gaps.py
│   └── s6_report.py
|
├── requirements.txt
├── .env.example
|
└── runs/
    └── <timestamp_subject>/
        ├── pipeline_data.json
        ├── diagnostic_report.txt
        ├── diagnostic_onepage.txt
        ├── diagnostic_report.docx
        ├── run.log
        ├── events.jsonl
        ├── APPROVAL.json
        └── raw/
            ├── stage1_profile.json
            ├── stage2_research.txt
            ├── stage2_sources.json
            ├── stage3_claims.json
            ├── stage4_verified.json
            └── stage5_gaps.json
```

Keep the stage files separate. Do not merge them into one Python file
merely to reduce repository size.

------------------------------------------------------------------------

## 4. `app.py` --- Reviewer Console

`app.py` is the Streamlit operator and reviewer interface.

Run it with:

``` bash
streamlit run app.py
```

The application is intentionally a reviewer console rather than a simple
"enter a name and get an answer" interface.

### Run tab

Inputs:

-   Subject name
-   Optional LinkedIn URL

The LinkedIn URL is used for disambiguation only. The profile itself is
not scraped.

The sidebar exposes:

-   Model selection
-   Verification passes
-   Maximum claims
-   Research searches
-   Searches per verification pass
-   API-key presence

The GUI streams these stages:

1.  Subject profiling
2.  Public research
3.  Claim extraction
4.  Verification passes
5.  Gap analysis
6.  Mandatory refusal
7.  Report generation

### Report and sign-off

This is the human gate.

The reviewer can:

-   Review claims.
-   Approve, hold, or reject individual claims.
-   Enter a reviewer name.
-   Add reviewer notes.
-   Confirm approved claims were checked against named sources.
-   Sign off the run.

Approval is written to `APPROVAL.json`.

### Claims tab

Shows:

-   Verified count
-   Partially verified count
-   Unverified count
-   Bilingual review count
-   Claim category
-   Independent source count
-   Best source tier
-   Conflict
-   Superlative flag
-   Bilingual flag
-   Verification reasoning
-   Verification passes
-   Rules applied
-   Sources
-   Technical failures

### Sources tab

Shows:

-   Public sources
-   Inaccessible sources
-   Subject profile
-   Raw research

### Logs tab

Shows:

-   Stage failures
-   Verification-pass failures
-   LLM retries
-   JSON parsing failures
-   DOCX failures
-   House-style warnings
-   Event stream
-   `run.log`

### Run history

Shows previous runs, claim counts, verification counts, state, and
reviewer information.

------------------------------------------------------------------------

## 5. `config.py` --- Central Configuration

`config.py` centralizes tunable settings and loads environment variables
with `python-dotenv`.

Important settings include:

``` text
VERIFY_PASSES
MAX_CLAIMS
MAX_SEARCHES_RESEARCH
MAX_SEARCHES_VERIFY
LLM_RETRIES
LLM_RETRY_DELAY
MAX_TOKENS
RUNS_DIR
```

It also defines:

``` text
STATUS_VERIFIED
STATUS_PARTIAL
STATUS_UNVERIFIED
```

### Verification depth

`VERIFY_PASSES` is constrained to 2 or 3.

### Search limits

`MAX_SEARCHES_RESEARCH` controls research-stage search volume.

`MAX_SEARCHES_VERIFY` controls searches available to each verification
pass.

### Source hierarchy

The current configuration maps:

``` text
Tier 1: regulator, government, court
Tier 2: press
Tier 3: company, self
Tier 4: social, unknown
```

Lower tier number means stronger evidence.

Independent/strong tiers are 1 and 2.

### Important provider note

The active `llm_client.py` uses OpenRouter. The repository's `config.py`
still contains legacy Anthropic/Claude configuration fields from the
earlier transport. The active provider credentials are controlled by
`OPENROUTER_API_KEY` and `OPENROUTER_MODEL` in the current OpenRouter
transport.

------------------------------------------------------------------------

## 6. Environment Configuration

Do not commit `.env`.

Create it from `.env.example`.

Current OpenRouter configuration:

``` env
OPENROUTER_API_KEY=your_api_key_here
OPENROUTER_MODEL=openrouter/free
```

Never commit:

-   API keys
-   cookies
-   session keys
-   passwords
-   private credentials

------------------------------------------------------------------------

## 7. `llm_client.py` --- LLM Transport

`llm_client.py` isolates provider-specific logic from the rest of the
application.

The active provider is OpenRouter through its OpenAI-compatible API:

``` text
https://openrouter.ai/api/v1
```

The active model comes from:

``` text
OPENROUTER_MODEL
```

The current default is:

``` text
openrouter/free
```

### `call_llm()`

Responsibilities:

-   Send a prompt to OpenRouter.
-   Enable the web plugin internally.
-   Validate the response.
-   Reject empty responses.
-   Retry failed requests.
-   Return response text.

The active `call_llm()` intentionally has no `search` argument.

### `call_llm_json()`

Responsibilities:

-   Preserve compatibility with existing stages.
-   Add strict JSON instructions.
-   Call `call_llm()`.
-   Strip accidental Markdown fences.
-   Recover JSON surrounded by extra text when possible.
-   Retry malformed JSON.
-   Log JSON parsing failures.
-   Return:

``` text
(parsed_json, retrieved_urls)
```

Existing stages may still pass `search`, `max_searches`, and `stage` to
`call_llm_json()`. Those parameters are accepted for compatibility and
are not passed into `call_llm()`.

### Web search

OpenRouter web search is enabled inside the actual provider request
through its web plugin.

------------------------------------------------------------------------

## 8. `models.py` --- Data Contracts

Important models include:

### `Source`

``` text
name
url
date
language
source_type
tier
supports_claim
access
note
```

### `Claim`

``` text
claim
category
self_reported
source_language
contains_superlative
superlative_terms
```

### `VerificationPass`

``` text
pass_number
model_status
conflict
reasoning
sources
error
retrieved_urls
```

### `VerifiedClaim`

``` text
claim
status
reasoning
passes
sources
independent_source_count
best_tier
conflict
bilingual_review_required
rule_notes
system_error
```

### `Gap`

``` text
title
explanation
basis
```

### `RunResult`

Contains the complete run state including profile, research, sources,
claims, verification, gaps, refused claim, refusal reason, stage errors,
and limitations.

------------------------------------------------------------------------

## 9. `pipeline.py` --- Orchestration

`pipeline.py` controls the complete workflow.

It exposes:

``` text
iter_run()
run()
```

`iter_run()` is used by Streamlit to stream progress.

`run()` is the blocking execution path.

Both use the same underlying stage logic.

### Stage order

``` text
1. Subject profiling
2. Public research
3. Claim extraction
4. Verification passes
5. Gap analysis
6. Mandatory refusal
7. Report generation
```

### Checkpointing

After stages complete, artifacts are written to disk.

If a stage fails:

-   The stage error is recorded.
-   The traceback is logged.
-   Partial state is preserved.
-   `pipeline_data.partial.json` is available.
-   The GUI identifies the failed stage.

------------------------------------------------------------------------

## 10. Stage 1 --- `s1_profile.py`

Establishes the subject context before broader research.

It is intended to establish:

-   Name
-   Company
-   Country
-   Jurisdiction
-   Industry/context
-   Likely languages
-   Relevant regulators

The optional LinkedIn URL helps identity disambiguation only.

Jurisdiction matters because it determines which regulators, government
records, courts, and local-language sources are relevant.

------------------------------------------------------------------------

## 11. Stage 2 --- `s2_research.py`

Performs targeted public research.

Research categories include:

-   Funding history
-   Founding information
-   Company milestones
-   Recent news
-   Awards
-   Regulatory information
-   Relevant public court/regulator records
-   Public narrative information

The stage stores source metadata, not only research prose.

Outputs:

``` text
raw/stage2_research.txt
raw/stage2_sources.json
```

Inaccessible or login-gated sources are recorded as inaccessible rather
than treated as successfully reviewed.

------------------------------------------------------------------------

## 12. Stage 3 --- `s3_extract.py`

Converts research into discrete, checkable factual claims.

It prioritizes:

-   Numbers
-   Dates
-   Named entities
-   Named milestones
-   Funding
-   Founding facts
-   Regulatory facts
-   Controversy information

It skips vague opinions and unfalsifiable marketing language.

The stage also runs code-level superlative detection.

Examples:

``` text
first
largest
only
leading
```

Output:

``` text
raw/stage3_claims.json
```

------------------------------------------------------------------------

## 13. Stage 4 --- `s4_verify.py`

This is the main evidence-verification stage.

Each claim is checked through 2--3 independent verification passes.

Each pass searches for evidence and records:

-   Model status opinion
-   Reasoning
-   Sources
-   Source metadata
-   Conflict
-   Retrieved URLs
-   Technical errors

The LLM is instructed not to invent sources and to return an empty
source set when evidence is unavailable.

### Conflict types

``` text
none
minor
core
```

`core` means the central fact is contradicted.

`minor` means sources agree on the substance but differ on a detail such
as a figure or date.

A technical verification failure is recorded as a system error, not
silently treated as factual evidence.

------------------------------------------------------------------------

## 14. `rules.py` --- Deterministic Verification

`rules.py` is the main fact-integrity layer.

The LLM provides evidence and reasoning. It does not directly control
the final publishability label.

Conceptually:

``` text
LLM evidence
    ↓
Source normalization
    ↓
Independent-source evaluation
    ↓
rules.py
    ↓
Final status
```

### Base status logic

``` text
2+ independent credible source domains
    -> VERIFIED

1 independent source / supporting evidence
    -> PARTIALLY VERIFIED

No supporting public evidence
    -> UNVERIFIED
```

### Source-tier rule

Tier-4-only evidence cannot produce `VERIFIED`.

### Conflict rule

A core conflict produces `UNVERIFIED`.

The refined rule implementation does not automatically downgrade
otherwise sufficient evidence merely because sources have a minor
numerical/date variation that does not contradict the core claim.

### Superlative rule

Superlatives require stronger evidence.

If a superlative does not have two strongest-tier independent sources,
the status is downgraded.

A self-reported superlative is not promoted to `VERIFIED` merely because
secondary sources repeat the same underlying statement.

### Compound claims

Claims containing several material assertions are evaluated component by
component.

If a material component is contradicted:

``` text
UNVERIFIED
```

If a material component remains unsupported:

``` text
PARTIALLY VERIFIED
```

### Bilingual review

When load-bearing independent evidence is non-English and English
evidence alone is insufficient, the claim is flagged for bilingual
reviewer sign-off.

------------------------------------------------------------------------

## 15. Mandatory Refusal

At least one claim is deliberately excluded from the publishable set.

This is selected by code/rules, not by asking the LLM whether it wants
to refuse.

The selection prioritizes an `UNVERIFIED` claim and has additional
evidence-sensitive handling for suitable partially verified/superlative
claims.

The report includes:

``` text
CLAIM REFUSED (MANDATORY EXCLUSION)
```

and the reason for exclusion.

------------------------------------------------------------------------

## 16. Stage 5 --- `s5_gaps.py`

Produces exactly three evidence-based public-presence gaps.

Patterns include:

-   Independently reported fact missing from subject materials.
-   Over-reliance on self-reported information.
-   Inconsistent details across sources.
-   Negative, regulatory, or contested finding that is publicly
    unaddressed.
-   Lack of recent independent coverage.
-   Unsupported superlative/headline claim.

Each gap must point back to the verification data.

Output:

``` text
raw/stage5_gaps.json
```

------------------------------------------------------------------------

## 17. Stage 6 --- `s6_report.py`

Generates the fixed one-page diagnostic.

The report contains:

``` text
1. FACT VERIFICATION
2. GAPS IN PUBLIC PRESENCE
3. CLAIM REFUSED (MANDATORY EXCLUSION)
4. RESEARCH LIMITATIONS (THIS RUN)
```

It also contains subject, company, jurisdiction, run ID, reviewer
sign-off, claim evidence, rule notes, and technical limitations.

### One-page compression

The LLM is used only for presentation compression.

It must not:

-   Research again.
-   Change statuses.
-   Add facts.
-   Add sources.
-   Upgrade/downgrade claims.
-   Remove the refused claim.
-   Remove the three gaps.

### Compression retry policy

The current implementation attempts compression up to three times:

``` text
Attempt 1
    |
    failure
    |
    wait 120 seconds
    |
Attempt 2
    |
    failure
    |
    wait 120 seconds
    |
Attempt 3
    |
    failure
    |
fallback to original report
```

Every failure is logged.

If all attempts fail, the original diagnostic is used. The underlying
diagnostic data is not destroyed.

Outputs:

``` text
diagnostic_report.txt
diagnostic_onepage.txt
diagnostic_report.docx
```

------------------------------------------------------------------------

## 18. `house_style.py`

Applies and checks the project's report-writing style.

Stage 6 runs the style check after compression.

Remaining issues are logged as `house_style_warning`.

The style layer does not replace verification.

------------------------------------------------------------------------

## 19. `storage.py`

Handles run directories and persisted artifacts.

Each execution receives a dedicated run directory:

``` text
runs/<timestamp_subject>/
```

The storage layer provides run IDs and JSON/checkpoint persistence.

This allows completed runs to be inspected without rerunning the
research.

------------------------------------------------------------------------

## 20. `logging_setup.py`

Logging is separated from business logic.

The application records:

``` text
run.log
events.jsonl
```

Events can include:

``` text
run_start
stage start/done
stage_failed
pass_failed
llm_retry
json_parse_failed
compression_failed
compression_retry_wait
compression_fallback
house_style_warning
docx_failed
done
```

The purpose is to distinguish evidence failure from technical failure.

For example:

``` text
"No public evidence found"
```

is different from:

``` text
"Search/API/LLM failed"
```

------------------------------------------------------------------------

## 21. Run Artifacts

A typical completed run contains:

``` text
pipeline_data.json
```

Complete structured pipeline state.

``` text
pipeline_data.partial.json
```

Partial state if the run stops before completion.

``` text
raw/stage1_profile.json
```

Subject profile.

``` text
raw/stage2_research.txt
```

Research text.

``` text
raw/stage2_sources.json
```

Research source metadata.

``` text
raw/stage3_claims.json
```

Extracted claims.

``` text
raw/stage4_verified.json
```

Verification results, sources, passes, conflicts and rules.

``` text
raw/stage5_gaps.json
```

Three identified gaps.

``` text
diagnostic_report.txt
diagnostic_onepage.txt
diagnostic_report.docx
```

Final report outputs.

``` text
events.jsonl
run.log
```

Audit/log data.

``` text
APPROVAL.json
```

Human approval record after sign-off.

------------------------------------------------------------------------

## 22. Human Approval Flow

The intended flow is:

``` text
Research
   ↓
Verification
   ↓
Draft diagnostic
   ↓
Human review
   ↓
Claim decisions
   ↓
Sign-off
   ↓
Client-ready state
```

The system does not automatically treat a generated report as
client-ready.

The reviewer can approve, hold, or reject individual claims.

Reviewer name, timestamp, notes, and claim decisions are persisted.

------------------------------------------------------------------------

## 23. Installation

Install the repository dependencies:

``` bash
pip install -r requirements.txt
```

Create `.env` from `.env.example`.

Configure:

``` env
OPENROUTER_API_KEY=your_api_key_here
OPENROUTER_MODEL=openrouter/free
```

Do not commit `.env`.

------------------------------------------------------------------------

## 24. Run the Application

Start Streamlit:

``` bash
streamlit run app.py
```

Then:

1.  Enter the subject name.
2.  Optionally enter a LinkedIn URL for disambiguation.
3.  Select verification/search settings.
4.  Click `Run Diagnostic`.
5.  Wait for the pipeline stages.
6.  Inspect Claims.
7.  Inspect Sources.
8.  Inspect the mandatory refusal and gaps.
9.  Open Report and sign-off.
10. Review individual claims.
11. Enter reviewer name.
12. Approve, hold, or reject claims.
13. Complete sign-off.

------------------------------------------------------------------------

## 25. Programmatic Pipeline

The pipeline exposes:

``` python
pipeline.iter_run(subject_name, linkedin_url, verbose=False)
```

for streaming execution.

The blocking pipeline path uses `pipeline.run()`.

The GUI uses `iter_run()` rather than maintaining a separate
implementation.

This prevents the GUI and CLI from drifting into different business
logic.

------------------------------------------------------------------------

## 26. Failure Handling

### No evidence

The system records that no supporting public evidence was found.

### Conflicting evidence

The conflict is surfaced and classified.

### API/LLM failure

The error is logged and associated with the relevant stage/pass.

### JSON parsing failure

The transport retries and records the parsing event.

### Pipeline stage failure

The system saves partial state and traceback information.

### DOCX failure

Text output remains available.

### Stage 6 compression failure

Three presentation-compression attempts are made with 120-second waits
between failed attempts. If all three fail, the original report is
preserved.

------------------------------------------------------------------------

## 27. Public-Source Boundary

The system is designed for public-source research.

It does not intentionally use:

-   Private data
-   Login-gated data
-   Direct outreach
-   Automated messaging
-   Automated publishing

The LinkedIn URL is used for identity disambiguation only.

------------------------------------------------------------------------

## 28. Known Limitations

The system does not claim perfect factual certainty.

Known limitations include:

-   Search ranking affects discovery.
-   English sources may be easier to discover than non-English sources.
-   Some court/regulator records may not be freely accessible.
-   Several articles may repeat the same underlying source.
-   Source independence is not perfectly knowable in every case.
-   A single claim may contain multiple assertions.
-   LLM search/reasoning can fail or return incomplete evidence.
-   Human review remains necessary.
-   Broader testing is needed across low-data, non-English-heavy, and
    difficult subjects.

------------------------------------------------------------------------

## 29. Security Checklist Before Publishing

``` text
[ ] Remove .env
[ ] Remove API keys
[ ] Remove cookies/session keys
[ ] Remove passwords
[ ] Remove virtual environments
[ ] Remove __pycache__
[ ] Check run logs for accidental secrets
[ ] Check sample run data before making it public
[ ] Keep .env.example
[ ] Keep requirements.txt
[ ] Keep README.md
```

------------------------------------------------------------------------

## 30. Current LLM Provider

The active transport is OpenRouter.

The previous Claude/Anthropic transport is retained as
reference/commented code in the transport file but is not the active
provider path.

Active API base:

``` text
https://openrouter.ai/api/v1
```

Active model:

``` text
OPENROUTER_MODEL
```

Default:

``` text
openrouter/free
```

Web search is enabled through the OpenRouter web plugin.

------------------------------------------------------------------------

## 31. Design Philosophy

The application deliberately separates model reasoning from
deterministic evidence decisions.

The important chain is:

``` text
Claim
  ↓
Verification passes
  ↓
Sources
  ↓
Source quality / independence
  ↓
Conflicts / unsupported components
  ↓
rules.py
  ↓
Final status
  ↓
Human review
```

The system is therefore not intended to be:

``` text
Ask LLM everything
        ↓
Trust final answer
```

It is intended to be:

``` text
LLM evidence/reasoning
        +
source metadata
        +
deterministic rules
        +
explicit refusal
        +
human approval
```

The result is an auditable workflow in which a reviewer can trace a
conclusion back to the evidence, see what remains uncertain, identify
technical failures, and see why a claim was refused.
