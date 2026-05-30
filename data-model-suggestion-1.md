# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Corporate Governance Platform (Candidate #435)
> Generated: 2026-05-25

## Overview

This model implements a fully normalized relational schema in PostgreSQL, designed to support the complete corporate governance lifecycle: board portal, meeting management, entity management, insider trading compliance, document governance, and board effectiveness analytics. Every table enforces referential integrity, supports audit requirements, and enables complex multi-entity, multi-board queries through well-defined foreign key relationships.

PostgreSQL is the natural choice here because corporate governance data is inherently relational: entities own subsidiaries, directors sit on boards, boards hold meetings, meetings produce resolutions, resolutions create action items, insiders request trade pre-clearances against trading windows, and every one of these relationships must be queryable, auditable, and enforceable at the database level.

---

## Schema Design Principles

1. **Multi-tenancy via `organisation_id`**: Every core table carries an `organisation_id` foreign key to support SaaS multi-tenancy with row-level security (RLS). This enables a single database to serve multiple organisations while maintaining strict data isolation.

2. **Temporal tracking**: All mutable entities include `created_at`, `updated_at`, and where appropriate `deleted_at` (soft delete) timestamps. Governance data must never be physically deleted — it must remain audit-ready indefinitely.

3. **Comprehensive audit trail**: A dedicated `audit_log` table captures every state change with before/after snapshots, user attribution, and IP address.

4. **ISO standard alignment**: Entity identifiers follow ISO 17442 (LEI), jurisdictions use ISO 3166-1 country codes, and timestamps are stored as `TIMESTAMPTZ` (UTC).

5. **Role-based access control**: A fine-grained RBAC model with permissions at the organisation, board, committee, meeting, and document level.

---

## Complete Schema

### Core Organisation & Tenant Tables

```sql
-- Multi-tenant organisation root
CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(500) NOT NULL,
    legal_name          VARCHAR(500),
    lei                 CHAR(20),                           -- ISO 17442 Legal Entity Identifier
    jurisdiction        CHAR(2) NOT NULL,                   -- ISO 3166-1 alpha-2
    incorporation_date  DATE,
    organisation_type   VARCHAR(50) NOT NULL                -- 'public_company', 'private_company', 'nonprofit', 'cooperative', 'government'
                        CHECK (organisation_type IN ('public_company','private_company','nonprofit','cooperative','government')),
    fiscal_year_end     VARCHAR(5),                          -- 'MM-DD' format
    sec_cik             VARCHAR(20),                         -- SEC Central Index Key for US public companies
    website             VARCHAR(500),
    logo_url            VARCHAR(1000),
    timezone            VARCHAR(50) NOT NULL DEFAULT 'UTC',
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','suspended','dissolved','merged')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT uq_organisations_lei UNIQUE (lei)
);

CREATE INDEX idx_organisations_status ON organisations(status) WHERE deleted_at IS NULL;

-- Subsidiary / parent-child entity relationships
CREATE TABLE entity_relationships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_entity_id    UUID NOT NULL REFERENCES organisations(id),
    child_entity_id     UUID NOT NULL REFERENCES organisations(id),
    relationship_type   VARCHAR(50) NOT NULL DEFAULT 'subsidiary'
                        CHECK (relationship_type IN ('subsidiary','branch','joint_venture','affiliate','division')),
    ownership_pct       DECIMAL(5,2),                        -- Ownership percentage (0.00 - 100.00)
    effective_date      DATE NOT NULL,
    end_date            DATE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_no_self_parent CHECK (parent_entity_id != child_entity_id),
    CONSTRAINT uq_entity_relationship UNIQUE (parent_entity_id, child_entity_id, effective_date)
);

CREATE INDEX idx_entity_rel_parent ON entity_relationships(parent_entity_id);
CREATE INDEX idx_entity_rel_child ON entity_relationships(child_entity_id);
```

### User & Authentication Tables

```sql
-- Platform users (directors, officers, admins, company secretaries)
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(320) NOT NULL,
    first_name          VARCHAR(200) NOT NULL,
    last_name           VARCHAR(200) NOT NULL,
    title               VARCHAR(200),                        -- e.g. 'Chairman', 'CEO', 'Company Secretary'
    phone               VARCHAR(50),
    mobile              VARCHAR(50),
    bio                 TEXT,
    avatar_url          VARCHAR(1000),
    timezone            VARCHAR(50) DEFAULT 'UTC',
    locale              VARCHAR(10) DEFAULT 'en',
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_method          VARCHAR(20),                         -- 'totp', 'webauthn', 'sms'
    last_login_at       TIMESTAMPTZ,
    password_hash       VARCHAR(500),                        -- NULL if SSO-only
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','invited','suspended','deactivated')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT uq_users_email UNIQUE (email)
);

CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_status ON users(status) WHERE deleted_at IS NULL;

-- SSO identity provider links
CREATE TABLE user_sso_identities (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    provider            VARCHAR(50) NOT NULL,                -- 'saml', 'oidc'
    provider_name       VARCHAR(200) NOT NULL,               -- 'Okta', 'Azure AD', 'Google Workspace'
    external_id         VARCHAR(500) NOT NULL,                -- IdP subject identifier
    metadata            JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_sso_identity UNIQUE (provider, external_id)
);

-- Organisation membership for users
CREATE TABLE organisation_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    role                VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (role IN ('owner','admin','company_secretary','director','officer','member','observer','auditor')),
    start_date          DATE NOT NULL,
    end_date            DATE,
    is_insider          BOOLEAN NOT NULL DEFAULT FALSE,       -- Section 16 insider flag
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_org_member UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members(organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members(user_id);
```

### RBAC Permission Tables

```sql
-- Roles and permissions
CREATE TABLE roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(200) NOT NULL,
    description         TEXT,
    is_system           BOOLEAN NOT NULL DEFAULT FALSE,       -- System-defined vs custom
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_role_name UNIQUE (organisation_id, name)
);

CREATE TABLE permissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type       VARCHAR(50) NOT NULL,                 -- 'board', 'meeting', 'document', 'entity', 'trading'
    action              VARCHAR(50) NOT NULL,                 -- 'read', 'write', 'delete', 'approve', 'admin'
    description         TEXT,
    CONSTRAINT uq_permission UNIQUE (resource_type, action)
);

CREATE TABLE role_permissions (
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id       UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type          VARCHAR(50),                          -- 'board', 'committee', 'entity' (NULL = org-wide)
    scope_id            UUID,                                 -- ID of the specific board/committee/entity
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by          UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_type, ''), COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);
```

### Board & Committee Tables

```sql
-- Boards within an organisation
CREATE TABLE boards (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(500) NOT NULL,                -- 'Board of Directors', 'Advisory Board'
    board_type          VARCHAR(50) NOT NULL DEFAULT 'governing'
                        CHECK (board_type IN ('governing','advisory','subsidiary','joint_venture')),
    description         TEXT,
    max_members         INTEGER,
    quorum_requirement  INTEGER,                              -- Minimum members for quorum
    quorum_type         VARCHAR(20) DEFAULT 'count'           -- 'count' or 'percentage'
                        CHECK (quorum_type IN ('count','percentage')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','inactive','dissolved')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_boards_org ON boards(organisation_id) WHERE deleted_at IS NULL;

-- Board members (linking users to boards with specific board roles)
CREATE TABLE board_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id            UUID NOT NULL REFERENCES boards(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    board_role          VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (board_role IN ('chair','vice_chair','member','secretary','observer')),
    independence_status VARCHAR(50) DEFAULT 'independent'
                        CHECK (independence_status IN ('independent','non_independent','executive','not_applicable')),
    appointed_date      DATE NOT NULL,
    term_expiry_date    DATE,
    resigned_date       DATE,
    appointment_type    VARCHAR(50),                          -- 'elected', 'appointed', 'ex_officio'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_board_member UNIQUE (board_id, user_id)
);

-- Committees (sub-groups of boards)
CREATE TABLE committees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id            UUID NOT NULL REFERENCES boards(id),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(500) NOT NULL,                -- 'Audit Committee', 'Compensation Committee', 'Nominating Committee'
    committee_type      VARCHAR(50) NOT NULL
                        CHECK (committee_type IN ('audit','compensation','nominating','governance','risk','executive','special','other')),
    charter_document_id UUID,                                 -- FK to documents table (added after documents table)
    quorum_requirement  INTEGER,
    description         TEXT,
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','inactive','dissolved')),
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

### Meeting Management Tables

```sql
-- Meetings (board and committee meetings)
CREATE TABLE meetings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    board_id            UUID REFERENCES boards(id),
    committee_id        UUID REFERENCES committees(id),
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL DEFAULT 'regular'
                        CHECK (meeting_type IN ('regular','special','annual','emergency','circular_resolution')),
    status              VARCHAR(50) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','scheduled','in_progress','adjourned','completed','cancelled')),
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    location            VARCHAR(500),
    virtual_meeting_url VARCHAR(1000),                        -- Zoom/Teams link
    virtual_provider    VARCHAR(50),                          -- 'zoom', 'teams', 'webex'
    timezone            VARCHAR(50),
    quorum_met          BOOLEAN,
    quorum_count        INTEGER,
    template_id         UUID,                                 -- FK to meeting_templates
    notes               TEXT,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT chk_meeting_context CHECK (board_id IS NOT NULL OR committee_id IS NOT NULL)
);

CREATE INDEX idx_meetings_org ON meetings(organisation_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_meetings_board ON meetings(board_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_meetings_date ON meetings(scheduled_start) WHERE deleted_at IS NULL;
CREATE INDEX idx_meetings_status ON meetings(status) WHERE deleted_at IS NULL;

-- Meeting attendees with attendance tracking
CREATE TABLE meeting_attendees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    attendance_status   VARCHAR(30) NOT NULL DEFAULT 'invited'
                        CHECK (attendance_status IN ('invited','confirmed','declined','attended','absent','excused','proxy')),
    attendee_role       VARCHAR(50) DEFAULT 'member',         -- 'chair', 'secretary', 'member', 'presenter', 'observer', 'guest'
    proxy_for_user_id   UUID REFERENCES users(id),            -- If attending as proxy for another member
    joined_at           TIMESTAMPTZ,
    left_at             TIMESTAMPTZ,
    rsvp_at             TIMESTAMPTZ,
    notes               TEXT,
    CONSTRAINT uq_meeting_attendee UNIQUE (meeting_id, user_id)
);

-- Agenda items
CREATE TABLE agenda_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    parent_item_id      UUID REFERENCES agenda_items(id),     -- For nested/sub-items
    sort_order          INTEGER NOT NULL,
    item_number         VARCHAR(20),                          -- Display number: '1', '1.1', '2.a'
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    item_type           VARCHAR(50) NOT NULL DEFAULT 'discussion'
                        CHECK (item_type IN ('procedural','information','discussion','decision','vote','presentation','consent','other')),
    presenter_user_id   UUID REFERENCES users(id),
    allocated_minutes   INTEGER,                              -- Time allocation in minutes
    is_consent_item     BOOLEAN NOT NULL DEFAULT FALSE,       -- Part of consent agenda
    is_confidential     BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','in_progress','completed','deferred','withdrawn')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agenda_items_meeting ON agenda_items(meeting_id, sort_order);

-- Pre-read attachments linked to agenda items
CREATE TABLE agenda_attachments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agenda_item_id      UUID NOT NULL REFERENCES agenda_items(id) ON DELETE CASCADE,
    document_id         UUID NOT NULL,                        -- FK to documents table
    attachment_type     VARCHAR(50) NOT NULL DEFAULT 'pre_read'
                        CHECK (attachment_type IN ('pre_read','reference','presentation','handout','appendix')),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Meeting minutes
CREATE TABLE meeting_minutes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    version             INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','ai_generated','review','approved','signed')),
    content             TEXT NOT NULL,                         -- Rich text / Markdown content
    content_format      VARCHAR(20) NOT NULL DEFAULT 'markdown'
                        CHECK (content_format IN ('markdown','html','plain')),
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,
    approved_at         TIMESTAMPTZ,
    approved_by         UUID REFERENCES users(id),
    signed_document_id  UUID,                                 -- FK to documents (signed PDF)
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_minutes_version UNIQUE (meeting_id, version)
);
```

### Voting & Resolution Tables

```sql
-- Votes / motions tied to agenda items
CREATE TABLE votes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    agenda_item_id      UUID REFERENCES agenda_items(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    vote_type           VARCHAR(50) NOT NULL DEFAULT 'simple_majority'
                        CHECK (vote_type IN ('simple_majority','supermajority','unanimous','plurality','show_of_hands','roll_call','consent')),
    required_threshold  DECIMAL(5,2),                         -- e.g. 66.67 for supermajority
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

-- Individual vote records
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

-- Formal resolutions
CREATE TABLE resolutions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    meeting_id          UUID REFERENCES meetings(id),         -- NULL for written/circular resolutions
    vote_id             UUID REFERENCES votes(id),
    resolution_number   VARCHAR(50) NOT NULL,                 -- Organisation-specific numbering: 'RES-2026-001'
    title               VARCHAR(500) NOT NULL,
    body                TEXT NOT NULL,
    resolution_type     VARCHAR(50) NOT NULL DEFAULT 'ordinary'
                        CHECK (resolution_type IN ('ordinary','special','written','circular','consent','emergency')),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','proposed','passed','rejected','superseded','rescinded')),
    effective_date      DATE,
    expiry_date         DATE,
    supersedes_id       UUID REFERENCES resolutions(id),      -- If this resolution replaces an earlier one
    passed_at           TIMESTAMPTZ,
    signed_document_id  UUID,                                 -- FK to documents (signed copy)
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_resolution_number UNIQUE (organisation_id, resolution_number)
);

CREATE INDEX idx_resolutions_org ON resolutions(organisation_id);
CREATE INDEX idx_resolutions_meeting ON resolutions(meeting_id);

-- Action items extracted from meetings
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

CREATE INDEX idx_action_items_assignee ON action_items(assigned_to, status);
CREATE INDEX idx_action_items_meeting ON action_items(meeting_id);
```

### Document Management Tables

```sql
-- Document repository
CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    folder_id           UUID REFERENCES document_folders(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    document_type       VARCHAR(50) NOT NULL DEFAULT 'general'
                        CHECK (document_type IN (
                            'board_pack','agenda','minutes','resolution','charter','bylaws',
                            'policy','contract','report','presentation','financial','legal',
                            'compliance','questionnaire','disclosure','certificate','general'
                        )),
    mime_type           VARCHAR(200) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    storage_path        VARCHAR(1000) NOT NULL,               -- Object storage path (S3/MinIO)
    storage_provider    VARCHAR(50) NOT NULL DEFAULT 'local', -- 's3', 'azure_blob', 'local'
    checksum_sha256     CHAR(64) NOT NULL,                    -- Integrity verification
    encryption_key_id   VARCHAR(200),                         -- Reference to KMS key used for encryption
    is_encrypted        BOOLEAN NOT NULL DEFAULT TRUE,
    version             INTEGER NOT NULL DEFAULT 1,
    parent_document_id  UUID REFERENCES documents(id),        -- Previous version
    is_latest           BOOLEAN NOT NULL DEFAULT TRUE,
    classification      VARCHAR(30) NOT NULL DEFAULT 'confidential'
                        CHECK (classification IN ('public','internal','confidential','restricted','top_secret')),
    retention_until     DATE,                                 -- Data retention policy
    uploaded_by         UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_documents_org ON documents(organisation_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_documents_folder ON documents(folder_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_documents_type ON documents(document_type) WHERE deleted_at IS NULL;

-- Document folders with hierarchical structure
CREATE TABLE document_folders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    parent_folder_id    UUID REFERENCES document_folders(id),
    name                VARCHAR(300) NOT NULL,
    description         TEXT,
    folder_path         VARCHAR(2000) NOT NULL,               -- Materialized path: '/bylaws/2026/'
    board_id            UUID REFERENCES boards(id),           -- Scope to a specific board
    committee_id        UUID REFERENCES committees(id),       -- Scope to a specific committee
    classification      VARCHAR(30) NOT NULL DEFAULT 'confidential',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT uq_folder_path UNIQUE (organisation_id, folder_path)
);

-- Document access permissions (granular per-document or per-folder)
CREATE TABLE document_permissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID REFERENCES documents(id) ON DELETE CASCADE,
    folder_id           UUID REFERENCES document_folders(id) ON DELETE CASCADE,
    user_id             UUID REFERENCES users(id),
    role_id             UUID REFERENCES roles(id),
    permission_level    VARCHAR(20) NOT NULL DEFAULT 'read'
                        CHECK (permission_level IN ('read','annotate','download','edit','admin')),
    granted_by          UUID REFERENCES users(id),
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,
    CONSTRAINT chk_doc_perm_target CHECK (document_id IS NOT NULL OR folder_id IS NOT NULL),
    CONSTRAINT chk_doc_perm_grantee CHECK (user_id IS NOT NULL OR role_id IS NOT NULL)
);

-- Document annotations
CREATE TABLE document_annotations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    page_number         INTEGER,
    annotation_type     VARCHAR(30) NOT NULL DEFAULT 'highlight'
                        CHECK (annotation_type IN ('highlight','note','bookmark','underline','strikethrough','freehand')),
    content             TEXT,
    position_data       JSONB,                                -- x, y, width, height coordinates
    colour              VARCHAR(7),                           -- Hex colour code
    is_private          BOOLEAN NOT NULL DEFAULT TRUE,        -- Private vs shared annotation
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_annotations_doc ON document_annotations(document_id);
CREATE INDEX idx_annotations_user ON document_annotations(user_id);

-- E-signature requests and tracking
CREATE TABLE signature_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    document_id         UUID NOT NULL REFERENCES documents(id),
    requested_by        UUID NOT NULL REFERENCES users(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','in_progress','completed','cancelled','expired')),
    provider            VARCHAR(30) NOT NULL DEFAULT 'built_in'
                        CHECK (provider IN ('built_in','docusign','adobe_sign')),
    external_envelope_id VARCHAR(200),                        -- DocuSign envelope ID
    due_date            TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE signature_signers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id          UUID NOT NULL REFERENCES signature_requests(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    sign_order          INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','sent','viewed','signed','declined')),
    signed_at           TIMESTAMPTZ,
    signature_data      TEXT,                                 -- Encrypted signature image/certificate
    ip_address          INET,
    user_agent          VARCHAR(500),
    CONSTRAINT uq_signer UNIQUE (request_id, user_id)
);
```

### Insider Trading Compliance Tables

```sql
-- Trading windows (blackout periods)
CREATE TABLE trading_windows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(300) NOT NULL,                -- 'Q1 2026 Earnings Blackout'
    window_type         VARCHAR(50) NOT NULL DEFAULT 'quarterly_blackout'
                        CHECK (window_type IN ('quarterly_blackout','event_specific','annual_meeting','ipo_lockup','custom')),
    status              VARCHAR(20) NOT NULL DEFAULT 'open'
                        CHECK (status IN ('open','closed','pending')),
    open_date           DATE NOT NULL,                        -- Date trading window opens (trading allowed)
    close_date          DATE NOT NULL,                        -- Date trading window closes (blackout begins)
    blackout_start      TIMESTAMPTZ NOT NULL,
    blackout_end        TIMESTAMPTZ,
    reason              TEXT,
    applies_to_all      BOOLEAN NOT NULL DEFAULT TRUE,        -- Applies to all insiders
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trading_windows_org ON trading_windows(organisation_id);
CREATE INDEX idx_trading_windows_dates ON trading_windows(blackout_start, blackout_end);

-- Insider designations
CREATE TABLE insider_designations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    insider_type        VARCHAR(50) NOT NULL
                        CHECK (insider_type IN ('section_16_officer','section_16_director','10pct_owner','designated_insider','connected_person')),
    designation_date    DATE NOT NULL,
    revocation_date     DATE,
    sec_filing_status   VARCHAR(30) DEFAULT 'not_required'
                        CHECK (sec_filing_status IN ('not_required','form3_filed','form4_required','form5_required','delinquent')),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_insider_designation UNIQUE (organisation_id, user_id, insider_type)
);

-- Pre-clearance requests for insider trades
CREATE TABLE preclearance_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    requester_user_id   UUID NOT NULL REFERENCES users(id),
    insider_designation_id UUID NOT NULL REFERENCES insider_designations(id),
    security_type       VARCHAR(50) NOT NULL
                        CHECK (security_type IN ('common_stock','preferred_stock','option','restricted_stock','warrant','convertible','derivative','other')),
    security_name       VARCHAR(300) NOT NULL,
    ticker_symbol       VARCHAR(20),
    transaction_type    VARCHAR(50) NOT NULL
                        CHECK (transaction_type IN ('purchase','sale','exercise','gift','conversion','other')),
    requested_shares    DECIMAL(18,4),
    estimated_price     DECIMAL(18,4),
    estimated_value     DECIMAL(18,2),
    transaction_date    DATE,
    broker_name         VARCHAR(300),
    broker_account      VARCHAR(100),
    justification       TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','approved','denied','expired','cancelled','executed')),
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    review_notes        TEXT,
    approval_expires_at TIMESTAMPTZ,                          -- Typically 2-5 business days
    executed_at         TIMESTAMPTZ,
    actual_shares       DECIMAL(18,4),
    actual_price        DECIMAL(18,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_preclearance_org ON preclearance_requests(organisation_id);
CREATE INDEX idx_preclearance_requester ON preclearance_requests(requester_user_id);
CREATE INDEX idx_preclearance_status ON preclearance_requests(status);

-- Insider holdings tracking
CREATE TABLE insider_holdings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    security_type       VARCHAR(50) NOT NULL,
    security_name       VARCHAR(300) NOT NULL,
    ticker_symbol       VARCHAR(20),
    shares_held         DECIMAL(18,4) NOT NULL,
    direct_indirect     VARCHAR(10) NOT NULL DEFAULT 'direct'
                        CHECK (direct_indirect IN ('direct','indirect')),
    indirect_nature     VARCHAR(300),                          -- Nature of indirect ownership
    as_of_date          DATE NOT NULL,
    source              VARCHAR(50) NOT NULL DEFAULT 'self_reported'
                        CHECK (source IN ('self_reported','form4_filing','broker_feed','manual')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_insider_holdings_user ON insider_holdings(user_id, as_of_date);
```

### Conflict of Interest & Disclosures

```sql
-- Director interest register
CREATE TABLE interest_disclosures (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    disclosure_type     VARCHAR(50) NOT NULL
                        CHECK (disclosure_type IN (
                            'financial_interest','employment','board_membership',
                            'family_relationship','contractual','property','other'
                        )),
    entity_name         VARCHAR(500),                         -- Name of entity with which interest exists
    description         TEXT NOT NULL,
    monetary_value      DECIMAL(18,2),
    currency            CHAR(3),                              -- ISO 4217
    is_material         BOOLEAN,
    disclosed_date      DATE NOT NULL,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','expired','withdrawn','superseded')),
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_disclosures_user ON interest_disclosures(user_id);
CREATE INDEX idx_disclosures_org ON interest_disclosures(organisation_id);

-- D&O questionnaires
CREATE TABLE questionnaires (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    title               VARCHAR(500) NOT NULL,
    questionnaire_type  VARCHAR(50) NOT NULL
                        CHECK (questionnaire_type IN ('dno_annual','dno_onboarding','conflict_of_interest','independence','suitability','custom')),
    description         TEXT,
    template_content    JSONB NOT NULL,                        -- Question definitions
    is_template         BOOLEAN NOT NULL DEFAULT FALSE,
    due_date            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','active','closed','archived')),
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE questionnaire_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    questionnaire_id    UUID NOT NULL REFERENCES questionnaires(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    responses           JSONB NOT NULL,                        -- Structured answers
    status              VARCHAR(20) NOT NULL DEFAULT 'in_progress'
                        CHECK (status IN ('in_progress','submitted','reviewed','approved')),
    submitted_at        TIMESTAMPTZ,
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    review_notes        TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_questionnaire_response UNIQUE (questionnaire_id, user_id)
);
```

### Board Effectiveness & Skills Tracking

```sql
-- Skills taxonomy for board composition analysis
CREATE TABLE skills_taxonomy (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    category            VARCHAR(200) NOT NULL,                -- 'Finance', 'Technology', 'Legal', 'Industry'
    skill_name          VARCHAR(300) NOT NULL,                -- 'M&A Experience', 'Cybersecurity', 'ESG'
    description         TEXT,
    is_critical         BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_skill UNIQUE (organisation_id, category, skill_name)
);

-- Director skills mapping
CREATE TABLE director_skills (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    skill_id            UUID NOT NULL REFERENCES skills_taxonomy(id),
    proficiency_level   VARCHAR(20) NOT NULL DEFAULT 'competent'
                        CHECK (proficiency_level IN ('awareness','competent','expert','thought_leader')),
    years_experience    INTEGER,
    self_assessed       BOOLEAN NOT NULL DEFAULT TRUE,
    assessed_by         UUID REFERENCES users(id),
    assessed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_director_skill UNIQUE (user_id, skill_id)
);

-- Board evaluations
CREATE TABLE board_evaluations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    board_id            UUID NOT NULL REFERENCES boards(id),
    evaluation_year     INTEGER NOT NULL,
    evaluation_type     VARCHAR(50) NOT NULL DEFAULT 'self_assessment'
                        CHECK (evaluation_type IN ('self_assessment','peer_review','external','chair_led')),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','in_progress','completed','archived')),
    summary             TEXT,
    overall_rating      DECIMAL(3,1),                         -- 1.0 to 5.0
    report_document_id  UUID REFERENCES documents(id),
    conducted_by        VARCHAR(300),                          -- External evaluator name if applicable
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_board_eval UNIQUE (board_id, evaluation_year, evaluation_type)
);
```

### Compliance & Filing Tracking

```sql
-- Regulatory filing deadlines
CREATE TABLE filing_deadlines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    entity_id           UUID REFERENCES organisations(id),    -- Specific subsidiary entity
    jurisdiction        CHAR(2) NOT NULL,
    filing_type         VARCHAR(100) NOT NULL,                -- 'annual_return', 'form_4', 'form_8k', 'proxy_statement', 'change_of_directors'
    regulatory_body     VARCHAR(200) NOT NULL,                -- 'SEC', 'Companies House', 'ASIC'
    due_date            DATE NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'upcoming'
                        CHECK (status IN ('upcoming','in_progress','filed','overdue','waived','not_applicable')),
    filed_at            TIMESTAMPTZ,
    filed_by            UUID REFERENCES users(id),
    filing_reference    VARCHAR(200),                         -- External filing confirmation number
    recurrence          VARCHAR(30),                          -- 'annual', 'quarterly', 'event_driven', 'one_time'
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_filing_deadlines_due ON filing_deadlines(due_date, status);
CREATE INDEX idx_filing_deadlines_org ON filing_deadlines(organisation_id);
```

### Secure Messaging

```sql
-- Secure message threads (replacing email for sensitive governance discussions)
CREATE TABLE message_threads (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    board_id            UUID REFERENCES boards(id),
    committee_id        UUID REFERENCES committees(id),
    subject             VARCHAR(500) NOT NULL,
    classification      VARCHAR(30) NOT NULL DEFAULT 'confidential',
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE message_participants (
    thread_id           UUID NOT NULL REFERENCES message_threads(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    added_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_read_at        TIMESTAMPTZ,
    is_muted            BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (thread_id, user_id)
);

CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id           UUID NOT NULL REFERENCES message_threads(id) ON DELETE CASCADE,
    sender_id           UUID NOT NULL REFERENCES users(id),
    body                TEXT NOT NULL,                         -- Encrypted content
    body_format         VARCHAR(20) NOT NULL DEFAULT 'plain',
    is_encrypted        BOOLEAN NOT NULL DEFAULT TRUE,
    reply_to_id         UUID REFERENCES messages(id),
    sent_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    edited_at           TIMESTAMPTZ,
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_messages_thread ON messages(thread_id, sent_at);
```

### Audit Log

```sql
-- Comprehensive audit trail
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    user_id             UUID,
    session_id          UUID,
    action              VARCHAR(50) NOT NULL,                 -- 'create', 'read', 'update', 'delete', 'login', 'download', 'share'
    resource_type       VARCHAR(100) NOT NULL,                -- 'document', 'meeting', 'vote', 'preclearance', etc.
    resource_id         UUID,
    resource_name       VARCHAR(500),
    old_values          JSONB,                                -- Before state
    new_values          JSONB,                                -- After state
    ip_address          INET,
    user_agent          VARCHAR(500),
    geo_location        VARCHAR(200),
    severity            VARCHAR(20) NOT NULL DEFAULT 'info'
                        CHECK (severity IN ('info','warning','critical','security')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partitioned by month for performance
-- CREATE TABLE audit_log PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_org ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_action ON audit_log(action, created_at);
```

### Notification & Calendar Integration

```sql
-- Notification system
CREATE TABLE notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    notification_type   VARCHAR(50) NOT NULL,                 -- 'meeting_invite', 'document_shared', 'vote_open', 'action_due', 'preclearance_status'
    title               VARCHAR(500) NOT NULL,
    body                TEXT,
    resource_type       VARCHAR(100),
    resource_id         UUID,
    channel             VARCHAR(20) NOT NULL DEFAULT 'in_app'
                        CHECK (channel IN ('in_app','email','push','sms')),
    is_read             BOOLEAN NOT NULL DEFAULT FALSE,
    read_at             TIMESTAMPTZ,
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read, created_at);

-- Calendar sync records
CREATE TABLE calendar_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    provider            VARCHAR(30) NOT NULL                  -- 'outlook', 'google', 'apple'
                        CHECK (provider IN ('outlook','google','apple','ical')),
    external_event_id   VARCHAR(500),
    ical_uid            VARCHAR(500),                         -- RFC 5545 UID
    sync_status         VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (sync_status IN ('pending','synced','failed','cancelled')),
    last_synced_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Pros and Cons

### Pros

1. **Referential integrity everywhere**: Every relationship is enforced at the database level. You cannot create a preclearance request for a non-existent insider, assign an action item to a non-existent user, or orphan a committee from its board. In a compliance-critical domain, this is not negotiable.

2. **Mature tooling and ecosystem**: PostgreSQL has 30+ years of battle-tested reliability. Every ORM, migration tool, backup solution, and monitoring platform supports it. Hiring developers who know PostgreSQL is straightforward.

3. **Row-level security for multi-tenancy**: PostgreSQL's native RLS policies can enforce `organisation_id` filtering at the database level, eliminating an entire class of data-leak bugs that would be catastrophic for a board portal.

4. **Complex reporting queries**: Governance platforms require cross-entity, cross-meeting, cross-time-period reporting. SQL excels at this. Querying "show me all resolutions passed by the audit committee in the last 12 months that reference cybersecurity risk" is a single JOIN query.

5. **ACID compliance**: Board votes, resolution status changes, and insider trading pre-clearance approvals must be atomic. Partial state is unacceptable. PostgreSQL's transactional guarantees are the strongest available.

6. **Regulatory audit readiness**: The schema explicitly tracks who did what, when, from where — meeting SOC 2, ISO 27001, and SEC requirements for tamper-evident records.

### Cons

1. **Schema rigidity**: Adding new entity types, new questionnaire formats, or jurisdiction-specific fields requires DDL migrations. In a domain where regulations change frequently and jurisdictions vary widely, this creates deployment friction.

2. **Hierarchical data is awkward**: Entity ownership trees and organisational hierarchies require recursive CTEs or materialized path columns. Neither is as natural as a graph database for "show me all ultimate beneficial owners of this subsidiary chain."

3. **Document metadata limitations**: Governance documents vary enormously in structure. A normalized schema can capture common fields but struggles with the long tail of document-type-specific metadata without resorting to EAV patterns or JSONB columns (which partially defeats the purpose of full normalization).

4. **Horizontal scaling constraints**: While PostgreSQL supports read replicas, sharding a highly relational schema across multiple nodes is significantly harder than with document databases. For organisations with thousands of entities across hundreds of jurisdictions, single-node write limits may eventually become relevant.

5. **Schema migration complexity at scale**: With 40+ interconnected tables, schema migrations require careful coordination, especially with zero-downtime deployment requirements for a board portal that directors may access at any time.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|----------------|
| **Database** | PostgreSQL 16+ with partitioning for audit_log |
| **ORM** | Prisma, Drizzle, or SQLAlchemy with Alembic migrations |
| **Connection pooling** | PgBouncer or built-in pool in application framework |
| **Encryption at rest** | pgcrypto extension + application-level AES-256 for documents |
| **Full-text search** | PostgreSQL tsvector/GIN indexes for document and resolution search |
| **Backup** | pg_basebackup + WAL archiving to S3-compatible storage |
| **Monitoring** | pg_stat_statements, pgwatch2, or Datadog PostgreSQL integration |
| **Row-level security** | PostgreSQL RLS policies on all tenant-scoped tables |
| **Partitioning** | Range partitioning on audit_log by month; hash partitioning on documents by organisation_id if >10M rows |

---

## Migration & Scaling Considerations

### Initial Deployment

- Start with a single PostgreSQL 16 instance on managed service (AWS RDS, Azure Database for PostgreSQL, or Google Cloud SQL)
- Enable connection pooling from day one (PgBouncer with 200+ connections)
- Create the audit_log table as a partitioned table from the start — retrofitting partitioning is painful
- Seed system roles and permissions as part of database initialization

### Growth Phase (100-1,000 organisations)

- Add read replicas for reporting and analytics queries
- Implement table partitioning on audit_log (monthly), documents (by organisation_id hash), and notifications (by created_at range)
- Consider moving document storage metadata to a separate tablespace for I/O isolation
- Add pg_cron for automated compliance deadline monitoring and notification generation

### Scale Phase (1,000+ organisations)

- Evaluate Citus extension for distributed PostgreSQL if write throughput becomes a bottleneck
- Archive historical audit data to cold storage (S3 + Athena or DuckDB for ad-hoc queries)
- Split read-heavy reporting into a dedicated analytics database (materialised views refreshed on schedule)
- Consider extracting document search into a dedicated Elasticsearch/OpenSearch cluster

### Data Retention

- Governance records (meetings, minutes, resolutions) should be retained for 7+ years per most jurisdictions
- Insider trading records must be retained for the duration of SEC statute of limitations (typically 5 years after last related transaction)
- Audit logs should be immutable and retained for the full retention period
- Implement automated retention policies via pg_cron + archival jobs
