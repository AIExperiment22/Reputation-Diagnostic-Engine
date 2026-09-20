# Reputation Diagnostic Engine

A Streamlit-based reviewer console that researches a public person or company, extracts factual claims, verifies them against public sources, identifies evidence gaps, and generates a one-page diagnostic report.

## Workflow

1. **Subject Profiling** — identifies the subject and company/jurisdiction.
2. **Public Research** — collects information from public sources.
3. **Claim Extraction** — converts research into factual claims.
4. **Verification** — performs multiple Claude-powered verification passes.
5. **Gap Analysis** — identifies unsupported, missing, conflicting, or self-reported information.
6. **Mandatory Refusal** — deliberately excludes at least one claim and records the reason.
7. **Report Generation** — creates the diagnostic report.

## Verification

Claude performs research and evidence analysis; deterministic rules determine the final status.

- **VERIFIED** — material parts are supported by credible evidence.
- **PARTIALLY VERIFIED** — some material evidence is missing or unclear.
- **UNVERIFIED** — sufficient evidence was not found or a core point is contradicted.
- Conflicting sources are surfaced rather than silently resolved.
- Superlative claims require stronger evidence.

## Source Policy

Only public sources are used.

Priority is generally given to:

1. Regulatory/government sources
2. Primary company or subject sources
3. Established news and industry publications
4. Other supporting public sources

Login-gated information, private outreach, and automated publishing are outside the system boundary.

## Mandatory Refusal

Every diagnostic excludes at least one claim from the verified set. The system records the refused claim, status, and reason. The refusal is selected by code, not by the model.

## Report

The diagnostic contains:

- Fact verification
- Sources and evidence
- Gaps in public presence
- Mandatory refused claim
- Research limitations
- Human reviewer sign-off

The report is not client-ready until a human reviewer signs off.

## Claude API

The system uses the **Anthropic Claude API** as its LLM provider.

Set the key in `.env`:

```env
ANTHROPIC_API_KEY=your_api_key_here
```

## Run

```bash
pip install -r requirements.txt
streamlit run app.py
```

Enter the subject name and optionally provide a LinkedIn URL for disambiguation.

## Run Artifacts

Each run stores research, claims, verification results, gap analysis, refusal information, final pipeline data, logs, and the diagnostic report.

## Limitations

- Search ranking and source availability can affect results.
- Non-English claims may require bilingual review.
- Court and regulatory records may have limited public accessibility.
- Claude may produce incorrect or incomplete interpretations.
- The system does not make autonomous reputational decisions.
