Reputation Diagnostic Engine
1. Overview
The Reputation Diagnostic Engine is a public-source research and verification application for producing an evidence-traceable diagnostic on a public founder, CEO, or fund manager.
The core principle is:
Never present an unverified claim as an established fact, and explicitly record what the available evidence cannot establish.
The system separates:
Subject profiling
Public research
Claim extraction
Evidence verification
Deterministic status assignment
Gap analysis
Mandatory refusal
Diagnostic report generation
Human review and sign-off
Main capabilities
Accept a subject name and optional LinkedIn URL.
Use the LinkedIn URL only for identity disambiguation.
Research public, freely accessible sources.
Store source metadata and accessibility.
Extract discrete, checkable factual claims.
Detect superlatives such as "first", "largest", and "only".
Verify claims through multiple independent search passes.
Apply deterministic rules rather than allowing the LLM to directly decide the final status.
Assign VERIFIED, PARTIALLY VERIFIED, or UNVERIFIED.
Distinguish core conflicts from minor differences.
Flag load-bearing non-English evidence for bilingual review.
Identify exactly three evidence-based public-presence gaps.
Select a mandatory refused claim through rules.
Generate a compact diagnostic report.
Keep the report in draft state until a named human reviewer signs off.
Save checkpoints, run artifacts, logs, events, and approval records.
Preserve partial state when a pipeline stage fails.
2. End-to-End Workflow
Subject
   ↓
Subject Profiling
   ↓
Public Research
   ↓
Claim Extraction
   ↓
Verification
   ↓
Deterministic Rules
   ↓
Gap Analysis
   ↓
Mandatory Refusal
   ↓
Diagnostic Report
   ↓
Human Review & Sign-off
The system is designed so that evidence collection and factual status assignment remain separate from the final human approval.
3. Verification Model
Each extracted claim is checked through multiple independent verification passes.
The verification process considers:
Source quality
Source independence
Supporting evidence
Conflicting evidence
Minor differences
Core contradictions
Self-reported information
Superlative claims
Missing evidence
Technical failures
Statuses
VERIFIED
PARTIALLY VERIFIED
UNVERIFIED
A technical failure is recorded separately and is not silently treated as factual evidence.
Source hierarchy
Tier 1: Regulators, government, courts
Tier 2: Established independent press
Tier 3: Company or self-reported sources
Tier 4: Social media or unknown sources
Lower tier numbers represent stronger evidence.
4. Compound Claims
A claim containing multiple material assertions is evaluated component by component.
If a material component is contradicted:
UNVERIFIED
If a material component remains unsupported:
PARTIALLY VERIFIED
A claim is not promoted to verified merely because some parts of it are supported.
5. Superlatives
Claims containing terms such as:
first
largest
only
leading
receive additional scrutiny.
A self-reported superlative is not treated as independently established simply because other sources repeat the same statement.
6. Mandatory Refusal
The system deliberately excludes at least one claim from the publishable set.
The refusal is selected through rules rather than by asking the LLM whether it wants to refuse a claim.
The final diagnostic includes:
CLAIM REFUSED (MANDATORY EXCLUSION)
together with the reason the claim was excluded.
This ensures that the system explicitly identifies an evidence boundary instead of presenting every researched statement as established fact.
7. Gap Analysis
The system produces exactly three evidence-based public-presence gaps.
Typical gaps include:
Independently reported facts missing from the subject's public narrative.
Heavy reliance on self-reported information.
Inconsistent details between sources.
Negative, regulatory, or contested findings that are not publicly addressed.
Unsupported superlative or headline claims.
Lack of sufficient independent coverage.
Each gap must be traceable back to the verification evidence.
8. Diagnostic Report
The report contains four main sections:
1. FACT VERIFICATION
2. GAPS IN PUBLIC PRESENCE
3. CLAIM REFUSED (MANDATORY EXCLUSION)
4. RESEARCH LIMITATIONS (THIS RUN)
The diagnostic also contains:
Subject
Company
Jurisdiction
Run ID
Verification status
Evidence/basis
Source information
Reviewer sign-off
Research limitations
Technical failures where applicable
The report is not considered client-ready until a named human reviewer signs off.
9. Human Review
Human review is a required part of the workflow.
The reviewer can:
Review individual claims.
Approve, hold, or reject claims.
Check claims against named sources.
Add reviewer notes.
Sign off the diagnostic.
The approval record is stored with the run so that the final decision remains auditable.
10. Claude API
The system uses the Anthropic Claude API as its LLM provider.
Claude is used for tasks such as:
Public research assistance
Claim extraction
Evidence analysis
Verification reasoning
Structured output generation
Presentation compression where required
The final verification status is determined by the application's evidence rules rather than by Claude alone.
API credentials should be supplied through environment variables and must never be committed to the repository.
Example:
ANTHROPIC_API_KEY=your_api_key_here
11. Installation
Install the required dependencies:
pip install -r requirements.txt
Create the environment file from .env.example and configure the Claude API key.
Do not commit .env.
12. Running the Application
Start the reviewer console with:
streamlit run app.py
Then:
Enter the subject name.
Optionally enter a LinkedIn URL for identity disambiguation.
Select the verification and research settings.
Click Run Diagnostic.
Follow the pipeline stages.
Review the claims.
Review the sources.
Review the gaps and mandatory refusal.
Open the report and sign-off section.
Review the claims against their named sources.
Enter the reviewer name.
Approve, hold, or reject claims.
Complete the sign-off.
13. Run Artifacts
Each completed run stores its own artifacts.
Typical outputs include:
runs/<timestamp_subject>/
│
├── pipeline_data.json
├── diagnostic_report.txt
├── diagnostic_onepage.txt
├── diagnostic_report.docx
├── run.log
├── events.jsonl
├── APPROVAL.json
│
└── raw/
    ├── stage1_profile.json
    ├── stage2_research.txt
    ├── stage2_sources.json
    ├── stage3_claims.json
    ├── stage4_verified.json
    └── stage5_gaps.json
These artifacts allow a completed run to be inspected without repeating the entire research process.
14. Failure Handling
No evidence
The system records that no supporting public evidence was found.
Conflicting evidence
The conflict is surfaced rather than silently resolved.
Claude/API failure
The error is logged and associated with the relevant stage or verification pass.
JSON parsing failure
The system records the parsing failure and retries according to the configured retry policy.
Pipeline stage failure
Partial state and error information are preserved.
Report-generation failure
The underlying diagnostic data remains available even if presentation generation fails.
The system distinguishes between:
No public evidence found
and:
Technical/API/LLM failure
These are different failure states.
15. Public-Source Boundary
The system is designed for public-source research.
It does not intentionally use:
Private data
Login-gated data
Direct outreach
Automated messaging
Automated publishing
The LinkedIn URL is used for identity disambiguation only.
16. Known Limitations
The system does not claim perfect factual certainty.
Known limitations include:
Search ranking can affect discovery.
English sources may be easier to discover than non-English sources.
Some court or regulator records may not be freely accessible.
Several articles may repeat the same underlying source.
Source independence is not perfectly knowable in every case.
A single claim may contain multiple assertions.
Claude research/reasoning can fail or return incomplete evidence.
Human review remains necessary.
Broader testing is needed across low-data, non-English-heavy, and difficult subjects.
17. Security Checklist
Before publishing the repository:
[ ] Remove .env
[ ] Remove API keys
[ ] Remove cookies/session keys if present
[ ] Remove passwords
[ ] Remove virtual environments
[ ] Remove __pycache__
[ ] Check run logs for accidental secrets
[ ] Check sample run data before making it public
[ ] Keep .env.example
[ ] Keep requirements.txt
[ ] Keep README.md
18. Design Philosophy
The application deliberately separates model reasoning from deterministic evidence decisions.
The core chain is:
Claim
  ↓
Verification passes
  ↓
Evidence and sources
  ↓
Source quality / independence
  ↓
Conflicts / unsupported components
  ↓
Deterministic rules
  ↓
Final status
  ↓
Human review
The system is therefore not intended to be:
Ask LLM everything
       ↓
Trust final answer
It is intended to be:
Claude evidence/reasoning
        +
source metadata
        +
deterministic rules
        +
explicit refusal
        +
human approval
The result is an auditable workflow in which a reviewer can trace a conclusion back to the evidence, see what remains uncertain, identify technical failures, and see why a claim was refused.
19. Current Validation Status
The system has been tested end-to-end on a real subject and the workflow has been exercised across research, claim extraction, verification, gap analysis, refusal, and report generation.
Two areas still require substantial testing and validation:
Verification — improving consistency of claim-level verification, especially for compound claims, minor numerical differences, superlatives, and negative findings.
One-page diagnostic generation — reducing the original multi-page diagnostic to a compact approximately one-to-two-page output while preserving all important evidence, gaps, refusal, and limitations.
Further testing is required before treating the system as broadly validated across different subjects and evidence
