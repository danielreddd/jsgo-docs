# JSGo ATC — Database Schema

Postgres with pgvector. DiscipleMaker is the centre; everything else hangs off it. New tables from migrations 018 to 020 are noted below.

```mermaid
erDiagram
  Country ||--o{ DiscipleMaker : "country of"
  Language ||--o{ DiscipleMaker : "language of"
  Zone ||--o{ DiscipleMaker : "zone of"
  DiscipleMaker ||--o{ DiscipleMaker : "coaches"
  DiscipleMaker ||--o{ DailyReport : "submits"
  DiscipleMaker ||--o{ MonthlyReport : "rolls up to"
  DiscipleMaker ||--o{ dm_profile_quarterly : "has snapshots"
  DiscipleMaker ||--o{ dm_pending_disciples : "reports pending"
  DiscipleMaker ||--o{ dm_alerts : "is target of"
  DiscipleMaker ||--o{ VoiceIntakeSession : "has sessions"
  DiscipleMaker ||--o{ DMNote : "has notes"
  DiscipleMaker ||--o{ QAExchange : "asks questions"
  DiscipleMaker ||--o{ ActivityLog : "has audit events"
  User ||--o{ DMNote : "authors"
  User ||--o{ ActivityLog : "authors"

  DiscipleMaker {
    int id PK
    text dm_code UK
    text zoho_contact_id
    int coach_id FK
    int country_id FK
    int language_id FK
    int zone_id FK
    text full_name
    text mobile_number
    text gender
    text city
    text status
    text dm_preferred_mode
    text french_literacy
    text age_range
    text family_status
    text occupation
    text social_network_size
    text literacy_program_status
    text years_in_faith
    int counted_disciples
    int active_groups_count
    int total_new_churches
    bool mobile_verified
    bool penpal_listed
    text review_status
  }

  DailyReport {
    int id PK
    int dm_id FK
    date activity_date
    text activity_type
    text contacts_range
    int contacts_numeric
    int appointments_count
    bool commitment_ask
    text commitment_response
    bool group_met
    text group_no_meet_reason
    int group_number
    int attendees
    text attendees_range
    int story_number
    text discussion_leader
    int new_disciples
    text followup_scheduled
    int followup_visits_week
    bool multiplication_event
    text new_group_location
    text meeting_location
    text prayer_request_text
    text testimony_text
    text session_mode_used
    timestamp created_at
  }

  MonthlyReport {
    int id PK
    int dm_id FK
    date month
    int total_contacts
    int total_appointments
    int total_groups
    int total_new_disciples
    int total_new_churches
    timestamp created_at
  }

  dm_profile_quarterly {
    text dm_code PK
    text profile_quarter PK
    text family_support_level
    bool has_personal_discipler
    text discipler_role
    text personal_prayer_freq
    text opposition_level
    int spiritual_energy_score
    int evangelism_confidence_score
    int gospel_clarity_score
    text support_need
    timestamp completed_date
    text session_mode_used
  }

  dm_pending_disciples {
    uuid pending_id PK
    int dm_id FK
    date reported_date
    int reported_count
    text followup_status
    date confirmed_date
    text coach_notes
  }

  dm_alerts {
    uuid alert_id PK
    int dm_id FK
    text alert_rule_code
    timestamp triggered_at
    text recipient_type
    text recipient_code
    text priority
    int sla_hours
    text status
    timestamp acknowledged_at
    timestamp resolved_at
    text notification_text_fr
  }

  VoiceIntakeSession {
    int id PK
    int dm_id FK
    text session_type
    text interaction_mode
    text current_question_id
    jsonb session_data
    timestamp started_at
    timestamp completed_at
  }

  QAExchange {
    int id PK
    int dm_id FK
    text question_text
    text response_text
    jsonb sources
    timestamp created_at
  }

  TeachingChunk {
    int id PK
    text tier
    text source
    text reference
    text content
    vector embedding
  }

  DMNote {
    int id PK
    int dm_id FK
    int user_id FK
    text note_text
    timestamp created_at
  }

  ActivityLog {
    int id PK
    int dm_id FK
    int user_id FK
    text action
    jsonb metadata
    timestamp created_at
  }

  User {
    int id PK
    text email UK
    text full_name
    text role
    text password_hash
  }

  Country {
    int id PK
    text name
    text code
  }

  Language {
    int id PK
    int country_id FK
    text name
    text lang_code
  }

  Zone {
    int id PK
    int country_id FK
    text name
    text zone_code
  }
```

## What this diagram shows and what it leaves out

- **Centre of the schema is DiscipleMaker.** Every meaningful row in the application is connected to a DM, directly or via a session record.
- **New tables from the Phase 1 build** are `dm_profile_quarterly` (migration 018), `dm_pending_disciples` (migration 019), and `dm_alerts` (migration 020). These appear as separate boxes; everything else was already in place through migration 015.
- **The dm_pending_disciples table is the methodological centrepiece.** Every donor-facing disciple count flows through its `confirmed` status. See ADR-0002 for the rationale.
- **pgvector embeddings live on TeachingChunk.** RAG retrieval queries against the `embedding` column using cosine similarity. Tier A is Scripture plus Statement of Faith, Tier B is JSG training, Tier C is cultural notes (currently empty).
- **dm_profile_quarterly has a composite primary key** on `dm_code` plus `profile_quarter` (e.g. `2026-Q2`). One row per DM per quarter.
- **LT tables are NOT shown here.** They live in Zoho, not Postgres. See PRD-3 when LTC scope is activated.
- **Settings, audit-only logs, and similar housekeeping tables are omitted for clarity.** They exist but don't drive the data model.
