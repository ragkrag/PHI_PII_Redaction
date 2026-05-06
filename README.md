[Readme.md](https://github.com/user-attachments/files/27455405/Readme.md)
PII Redaction for AI Agent Telemetry (OCI)

Defense-in-depth approach to keeping PII out of OCI Logging and OCI APM when instrumenting an AI agent with OpenTelemetry and an agent framework.

The Problem

When you instrument an AI agent with OpenTelemetry and export traces/logs to OCI Logging or OCI APM, PII from conversations can end up in your telemetry — including:

Prompt text
Tool inputs/outputs
LLM responses

This data can include names, emails, SSNs, credit card numbers, and other sensitive information.

The Solution: Three Layers of Protection
┌────────────────────────────────────────────────────────────┐
│  User  ──→  Agent (full PII)  ──→  LLM                     │
│                    │                                       │
│                    ▼                                       │
│  Layer 1: enable_sensitive_data=False                      │
│    └─ Stops recording message content in OTel spans        │
│                    │                                       │
│  Layer 2: redact_pii() via Presidio (or custom OCI AI)     │
│    └─ Scrubs PII from any text before logging              │
│                    │                                       │
│  Layer 3: PIILoggingFilter on all log handlers             │
│    └─ Intercepts ALL log records before export             │
│                    ▼                                       │
│        OCI Logging / OCI APM (clean telemetry)             │
└────────────────────────────────────────────────────────────┘

Key design decision: The LLM always sees the full PII so it can respond accurately. Only the logs and telemetry are scrubbed.

PII Redaction Approaches (OCI Context)

OCI does not currently provide a direct, built-in PII redaction API equivalent to Azure AI Language. Therefore, we support:

Approach	Env var value	Pros	Cons
Presidio (local)	presidio	Fast, offline, no external calls	Pattern-based only (limited NLP)
Custom OCI AI (optional)	oci_ai	Can be extended with OCI Data Science / custom models	Requires custom implementation

Switch between them:

PII_REDACTION_METHOD=presidio   # default, local
PII_REDACTION_METHOD=oci_ai     # custom extension
Prerequisites
Python 3.10+
pip or uv
OCI account with:
OCI Logging enabled
(Optional) OCI APM enabled
OpenTelemetry-compatible setup
An LLM backend:
OpenAI
Azure OpenAI
GitHub Models
Quickstart
# Clone repo
git clone https://github.com/gameri87/PII-Redaction-Demo.git
cd PII-Redaction

# Configure environment
cp .env.sample .env
# Edit .env

# Install dependencies
pip install -r requirements.txt
# OR
uv sync

# Run
python main.py
Environment Variables
Variable	Required	Description
OPENAI_API_KEY	Yes	API key for LLM
OPENAI_MODEL	No	Default: gpt-4o-mini
OTEL_EXPORTER_OTLP_ENDPOINT	Optional	OCI APM OTLP endpoint
PII_REDACTION_METHOD	No	presidio or oci_ai
What It Tests

The demo runs two scenarios:

1. PII in user message

User sends:

Name
Email
SSN
Credit card
2. PII via tool response

lookup_customer returns:

Phone
Address
Account number
Other sensitive fields

👉 After running:

Check OCI Logging
Check OCI APM traces

You should see:

✅ Redacted telemetry
❌ No raw PII
Interesting Implementation Details
Thread-local recursion guard

Presidio initialization may log internally:

redact_pii → logging → filter → redact_pii → ...

Solved using:

threading.local()
Filter applied to ALL handlers

Critical detail:

OpenTelemetry logging/exporters bypass normal logger hierarchy
Filters must be applied to:
Root logger
Every handler
OCI Logging behavior
Anything sent via logging.* automatically flows to OCI Logging
Your logging filter is the primary protection layer
OCI APM (OpenTelemetry)

To enable tracing export:

export OTEL_EXPORTER_OTLP_ENDPOINT=https://apm-otel.<region>.oci.oraclecloud.com
Key Takeaway

This design ensures:

✅ LLM accuracy is preserved (full PII access)
✅ Telemetry is safe (PII removed)
✅ Protection is enforced even if developers forget
Limitations & Future Improvements
Presidio misses contextual entities (names, locations)
No native OCI PII detection service (yet)
Structured logs may still leak PII if deeply nested
Recommended enhancements:
Deep object redaction (recursive JSON sanitizer)
Async/batch redaction for performance
Custom OCI AI model for NLP-based detection
