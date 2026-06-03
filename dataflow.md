# JSGo ATC — Data Flow

How data moves through ATC. Eight named flows trace the data paths; the diagram shows their shared components and dependencies.

**Flows.** ① Voice intake · ② Daily Zoho sync · ③ Scoring + alerts · ④ Coaching loop · ⑤ Encourager digest · ⑥ Givebutter webhook · ⑦ Stipend CSV · ⑧ Voice Q&A.

```mermaid
flowchart TB

  DM["DM phone<br/>(WhatsApp)"]
  TWILIO["Twilio"]
  BE["ATC backend"]
  S3[("S3 audio")]
  TRANSCRIBE["Transcribe"]
  PG[("Postgres<br/>+ pgvector")]
  ZOHO[("Zoho CRM")]
  SCORE["scoring_engine<br/>+ alert engine"]
  ALERTS[("dm_alerts")]
  COACH_LOOP["coaching_loop<br/>RAG → Bedrock"]
  BEDROCK["Bedrock"]
  RAG[("TeachingChunk<br/>pgvector")]
  POLLY["Polly TTS"]
  COACH["Coach phone"]
  TRANSLATE["translation.py"]
  DIGEST[("Encourager<br/>digest payload")]
  PP["PenPal"]
  GB["Givebutter"]
  STAGE[("staged donor<br/>events")]
  STIPEND["generate_payout.py"]
  CSV[("monthly stipend<br/>CSV")]
  FW["Flutterwave<br/>(Phase 2)"]
  QA["voice_qa"]

  DM -- "① voice note" --> TWILIO
  TWILIO -- "① audio URL" --> BE
  BE -- "① download<br/>+ store" --> S3
  S3 -- "① audio file" --> TRANSCRIBE
  TRANSCRIBE -- "① transcription" --> PG

  ZOHO -- "② COQL keyset<br/>daily 02:00 UTC" --> BE
  BE -- "② upsert DMs<br/>+ reports" --> PG

  PG -- "③ DAR + profile<br/>rows" --> SCORE
  SCORE -- "③ alert rules fired" --> ALERTS
  ALERTS -- "③ FR coaching audio<br/>via Polly + Twilio" --> COACH

  PG -- "④ DM context" --> COACH_LOOP
  RAG -- "④ Scripture +<br/>training content" --> COACH_LOOP
  COACH_LOOP -- "④ prompt" --> BEDROCK
  BEDROCK -- "④ response text" --> COACH_LOOP
  COACH_LOOP -- "④ FR text" --> POLLY
  POLLY -- "④ audio file" --> S3
  S3 -- "④ presigned URL<br/>via Twilio" --> COACH

  PG -- "⑤ weekly DM content<br/>(prayer + thanksgiving<br/>+ testimony)" --> TRANSLATE
  TRANSLATE -- "⑤ EN translation" --> DIGEST
  DIGEST -- "⑤ POST<br/>(Phase 2)" --> PP

  GB -- "⑥ HMAC webhook<br/>subscription.*" --> BE
  BE -- "⑥ staged events" --> STAGE
  STAGE -- "⑥ flip<br/>penpal_listed<br/>(Phase 2)" --> PG

  PG -- "⑦ active + listed DMs" --> STIPEND
  STIPEND -- "⑦ monthly CSV" --> CSV
  CSV -- "⑦ Phase 2" --> FW

  DM -- "⑧ free-form question<br/>(voice in FR)" --> TWILIO
  TWILIO --> BE
  BE --> TRANSCRIBE
  TRANSCRIBE --> QA
  QA --> BEDROCK
  RAG -.-> QA
  QA --> POLLY
  POLLY --> DM

  classDef store fill:#dcfce7,stroke:#15803d,color:#14532d;
  classDef service fill:#dbeafe,stroke:#1e40af,color:#1e3a8a;
  classDef ai fill:#f3e8ff,stroke:#7e22ce,color:#581c87;
  classDef ext fill:#fef3c7,stroke:#a16207,color:#78350f;
  classDef client fill:#fee2e2,stroke:#b91c1c,color:#7f1d1d;

  class S3,PG,RAG,ALERTS,DIGEST,STAGE,CSV store;
  class BE,SCORE,COACH_LOOP,TRANSLATE,STIPEND,QA service;
  class TRANSCRIBE,BEDROCK,POLLY ai;
  class ZOHO,PP,GB,FW,TWILIO ext;
  class DM,COACH client;
```

## The eight flows in plain words

1. **Voice intake.** DM speaks → Twilio → ATC backend → S3 → Transcribe → field on Postgres.
2. **Daily Zoho sync.** 02:00 UTC pull of Contacts + Daily_Reports + Monthly_Reports into Postgres via COQL with keyset pagination.
3. **Scoring and alerts.** Postgres data feeds scoring_engine; alert rules fire, write to dm_alerts; FR coaching audio goes to the coach via Polly + Twilio.
4. **Coaching loop.** Per-DM context plus the three-tier RAG corpus (Scripture, JSG training, cultural) feeds Bedrock to generate a coaching prompt; the response is voiced by Polly and delivered to the coach.
5. **Encourager digest.** Weekly DM content (prayer + thanksgiving + testimony) is translated FR→EN, packaged into a digest payload, and POSTed to PenPal in Phase 2.
6. **Givebutter webhook.** Donor subscription events arrive via HMAC-signed webhook, stage to a queue, and flip penpal_listed on the matched DM once PenPal donor-DM mapping is live (Phase 2).
7. **Stipend CSV.** Monthly generator produces a provider-agnostic CSV from active, listed DMs; Phase 2 feeds Flutterwave.
8. **Voice Q&A.** Free-form DM question → Transcribe → voice_qa retrieves from RAG, generates via Bedrock, voices via Polly, returns to DM. Citations stored on QAExchange.sources.
