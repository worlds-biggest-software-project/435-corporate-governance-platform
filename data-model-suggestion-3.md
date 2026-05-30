# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Corporate Governance Platform (Candidate #435)
> Generated: 2026-05-25

## Overview

This model takes a pragmatic hybrid approach: use normalized relational tables for the stable, well-understood core of the governance domain (boards, meetings, users, votes, entities) while leveraging PostgreSQL's native JSONB columns for the parts of the domain that are inherently variable, jurisdiction-dependent, or evolve faster than relational schemas can accommodate.

Corporate governance is a domain with two distinct halves. One half is remarkably stable: boards have members, meetings have agendas, agendas have items, items produce votes, votes produce resolutions. These structures are universal across every governance context from a Fortune 500 public company to a community nonprofit. This half belongs in normalized relational tables with full referential integrity.

The other half is wildly variable: D&O questionnaires differ by jurisdiction, by organisation type, and by year. Compliance filing requirements vary across 200+ jurisdictions. Insider trading rules differ between SEC (US), FCA (UK), SEBI (India), and dozens of other regulators. Director skills taxonomies are organisation-specific. Board evaluation criteria are customisable. Document metadata varies by document type. This half belongs in JSONB columns, where schema flexibility enables the platform to serve diverse governance contexts without requiring database migrations for every new jurisdiction or questionnaire format.

---

## Design Principles

1. **Relational for structure, JSONB for variability**: Any data element that is queried frequently by exact value, participates in foreign key relationships, or is shared across contexts belongs in a typed column. Data that varies by context, is displayed but rarely filtered, or evolves independently of the application version belongs in JSONB.

2. **JSONB is not schema-less**: Every JSONB column has a documented JSON Schema that is validated at the application layer. The database does not enforce the schema, but the application does. This provides flexibility without chaos.

3. **Indexed JSONB paths**: Frequently queried JSONB fields get GIN indexes or expression indexes. The goal is to avoid full JSONB scans for common access patterns.

4. **Audit trail in relational tables**: The audit log remains fully relational and append-only. Audit data is too critical for compliance to rely on flexible schemas.

5. **Multi-tenancy via RLS**: Row-level security policies enforce `organisation_id` isolation at the database level, covering both relational and JSONB data.

---

## Complete Schema

### Organisation & Entity Tables (Relational Core + JSONB Extensions)

```sql
-- Organisation with jurisdiction-specific metadata in JSONB
CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(500) NOT NULL,
    legal_name          VARCHAR(500),
    lei                 CHAR(20),                             -- ISO 17442
    jurisdiction        CHAR(2) NOT NULL,                     -- ISO 3166-1
    incorporation_date  DATE,
    organisation_type   VARCHAR(50) NOT NULL
                        CHECK (organisation_type IN ('public_company','private_company','nonprofit','cooperative','government','trust','partnership')),
    fiscal_year_end     VARCHAR(5),
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','suspended','dissolved','merged')),
    timezone            VARCHAR(50) NOT NULL DEFAULT 'UTC',

    -- JSONB: Jurisdiction-specific registration details
    -- Varies by country: SEC CIK for US, Companies House number for UK, ACN for Australia, etc.
    registration_details JSONB NOT NULL DEFAULT '{}',
    /*
    Example for US public company:
    {
        "sec_cik": "0001234567",
        "sec_file_number": "001-12345",
        "sic_code": "7372",
        "state_of_incorporation": "DE",
        "irs_ein": "12-3456789",
        "stock_exchange": "NYSE",
        "ticker_symbol": "ACME",
        "fiscal_year_end_month": 12,
        "accelerated_filer_status": "large_accelerated"
    }

    Example for UK company:
    {
        "companies_house_number": "12345678",
        "company_type": "private-limited-shares",
        "sic_codes": ["62020", "62090"],
        "registered_office": "England and Wales",
        "vat_number": "GB123456789"
    }

    Example for Australian company:
    {
        "acn": "123 456 789",
        "abn": "12 345 678 901",
        "asic_registered": true,
        "company_class": "LMSH"
    }
    */

    -- JSONB: Custom metadata and branding
    metadata            JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "logo_url": "https://...",
        "website": "https://...",
        "industry": "Technology",
        "employee_count": 5000,
        "annual_revenue_usd": 500000000,
        "custom_fields": { ... }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

-- GIN index for searching within registration_details
CREATE INDEX idx_org_registration ON organisations USING gin(registration_details);
-- Expression index for common lookups
CREATE INDEX idx_org_sec_cik ON organisations ((registration_details->>'sec_cik')) WHERE registration_details->>'sec_cik' IS NOT NULL;
CREATE INDEX idx_org_status ON organisations(status) WHERE deleted_at IS NULL;

-- Entity relationships (fully relational — structure is universal)
CREATE TABLE entity_relationships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_entity_id    UUID NOT NULL REFERENCES organisations(id),
    child_entity_id     UUID NOT NULL REFERENCES organisations(id),
    relationship_type   VARCHAR(50) NOT NULL DEFAULT 'subsidiary'
                        CHECK (relationship_type IN ('subsidiary','branch','joint_venture','affiliate','division')),
    ownership_pct       DECIMAL(5,2),
    effective_date      DATE NOT NULL,
    end_date            DATE,

    -- JSONB: Relationship-specific details that vary by jurisdiction
    relationship_details JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "voting_rights_pct": 75.00,
        "consolidation_method": "full",
        "transfer_pricing_agreement": true,
        "intercompany_loan_balance_usd": 5000000,
        "regulatory_approvals": [
            { "regulator": "FTC", "approval_date": "2024-03-15", "conditions": "..." }
        ]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_no_self_parent CHECK (parent_entity_id != child_entity_id),
    CONSTRAINT uq_entity_rel UNIQUE (parent_entity_id, child_entity_id, effective_date)
);

CREATE INDEX idx_entity_rel_parent ON entity_relationships(parent_entity_id);
CREATE INDEX idx_entity_rel_child ON entity_relationships(child_entity_id);

-- Registered officers per entity (jurisdictional requirements vary)
CREATE TABLE entity_officers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID REFERENCES users(id),            -- NULL if officer is not a platform user
    full_name           VARCHAR(400) NOT NULL,
    officer_type        VARCHAR(50) NOT NULL
                        CHECK (officer_type IN ('director','secretary','ceo','cfo','coo','cto','general_counsel','treasurer','registered_agent','other')),
    appointed_date      DATE NOT NULL,
    resigned_date       DATE,
    is_current          BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: Jurisdiction-specific officer details
    officer_details     JSONB NOT NULL DEFAULT '{}',
    /*
    UK example:
    {
        "nationality": "British",
        "country_of_residence": "GB",
        "occupation": "Company Director",
        "date_of_birth_month_year": "1975-06",
        "service_address": { ... },
        "residential_address_same_as_service": true
    }

    US SEC Section 16 example:
    {
        "section_16_filer": true,
        "relationship_to_issuer": "officer",
        "officer_title": "Chief Financial Officer",
        "reporting_owner_cik": "0009876543"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_entity_officers_org ON entity_officers(organisation_id);
CREATE INDEX idx_entity_officers_user ON entity_officers(user_id) WHERE user_id IS NOT NULL;
```

### User & Access Control (Relational Core)

```sql
-- Users remain fully relational — universal structure
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(320) NOT NULL,
    first_name          VARCHAR(200) NOT NULL,
    last_name           VARCHAR(200) NOT NULL,
    title               VARCHAR(200),
    phone               VARCHAR(50),
    mobile              VARCHAR(50),
    timezone            VARCHAR(50) DEFAULT 'UTC',
    locale              VARCHAR(10) DEFAULT 'en',
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_method          VARCHAR(20),
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','invited','suspended','deactivated')),

    -- JSONB: User preferences and profile extensions
    profile             JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "bio": "...",
        "avatar_url": "https://...",
        "notification_preferences": {
            "email_digest": "daily",
            "push_enabled": true,
            "sms_for_urgent": false
        },
        "accessibility": {
            "high_contrast": false,
            "font_size": "medium"
        },
        "external_ids": {
            "linkedin": "https://linkedin.com/in/...",
            "bloomberg_id": "..."
        }
    }
    */

    password_hash       VARCHAR(500),
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT uq_users_email UNIQUE (email)
);

CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;

-- Organisation membership (relational — core access control)
CREATE TABLE organisation_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    role                VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (role IN ('owner','admin','company_secretary','director','officer','member','observer','auditor')),
    start_date          DATE NOT NULL,
    end_date            DATE,
    is_insider          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_org_member UNIQUE (organisation_id, user_id)
);

-- RBAC roles and permissions (relational — security-critical)
CREATE TABLE roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(200) NOT NULL,
    description         TEXT,
    is_system           BOOLEAN NOT NULL DEFAULT FALSE,

    -- JSONB: Granular permission matrix
    permission_matrix   JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "board": { "read": true, "write": false, "admin": false },
        "meeting": { "read": true, "write": true, "approve": false, "admin": false },
        "document": { "read": true, "write": true, "download": true, "delete": false },
        "entity": { "read": true, "write": false, "admin": false },
        "trading": { "read": false, "write": false, "approve": false },
        "messaging": { "read": true, "write": true },
        "audit": { "read": false }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_role_name UNIQUE (organisation_id, name)
);

CREATE TABLE user_roles (
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type          VARCHAR(50),
    scope_id            UUID,
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by          UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

### Board & Committee Tables (Relational Core)

```sql
CREATE TABLE boards (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(500) NOT NULL,
    board_type          VARCHAR(50) NOT NULL DEFAULT 'governing'
                        CHECK (board_type IN ('governing','advisory','subsidiary','joint_venture')),
    quorum_requirement  INTEGER,
    quorum_type         VARCHAR(20) DEFAULT 'count'
                        CHECK (quorum_type IN ('count','percentage')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',

    -- JSONB: Board-specific governance rules that vary by charter
    governance_rules    JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "term_length_years": 3,
        "max_consecutive_terms": 2,
        "mandatory_retirement_age": 75,
        "independence_requirement_pct": 66.67,
        "diversity_targets": {
            "gender_minority_pct": 30,
            "ethnic_diversity_target": true
        },
        "meeting_frequency": "quarterly",
        "minimum_meetings_per_year": 4,
        "proxy_voting_allowed": true,
        "circular_resolution_allowed": true,
        "supermajority_threshold_pct": 75.0,
        "chair_has_casting_vote": true,
        "video_attendance_counts_for_quorum": true
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_boards_org ON boards(organisation_id) WHERE deleted_at IS NULL;

CREATE TABLE board_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id            UUID NOT NULL REFERENCES boards(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    board_role          VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (board_role IN ('chair','vice_chair','lead_independent','member','secretary','observer')),
    independence_status VARCHAR(50) DEFAULT 'independent'
                        CHECK (independence_status IN ('independent','non_independent','executive','not_applicable')),
    appointed_date      DATE NOT NULL,
    term_expiry_date    DATE,
    resigned_date       DATE,
    appointment_type    VARCHAR(50),

    -- JSONB: Director-specific board membership metadata
    membership_details  JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "appointment_letter_document_id": "uuid",
        "compensation": {
            "annual_retainer_usd": 75000,
            "meeting_fee_usd": 2000,
            "committee_chair_fee_usd": 15000,
            "equity_grant_shares": 5000
        },
        "committees_served": ["audit", "compensation"],
        "meeting_attendance_pct": 92.5,
        "tenure_years": 4.5,
        "independent_until_date": "2027-06-30"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_board_member UNIQUE (board_id, user_id)
);

CREATE TABLE committees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id            UUID NOT NULL REFERENCES boards(id),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(500) NOT NULL,
    committee_type      VARCHAR(50) NOT NULL
                        CHECK (committee_type IN ('audit','compensation','nominating','governance','risk','executive','esg','technology','special','other')),
    quorum_requirement  INTEGER,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    charter_document_id UUID,

    -- JSONB: Committee-specific charter requirements
    charter_requirements JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "minimum_members": 3,
        "all_independent_required": true,
        "financial_expert_required": true,
        "meeting_frequency": "quarterly",
        "reports_to_board_frequency": "after_each_meeting",
        "authority_matrix": {
            "approve_audit_fees": true,
            "hire_external_auditor": true,
            "approve_related_party_transactions_up_to_usd": 500000
        },
        "regulatory_requirements": [
            { "regulation": "SOX Section 301", "requirement": "Independent audit committee" },
            { "regulation": "NYSE Rule 10A-3", "requirement": "Financial literacy requirement" }
        ]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE committee_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    committee_id        UUID NOT NULL REFERENCES committees(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    committee_role      VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (committee_role IN ('chair','vice_chair','member','secretary','observer')),
    appointed_date      DATE NOT NULL,
    term_expiry_date    DATE,
    resigned_date       DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_committee_member UNIQUE (committee_id, user_id)
);
```

### Meeting Management (Relational Core + JSONB for Templates)

```sql
CREATE TABLE meetings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    board_id            UUID REFERENCES boards(id),
    committee_id        UUID REFERENCES committees(id),
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL DEFAULT 'regular'
                        CHECK (meeting_type IN ('regular','special','annual','emergency','circular_resolution','strategy_offsite')),
    status              VARCHAR(50) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','scheduled','in_progress','adjourned','completed','cancelled')),
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    location            VARCHAR(500),
    virtual_meeting_url VARCHAR(1000),
    virtual_provider    VARCHAR(50),
    quorum_met          BOOLEAN,
    quorum_count        INTEGER,
    created_by          UUID NOT NULL REFERENCES users(id),

    -- JSONB: Meeting-specific configuration and logistics
    meeting_config      JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "timezone": "America/New_York",
        "catering_notes": "Lunch service 12:00-13:00",
        "dial_in_number": "+1-555-123-4567",
        "dial_in_pin": "987654",
        "recording_permitted": true,
        "confidentiality_level": "board_only",
        "pre_read_deadline": "2026-06-01T17:00:00Z",
        "rsvp_deadline": "2026-06-05T17:00:00Z",
        "parking_instructions": "Visitor parking in Lot B",
        "nda_required": false,
        "remote_attendance_tech": {
            "platform": "zoom",
            "breakout_rooms": true,
            "interpretation": ["Spanish", "French"]
        }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT chk_meeting_context CHECK (board_id IS NOT NULL OR committee_id IS NOT NULL)
);

CREATE INDEX idx_meetings_org ON meetings(organisation_id, scheduled_start) WHERE deleted_at IS NULL;
CREATE INDEX idx_meetings_board ON meetings(board_id, scheduled_start) WHERE deleted_at IS NULL;
CREATE INDEX idx_meetings_status ON meetings(status) WHERE deleted_at IS NULL;

-- Meeting attendees (relational — core governance record)
CREATE TABLE meeting_attendees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    attendance_status   VARCHAR(30) NOT NULL DEFAULT 'invited'
                        CHECK (attendance_status IN ('invited','confirmed','declined','attended','absent','excused','proxy')),
    attendee_role       VARCHAR(50) DEFAULT 'member',
    proxy_for_user_id   UUID REFERENCES users(id),
    joined_at           TIMESTAMPTZ,
    left_at             TIMESTAMPTZ,
    rsvp_at             TIMESTAMPTZ,
    CONSTRAINT uq_meeting_attendee UNIQUE (meeting_id, user_id)
);

-- Agenda items (relational core)
CREATE TABLE agenda_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    parent_item_id      UUID REFERENCES agenda_items(id),
    sort_order          INTEGER NOT NULL,
    item_number         VARCHAR(20),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    item_type           VARCHAR(50) NOT NULL DEFAULT 'discussion'
                        CHECK (item_type IN ('procedural','information','discussion','decision','vote','presentation','consent','other')),
    presenter_user_id   UUID REFERENCES users(id),
    allocated_minutes   INTEGER,
    is_consent_item     BOOLEAN NOT NULL DEFAULT FALSE,
    is_confidential     BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','in_progress','completed','deferred','withdrawn')),

    -- JSONB: Item-specific presentation data and notes
    item_data           JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "speaking_notes": "...",
        "key_decision_required": true,
        "risk_level": "high",
        "financial_impact_usd": 5000000,
        "related_resolution_numbers": ["RES-2025-042"],
        "external_advisors_present": ["Jane Smith, Partner, Law Firm LLP"],
        "ai_summary": "This item concerns the proposed acquisition of..."
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agenda_items_meeting ON agenda_items(meeting_id, sort_order);

-- Meeting minutes (relational core with JSONB for structured content)
CREATE TABLE meeting_minutes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    version             INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','ai_generated','review','approved','signed')),
    content             TEXT NOT NULL,
    content_format      VARCHAR(20) NOT NULL DEFAULT 'markdown',
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,

    -- JSONB: Structured minutes metadata for AI and compliance
    minutes_metadata    JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "ai_model_used": "gpt-4o",
        "ai_confidence_score": 0.87,
        "source_materials": ["transcript", "agenda", "notes"],
        "transcript_id": "uuid",
        "key_decisions": [
            { "item": "3.1", "decision": "Approved Q1 budget of $12M", "vote_result": "7-0-1" }
        ],
        "attendee_summary": {
            "present": 8,
            "absent": 1,
            "by_proxy": 1
        },
        "meeting_duration_minutes": 145,
        "executive_session_held": true,
        "executive_session_duration_minutes": 30
    }
    */

    approved_at         TIMESTAMPTZ,
    approved_by         UUID REFERENCES users(id),
    signed_document_id  UUID,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_minutes_version UNIQUE (meeting_id, version)
);

-- Meeting templates with JSONB for flexible template definitions
CREATE TABLE meeting_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(300) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL,
    description         TEXT,

    -- JSONB: Full template definition
    template_definition JSONB NOT NULL,
    /*
    {
        "default_duration_minutes": 120,
        "default_agenda_items": [
            { "title": "Call to Order", "type": "procedural", "minutes": 5 },
            { "title": "Approval of Previous Minutes", "type": "consent", "minutes": 5 },
            { "title": "Chair's Report", "type": "information", "minutes": 15 },
            { "title": "Financial Report", "type": "presentation", "minutes": 20 },
            { "title": "Committee Reports", "type": "information", "minutes": 30, "sub_items": [
                { "title": "Audit Committee", "type": "information", "minutes": 10 },
                { "title": "Compensation Committee", "type": "information", "minutes": 10 },
                { "title": "Governance Committee", "type": "information", "minutes": 10 }
            ]},
            { "title": "New Business", "type": "discussion", "minutes": 30 },
            { "title": "Executive Session", "type": "confidential", "minutes": 30 },
            { "title": "Adjournment", "type": "procedural", "minutes": 5 }
        ],
        "required_attendees_by_role": ["chair", "secretary"],
        "pre_read_deadline_days_before": 5,
        "minutes_template": "# Minutes of {{meeting_type}} Meeting\n\n..."
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Voting & Resolutions (Relational Core)

```sql
-- Votes remain fully relational — critical governance records
CREATE TABLE votes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    agenda_item_id      UUID REFERENCES agenda_items(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    vote_type           VARCHAR(50) NOT NULL DEFAULT 'simple_majority'
                        CHECK (vote_type IN ('simple_majority','supermajority','unanimous','plurality','show_of_hands','roll_call','consent')),
    required_threshold  DECIMAL(5,2),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','open','closed','passed','failed','withdrawn','tabled')),
    opened_at           TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    moved_by            UUID REFERENCES users(id),
    seconded_by         UUID REFERENCES users(id),
    votes_for           INTEGER DEFAULT 0,
    votes_against       INTEGER DEFAULT 0,
    votes_abstain       INTEGER DEFAULT 0,
    result_notes        TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE vote_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vote_id             UUID NOT NULL REFERENCES votes(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    vote_cast           VARCHAR(20) NOT NULL
                        CHECK (vote_cast IN ('for','against','abstain','recuse')),
    voted_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes               TEXT,
    CONSTRAINT uq_vote_record UNIQUE (vote_id, user_id)
);

CREATE TABLE resolutions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    meeting_id          UUID REFERENCES meetings(id),
    vote_id             UUID REFERENCES votes(id),
    resolution_number   VARCHAR(50) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    body                TEXT NOT NULL,
    resolution_type     VARCHAR(50) NOT NULL DEFAULT 'ordinary'
                        CHECK (resolution_type IN ('ordinary','special','written','circular','consent','emergency')),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','proposed','passed','rejected','superseded','rescinded')),
    effective_date      DATE,
    expiry_date         DATE,
    supersedes_id       UUID REFERENCES resolutions(id),
    passed_at           TIMESTAMPTZ,
    signed_document_id  UUID,

    -- JSONB: Resolution-specific metadata varying by type
    resolution_metadata JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "regulatory_filing_required": true,
        "filing_type": "8-K",
        "filing_deadline": "2026-06-20",
        "material_event": true,
        "related_contracts": ["uuid1", "uuid2"],
        "legal_review_by": "General Counsel",
        "legal_review_date": "2026-06-08",
        "delegation_of_authority": {
            "delegated_to": "CEO",
            "authority": "Execute agreement on terms substantially as presented",
            "limit_usd": 10000000
        }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_resolution_number UNIQUE (organisation_id, resolution_number)
);

-- Action items (relational core)
CREATE TABLE action_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    meeting_id          UUID REFERENCES meetings(id),
    agenda_item_id      UUID REFERENCES agenda_items(id),
    resolution_id       UUID REFERENCES resolutions(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    assigned_to         UUID NOT NULL REFERENCES users(id),
    assigned_by         UUID REFERENCES users(id),
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium'
                        CHECK (priority IN ('critical','high','medium','low')),
    status              VARCHAR(30) NOT NULL DEFAULT 'open'
                        CHECK (status IN ('open','in_progress','completed','deferred','cancelled')),
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    completed_notes     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Document Management (Relational + JSONB Metadata)

```sql
CREATE TABLE document_folders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    parent_folder_id    UUID REFERENCES document_folders(id),
    name                VARCHAR(300) NOT NULL,
    folder_path         VARCHAR(2000) NOT NULL,
    board_id            UUID REFERENCES boards(id),
    committee_id        UUID REFERENCES committees(id),
    classification      VARCHAR(30) NOT NULL DEFAULT 'confidential',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT uq_folder_path UNIQUE (organisation_id, folder_path)
);

CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    folder_id           UUID REFERENCES document_folders(id),
    title               VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50) NOT NULL DEFAULT 'general'
                        CHECK (document_type IN (
                            'board_pack','agenda','minutes','resolution','charter','bylaws',
                            'policy','contract','report','presentation','financial','legal',
                            'compliance','questionnaire','disclosure','certificate','general'
                        )),
    mime_type           VARCHAR(200) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    storage_path        VARCHAR(1000) NOT NULL,
    checksum_sha256     CHAR(64) NOT NULL,
    encryption_key_id   VARCHAR(200),
    is_encrypted        BOOLEAN NOT NULL DEFAULT TRUE,
    version             INTEGER NOT NULL DEFAULT 1,
    parent_document_id  UUID REFERENCES documents(id),
    is_latest           BOOLEAN NOT NULL DEFAULT TRUE,
    classification      VARCHAR(30) NOT NULL DEFAULT 'confidential',
    retention_until     DATE,
    uploaded_by         UUID NOT NULL REFERENCES users(id),

    -- JSONB: Document-type-specific metadata
    doc_metadata        JSONB NOT NULL DEFAULT '{}',
    /*
    For a contract:
    {
        "contract_type": "service_agreement",
        "counterparty": "Law Firm LLP",
        "effective_date": "2026-01-01",
        "expiry_date": "2028-12-31",
        "renewal_type": "auto_renew",
        "annual_value_usd": 250000,
        "key_terms": ["exclusivity", "non-compete"],
        "governing_law": "State of New York",
        "signatories": ["CEO", "General Counsel"]
    }

    For a financial report:
    {
        "period": "Q1 2026",
        "report_type": "quarterly",
        "audited": false,
        "preparer": "CFO Office",
        "currency": "USD",
        "key_metrics": {
            "revenue": 125000000,
            "net_income": 15000000,
            "eps": 2.35
        }
    }

    For a policy document:
    {
        "policy_category": "information_security",
        "version": "3.1",
        "approved_by_board": true,
        "approval_date": "2026-03-15",
        "next_review_date": "2027-03-15",
        "applicable_to": ["all_employees", "contractors"],
        "regulatory_basis": ["ISO 27001", "SOC 2"]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_documents_org ON documents(organisation_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_documents_type ON documents(document_type) WHERE deleted_at IS NULL;
CREATE INDEX idx_documents_metadata ON documents USING gin(doc_metadata);

-- Full-text search across documents
ALTER TABLE documents ADD COLUMN search_vector TSVECTOR;
CREATE INDEX idx_documents_search ON documents USING gin(search_vector);
```

### Insider Trading Compliance (Relational + JSONB for Jurisdiction Variations)

```sql
CREATE TABLE trading_windows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(300) NOT NULL,
    window_type         VARCHAR(50) NOT NULL DEFAULT 'quarterly_blackout'
                        CHECK (window_type IN ('quarterly_blackout','event_specific','annual_meeting','ipo_lockup','custom')),
    status              VARCHAR(20) NOT NULL DEFAULT 'open'
                        CHECK (status IN ('open','closed','pending')),
    blackout_start      TIMESTAMPTZ NOT NULL,
    blackout_end        TIMESTAMPTZ,
    reason              TEXT,
    applies_to_all      BOOLEAN NOT NULL DEFAULT TRUE,
    created_by          UUID NOT NULL REFERENCES users(id),

    -- JSONB: Jurisdiction-specific trading rules
    trading_rules       JSONB NOT NULL DEFAULT '{}',
    /*
    US SEC rules:
    {
        "regulation": "SEC Rule 10b-5",
        "cooling_off_period_days": 2,
        "pre_announcement_blackout_days": 14,
        "post_announcement_open_days": 3,
        "applies_to_derivatives": true,
        "gift_transactions_restricted": true,
        "rule_10b5_1_plan_exempt": true
    }

    UK FCA/MAR rules:
    {
        "regulation": "UK MAR",
        "closed_period_days": 30,
        "applies_to_pdmr": true,
        "applies_to_pca": true,
        "notification_requirement_days": 3,
        "de_minimis_threshold_eur": 20000
    }

    SEBI (India) rules:
    {
        "regulation": "SEBI PIT 2015",
        "trading_plan_required": true,
        "contra_trade_restriction_months": 6,
        "sdd_required": true,
        "upsi_tracking": true
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE insider_designations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    insider_type        VARCHAR(50) NOT NULL
                        CHECK (insider_type IN ('section_16_officer','section_16_director','10pct_owner','designated_insider','pdmr','connected_person')),
    designation_date    DATE NOT NULL,
    revocation_date     DATE,

    -- JSONB: Jurisdiction-specific insider details
    insider_details     JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "sec_filing_status": "form3_filed",
        "initial_filing_date": "2024-01-15",
        "reporting_owner_cik": "0009876543",
        "relationship_codes": ["D", "O"],
        "officer_title": "Chief Financial Officer",
        "beneficial_ownership_report_filed": true,
        "power_of_attorney_on_file": true
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_insider_designation UNIQUE (organisation_id, user_id, insider_type)
);

CREATE TABLE preclearance_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    requester_user_id   UUID NOT NULL REFERENCES users(id),
    insider_designation_id UUID NOT NULL REFERENCES insider_designations(id),
    security_type       VARCHAR(50) NOT NULL,
    security_name       VARCHAR(300) NOT NULL,
    ticker_symbol       VARCHAR(20),
    transaction_type    VARCHAR(50) NOT NULL
                        CHECK (transaction_type IN ('purchase','sale','exercise','gift','conversion','10b5_1_plan','other')),
    requested_shares    DECIMAL(18,4),
    estimated_price     DECIMAL(18,4),
    estimated_value     DECIMAL(18,2),
    transaction_date    DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','approved','denied','expired','cancelled','executed')),
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    review_notes        TEXT,
    approval_expires_at TIMESTAMPTZ,
    executed_at         TIMESTAMPTZ,
    actual_shares       DECIMAL(18,4),
    actual_price        DECIMAL(18,4),

    -- JSONB: Transaction-specific details
    transaction_details JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "broker_name": "Morgan Stanley",
        "broker_account": "XXX-1234",
        "brokerage_firm_dtc": "0015",
        "derivative_details": {
            "underlying_security": "Common Stock",
            "exercise_price": 45.00,
            "expiration_date": "2028-03-15",
            "conversion_ratio": 1.0
        },
        "10b5_1_plan": {
            "plan_adoption_date": "2025-12-01",
            "plan_modification_date": null,
            "plan_termination_date": "2026-12-01",
            "cooling_off_period_met": true
        },
        "form_4_filing": {
            "filed": true,
            "filing_date": "2026-06-12",
            "accession_number": "0001234567-26-001234"
        }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_preclearance_org ON preclearance_requests(organisation_id);
CREATE INDEX idx_preclearance_status ON preclearance_requests(status);
CREATE INDEX idx_preclearance_requester ON preclearance_requests(requester_user_id);
```

### Questionnaires & Disclosures (JSONB-Driven)

```sql
-- Questionnaires are inherently flexible — JSONB is the natural fit
CREATE TABLE questionnaires (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    title               VARCHAR(500) NOT NULL,
    questionnaire_type  VARCHAR(50) NOT NULL
                        CHECK (questionnaire_type IN ('dno_annual','dno_onboarding','conflict_of_interest','independence','suitability','board_evaluation','custom')),
    description         TEXT,
    due_date            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','active','closed','archived')),

    -- JSONB: Complete questionnaire definition — sections, questions, validation rules
    form_definition     JSONB NOT NULL,
    /*
    {
        "version": "2026.1",
        "sections": [
            {
                "id": "s1",
                "title": "Personal Information",
                "questions": [
                    {
                        "id": "q1",
                        "text": "Have you served as an officer or director of any other public company in the past five years?",
                        "type": "yes_no",
                        "required": true,
                        "follow_up_if_yes": {
                            "id": "q1a",
                            "text": "Please list all such positions:",
                            "type": "table",
                            "columns": ["Company Name", "Position", "Start Date", "End Date"],
                            "required": true
                        }
                    },
                    {
                        "id": "q2",
                        "text": "Do you have any family relationships with any officer or director of the company?",
                        "type": "yes_no",
                        "required": true,
                        "regulatory_reference": "Item 401(d) of Regulation S-K"
                    }
                ]
            },
            {
                "id": "s2",
                "title": "Financial Interests",
                "questions": [...]
            }
        ],
        "scoring": null,
        "auto_flag_rules": [
            { "condition": "q1 == 'yes' AND q1a.rows > 3", "flag": "multiple_board_service", "severity": "review" }
        ]
    }
    */

    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE questionnaire_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    questionnaire_id    UUID NOT NULL REFERENCES questionnaires(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'in_progress'
                        CHECK (status IN ('in_progress','submitted','reviewed','approved','flagged')),

    -- JSONB: Complete response data matching the form_definition structure
    responses           JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "s1": {
            "q1": { "answer": "yes", "q1a": { "rows": [
                { "Company Name": "Acme Corp", "Position": "Director", "Start Date": "2022-01", "End Date": "present" }
            ]}},
            "q2": { "answer": "no" }
        },
        "s2": { ... }
    }
    */

    -- JSONB: Automated flags and reviewer notes
    review_data         JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "auto_flags": [
            { "rule": "multiple_board_service", "severity": "review", "triggered_by": "q1" }
        ],
        "reviewer_notes": "No concerns identified. Multiple board service is within policy limits.",
        "follow_up_required": false
    }
    */

    submitted_at        TIMESTAMPTZ,
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_questionnaire_response UNIQUE (questionnaire_id, user_id)
);

-- Interest disclosures (relational core with JSONB for variable detail)
CREATE TABLE interest_disclosures (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    disclosure_type     VARCHAR(50) NOT NULL
                        CHECK (disclosure_type IN ('financial_interest','employment','board_membership','family_relationship','contractual','property','other')),
    entity_name         VARCHAR(500),
    description         TEXT NOT NULL,
    monetary_value      DECIMAL(18,2),
    currency            CHAR(3),
    is_material         BOOLEAN,
    disclosed_date      DATE NOT NULL,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','expired','withdrawn','superseded')),

    -- JSONB: Disclosure-type-specific details
    disclosure_details  JSONB NOT NULL DEFAULT '{}',
    /*
    For board_membership:
    {
        "position": "Non-Executive Director",
        "company_type": "public",
        "industry": "Technology",
        "compensation_annual_usd": 50000,
        "committee_memberships": ["Audit", "Technology"],
        "potential_conflict_areas": ["competing product line"]
    }

    For financial_interest:
    {
        "instrument_type": "common_stock",
        "shares_held": 10000,
        "acquisition_date": "2023-06-15",
        "acquisition_method": "market_purchase",
        "percentage_ownership": 0.05
    }
    */

    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_disclosures_user ON interest_disclosures(user_id, status);
CREATE INDEX idx_disclosures_org ON interest_disclosures(organisation_id);
```

### Board Effectiveness (Relational + JSONB Skills Model)

```sql
-- Skills taxonomy with flexible scoring in JSONB
CREATE TABLE skills_taxonomy (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    category            VARCHAR(200) NOT NULL,
    skill_name          VARCHAR(300) NOT NULL,
    is_critical         BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order          INTEGER NOT NULL DEFAULT 0,

    -- JSONB: Skill definition metadata
    skill_definition    JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "description": "Experience leading or overseeing M&A transactions valued at $100M+",
        "proficiency_scale": {
            "1": "Awareness only",
            "2": "Working knowledge",
            "3": "Applied experience",
            "4": "Deep expertise",
            "5": "Thought leadership"
        },
        "assessment_questions": [
            "How many M&A transactions have you been involved in?",
            "What was the largest transaction value?"
        ],
        "regulatory_requirement": false,
        "succession_priority": "high"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_skill UNIQUE (organisation_id, category, skill_name)
);

CREATE TABLE director_skills (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    skill_id            UUID NOT NULL REFERENCES skills_taxonomy(id),
    proficiency_level   INTEGER NOT NULL DEFAULT 2
                        CHECK (proficiency_level BETWEEN 1 AND 5),
    years_experience    INTEGER,
    self_assessed       BOOLEAN NOT NULL DEFAULT TRUE,
    assessed_by         UUID REFERENCES users(id),
    assessed_at         TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_director_skill UNIQUE (user_id, skill_id)
);

-- Board evaluations with JSONB for flexible evaluation criteria
CREATE TABLE board_evaluations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    board_id            UUID NOT NULL REFERENCES boards(id),
    evaluation_year     INTEGER NOT NULL,
    evaluation_type     VARCHAR(50) NOT NULL DEFAULT 'self_assessment',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    overall_rating      DECIMAL(3,1),

    -- JSONB: Flexible evaluation criteria and results
    evaluation_framework JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "criteria": [
            { "id": "c1", "name": "Strategy oversight", "weight": 0.20, "avg_score": 4.2 },
            { "id": "c2", "name": "Risk management", "weight": 0.15, "avg_score": 3.8 },
            { "id": "c3", "name": "CEO oversight", "weight": 0.15, "avg_score": 4.5 },
            { "id": "c4", "name": "Financial oversight", "weight": 0.15, "avg_score": 4.0 },
            { "id": "c5", "name": "Board dynamics", "weight": 0.10, "avg_score": 3.5 },
            { "id": "c6", "name": "Meeting effectiveness", "weight": 0.10, "avg_score": 4.1 },
            { "id": "c7", "name": "Stakeholder engagement", "weight": 0.10, "avg_score": 3.9 },
            { "id": "c8", "name": "ESG/Sustainability", "weight": 0.05, "avg_score": 3.3 }
        ],
        "recommendations": [
            "Increase board focus on ESG and sustainability oversight",
            "Improve board dynamics through facilitated discussion sessions"
        ],
        "year_over_year_trend": "improving",
        "external_evaluator": null
    }
    */

    report_document_id  UUID REFERENCES documents(id),
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_board_eval UNIQUE (board_id, evaluation_year, evaluation_type)
);
```

### Compliance Calendar & Filing Tracking

```sql
CREATE TABLE filing_deadlines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    entity_id           UUID REFERENCES organisations(id),
    jurisdiction        CHAR(2) NOT NULL,
    filing_type         VARCHAR(100) NOT NULL,
    regulatory_body     VARCHAR(200) NOT NULL,
    due_date            DATE NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'upcoming'
                        CHECK (status IN ('upcoming','in_progress','filed','overdue','waived','not_applicable')),
    filed_at            TIMESTAMPTZ,
    filed_by            UUID REFERENCES users(id),
    filing_reference    VARCHAR(200),
    recurrence          VARCHAR(30),

    -- JSONB: Jurisdiction-specific filing requirements
    filing_requirements JSONB NOT NULL DEFAULT '{}',
    /*
    US SEC filing:
    {
        "form_type": "10-K",
        "filing_system": "EDGAR",
        "edgar_next_required": true,
        "xbrl_tagging_required": true,
        "inline_xbrl": true,
        "acceleration_status": "large_accelerated",
        "filing_deadline_rule": "60_days_after_fiscal_year_end",
        "late_filing_notification_required": true,
        "related_forms": ["10-K/A", "NT 10-K"]
    }

    UK Companies House:
    {
        "form_type": "confirmation_statement",
        "filing_system": "Companies House WebFiling",
        "authentication_code_required": true,
        "fee_gbp": 13,
        "penalty_for_late_filing": true,
        "penalty_amount_gbp": 150
    }
    */

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_filing_deadlines_due ON filing_deadlines(due_date, status);
CREATE INDEX idx_filing_deadlines_org ON filing_deadlines(organisation_id);
```

### Audit Log (Fully Relational — Compliance Critical)

```sql
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    session_id          UUID,
    action              VARCHAR(50) NOT NULL,
    resource_type       VARCHAR(100) NOT NULL,
    resource_id         UUID,
    resource_name       VARCHAR(500),
    old_values          JSONB,
    new_values          JSONB,
    ip_address          INET,
    user_agent          VARCHAR(500),
    geo_location        VARCHAR(200),
    severity            VARCHAR(20) NOT NULL DEFAULT 'info'
                        CHECK (severity IN ('info','warning','critical','security')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue for each month

CREATE INDEX idx_audit_org_date ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Pros and Cons

### Pros

1. **Best of both worlds**: Stable governance structures (boards, meetings, votes, resolutions) get full referential integrity, while variable data (jurisdiction-specific registration, questionnaire definitions, filing requirements, document metadata) gets the flexibility it needs. This reflects how the domain actually works — some things are universal, some things vary by context.

2. **Single database technology**: Everything runs on PostgreSQL. No additional database servers to operate, monitor, back up, or secure. The operational complexity of running a board portal (which must be highly available and secure) stays manageable.

3. **Jurisdiction-agnostic by design**: Adding support for a new jurisdiction (e.g., Singapore ACRA filing requirements, or German BaFin insider trading rules) requires only adding new JSONB content — no schema migrations, no deployment, no downtime. This is critical for a platform targeting multinational organisations.

4. **Queryable JSONB**: PostgreSQL JSONB is not a black box. You can query, index, and aggregate JSONB data using SQL operators (`->`, `->>`, `@>`, `?`). GIN indexes on JSONB columns enable performant queries like "find all entities with SEC CIK matching X" or "find all filings requiring XBRL tagging."

5. **Schema evolution without migration pain**: When questionnaire formats change, filing requirements evolve, or new metadata fields are needed, the JSONB columns accommodate the change immediately. Application-level validation ensures data quality. Old data with the old schema continues to work — no backfill migrations needed.

6. **Natural fit for form builders**: Questionnaires, board evaluations, and D&O disclosures are fundamentally dynamic forms. JSONB is the natural storage for form definitions and responses, enabling a generic form builder UI without hardcoded database columns for each question.

### Cons

1. **Validation burden shifts to application**: JSONB columns do not enforce constraints at the database level. A bug in the application could write malformed JSONB data that violates the expected schema. This requires disciplined application-layer validation (JSON Schema, Zod, or equivalent) and comprehensive test coverage.

2. **Referential integrity gaps in JSONB**: If a JSONB field references a document ID or user ID, PostgreSQL cannot enforce that reference. The application must ensure referential integrity for any cross-references stored in JSONB. For a compliance-critical system, this is a real risk that requires careful code review and testing.

3. **Query complexity for JSONB**: While PostgreSQL JSONB queries are powerful, they are less intuitive than standard SQL. Queries like `WHERE doc_metadata->>'contract_type' = 'service_agreement' AND (doc_metadata->'key_terms')::jsonb ? 'exclusivity'` are harder to write, read, and maintain than column-based WHERE clauses.

4. **JSONB indexing is coarser**: GIN indexes on JSONB cover all paths. You cannot create a unique constraint on a specific JSONB path (e.g., "registration_details->>'sec_cik' must be unique"). Expression indexes can partially address this but add maintenance overhead.

5. **Reporting across JSONB is harder**: Generating cross-organisation reports that aggregate data stored in JSONB (e.g., "total insider holdings across all subsidiaries by security type") requires JSONB extraction functions that are slower than column-based aggregation.

6. **Risk of JSONB as a dumping ground**: Without discipline, JSONB columns can become unstructured catch-alls where developers store data that should have its own table. The team needs clear guidelines about what belongs in JSONB versus relational columns.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|----------------|
| **Database** | PostgreSQL 16+ (JSONB performance improvements in recent versions) |
| **ORM** | Prisma (strong JSONB support) or Drizzle ORM |
| **JSONB Validation** | Zod (TypeScript) or Pydantic (Python) for application-layer schema enforcement |
| **JSON Schema Registry** | Maintain versioned JSON Schemas for every JSONB column in a shared registry |
| **JSONB Indexing** | GIN indexes on frequently queried JSONB columns; expression indexes for specific paths |
| **Full-text search** | PostgreSQL tsvector for document and resolution search; extract JSONB text into tsvector |
| **Migration tool** | Prisma Migrate or Alembic for relational schema; application-level for JSONB schema evolution |
| **Monitoring** | Track JSONB column sizes, GIN index bloat, and JSONB query performance |
| **Testing** | Property-based testing (fast-check, Hypothesis) for JSONB schema validation |

---

## Migration & Scaling Considerations

### Initial Deployment

- Start with a single PostgreSQL 16 instance
- Create GIN indexes on all JSONB columns from day one
- Document JSON Schemas for every JSONB column in a shared schema registry
- Implement application-layer JSONB validation using Zod or Pydantic
- Set up audit_log as a partitioned table from the start

### Growth Phase (100-1,000 organisations)

- Add expression indexes on frequently filtered JSONB paths (e.g., `registration_details->>'sec_cik'`)
- Monitor JSONB column sizes — if any column consistently exceeds 100KB, consider extracting heavily-used sub-objects into relational columns
- Add read replicas for reporting workloads
- Implement JSONB schema versioning: include a `"_schema_version": 2` field in JSONB objects to support backward-compatible evolution

### Scale Phase (1,000+ organisations)

- Extract high-volume JSONB analytics into materialised views that flatten JSONB into columnar format for reporting
- Consider PostgreSQL Citus extension for horizontal scaling
- Archive audit_log partitions older than 2 years to cold storage
- If document metadata querying becomes a bottleneck, extract `doc_metadata` into a dedicated Elasticsearch cluster

### JSONB Schema Evolution Strategy

1. Every JSONB column gets a versioned JSON Schema stored in the application's schema registry
2. Application code always writes the latest schema version with a `_schema_version` field
3. Application code reads and handles all known schema versions (backward compatibility)
4. Periodic background jobs can migrate old JSONB documents to the latest schema version (optional, for reporting consistency)
5. Never rely on JSONB structure alone for compliance-critical decisions — extract key compliance fields into relational columns as they stabilise
