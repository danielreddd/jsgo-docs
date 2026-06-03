# JSGo ATC — System Architecture

Components and connections inside the ATC platform. For data movement see dataflow.md; for the Postgres schema see schema.md.

**Legend.** ATC services · Datastore · AWS managed AI · External service · Client device.

```mermaid
flowchart LR

  subgraph CLIENT["Client devices"]
    direction TB
    DM_WA["DM WhatsApp<br/>(low-cost Android)"]
    COACH_WA["Coach WhatsApp"]
    BROWSER["Admin / Coach<br/>browser"]
  end

  subgraph EDGE["Edge"]
    TWILIO["Twilio<br/>WhatsApp Business API"]
    ALB["AWS ALB + WAF<br/>HTTPS only"]
  end

  subgraph ATC["JSG ATC on AWS ECS Fargate (us-east-1)"]
    direction TB
    FE["Next.js 14 frontend<br/>NextAuth JWT<br/>dashboard pages"]
    BE["FastAPI backend<br/>async SQLAlchemy<br/>APScheduler"]
    FE -->|JWT proxied API calls| BE
  end

  subgraph DATA["Data layer"]
    direction TB
    PG[("RDS Postgres 16<br/>+ pgvector")]
    S3[("S3 bucket<br/>jsg-atc-audio")]
  end

  subgraph AI["AWS managed AI"]
    direction TB
    BR["Bedrock<br/>Claude Haiku / Sonnet<br/>Titan embeddings"]
    POLLY["Polly<br/>FR (Lea) · EN (Joanna)"]
    TRANSCRIBE["Transcribe<br/>fr-FR / en-US"]
  end

  subgraph EXT["External services"]
    direction TB
    ZOHO[("Zoho CRM<br/>shared org")]
    GB["Givebutter<br/>donor subscriptions"]
    PP["PenPal<br/>(Eder + Autumn)"]
    FW["Flutterwave<br/>(Phase 2)"]
  end

  DM_WA <-->|"WhatsApp messages<br/>+ voice"| TWILIO
  COACH_WA <-->|"WhatsApp messages<br/>+ FR coaching audio"| TWILIO
  BROWSER -->|HTTPS| ALB

  TWILIO <-->|"webhook + API"| BE
  ALB --> FE
  ALB --> BE

  BE <-->|"SQL + pgvector"| PG
  BE -->|"audio store<br/>+ presigned URLs"| S3

  BE -->|"prompts + RAG"| BR
  BE -->|"TTS"| POLLY
  BE -->|"voice notes"| TRANSCRIBE

  BE <-->|"COQL keyset sync<br/>daily 02:00 UTC"| ZOHO
  BE <-.->|"HMAC webhook (in)<br/>donor events"| GB
  BE <-.->|"digest delivery<br/>donor-DM mapping"| PP
  BE -.->|"monthly stipend CSV"| FW

  classDef atc fill:#dbeafe,stroke:#1e40af,color:#1e3a8a;
  classDef data fill:#dcfce7,stroke:#15803d,color:#14532d;
  classDef ai fill:#f3e8ff,stroke:#7e22ce,color:#581c87;
  classDef ext fill:#fef3c7,stroke:#a16207,color:#78350f;
  classDef client fill:#fee2e2,stroke:#b91c1c,color:#7f1d1d;
  classDef edge fill:#f1f5f9,stroke:#64748b,color:#1e293b;

  class FE,BE atc;
  class PG,S3 data;
  class BR,POLLY,TRANSCRIBE ai;
  class ZOHO,GB,PP,FW ext;
  class DM_WA,COACH_WA,BROWSER client;
  class TWILIO,ALB edge;
```
