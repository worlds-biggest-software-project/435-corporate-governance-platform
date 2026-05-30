# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Corporate Governance Platform (Candidate #435)
> Generated: 2026-05-25

## Overview

This model applies Event Sourcing with Command Query Responsibility Segregation (CQRS) to the corporate governance domain. Every state change — from scheduling a meeting to approving an insider trade pre-clearance to casting a board vote — is captured as an immutable domain event in an append-only event store. The current state of any entity is derived by replaying its event stream.

This approach is particularly well-suited to corporate governance for a fundamental reason: **governance is, by its nature, a record of decisions and their consequences over time**. Boards do not simply "have" a current state — they accumulate a history of resolutions passed, votes cast, disclosures made, and approvals granted. An event-sourced system makes this history a first-class citizen rather than an afterthought bolted on via audit log tables.

The CQRS pattern separates the write side (commands that produce events) from the read side (projections optimized for specific query patterns). This means the system can simultaneously serve the needs of a director looking at their current action items (read model) and a compliance officer reconstructing the full decision chain that led to a specific insider trade approval (event replay).

---

## Architecture Overview

```
                    ┌─────────────┐
                    │  API Layer  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
         │ Command  │  │  Query  │  │  Event  │
         │ Handler  │  │ Handler │  │ Reactor │
         └────┬────┘  └────┬────┘  └────┬────┘
              │            │            │
         ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
         │  Event  │  │  Read   │  │  Side   │
         │  Store  │  │ Models  │  │ Effects │
         │ (Write) │  │ (Read)  │  │(Notify) │
         └─────────┘  └─────────┘  └─────────┘
```

**Write Side**: Commands are validated against business rules, then produce domain events that are appended to the event store. The event store is the single source of truth.

**Read Side**: Event handlers (projectors) consume events and update denormalized read models optimized for specific UI views and API queries.

**Side Effects**: Event reactors trigger notifications, compliance checks, calendar syncs, and external integrations in response to events.

---

## Event Store Schema (PostgreSQL)

The event store itself is a surprisingly simple schema — the complexity lives in the event types and the projections.

```sql
-- Core event store table (append-only, immutable)
CREATE TABLE event_store (
    global_position     BIGSERIAL PRIMARY KEY,                -- Global ordering across all streams
    stream_id           VARCHAR(500) NOT NULL,                -- Aggregate identity: 'meeting-{uuid}', 'preclearance-{uuid}'
    stream_position     BIGINT NOT NULL,                      -- Position within the specific stream
    event_type          VARCHAR(200) NOT NULL,                -- Fully qualified event name
    event_data          JSONB NOT NULL,                       -- Event payload
    metadata            JSONB NOT NULL DEFAULT '{}',          -- Causation, correlation, user context
    organisation_id     UUID NOT NULL,                        -- Tenant isolation
    aggregate_type      VARCHAR(100) NOT NULL,                -- 'Meeting', 'Board', 'PreclearanceRequest', etc.
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID,                                 -- User who triggered the command
    causation_id        UUID,                                 -- ID of the command that caused this event
    correlation_id      UUID,                                 -- Correlation chain for distributed tracing
    schema_version      INTEGER NOT NULL DEFAULT 1,           -- Event schema version for evolution
    checksum            VARCHAR(64),                          -- SHA-256 of event_data for tamper detection

    CONSTRAINT uq_stream_position UNIQUE (stream_id, stream_position)
);

-- Optimized indexes for event store access patterns
CREATE INDEX idx_events_stream ON event_store(stream_id, stream_position);
CREATE INDEX idx_events_type ON event_store(event_type, global_position);
CREATE INDEX idx_events_org ON event_store(organisation_id, global_position);
CREATE INDEX idx_events_aggregate ON event_store(aggregate_type, global_position);
CREATE INDEX idx_events_correlation ON event_store(correlation_id);
CREATE INDEX idx_events_created ON event_store(created_at);

-- Prevent any updates or deletes on the event store (immutability enforcement)
CREATE RULE no_update_events AS ON UPDATE TO event_store DO INSTEAD NOTHING;
CREATE RULE no_delete_events AS ON DELETE TO event_store DO INSTEAD NOTHING;

-- Snapshot store for performance (avoid replaying long event streams)
CREATE TABLE snapshots (
    stream_id           VARCHAR(500) PRIMARY KEY,
    stream_position     BIGINT NOT NULL,                      -- Position at which snapshot was taken
    snapshot_data       JSONB NOT NULL,                       -- Serialized aggregate state
    aggregate_type      VARCHAR(100) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Subscription checkpoints (for projectors and reactors)
CREATE TABLE subscription_checkpoints (
    subscription_id     VARCHAR(200) PRIMARY KEY,             -- 'projection:meetings_read_model', 'reactor:email_notifications'
    last_position       BIGINT NOT NULL DEFAULT 0,            -- Last processed global_position
    last_processed_at   TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'running'
                        CHECK (status IN ('running','paused','failed','rebuilding')),
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_events (
    id                  BIGSERIAL PRIMARY KEY,
    subscription_id     VARCHAR(200) NOT NULL,
    global_position     BIGINT NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    event_data          JSONB NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Domain Events Catalogue

### Organisation & Entity Events

```
OrganisationCreated
  { organisation_id, name, legal_name, lei, jurisdiction, organisation_type, fiscal_year_end }

OrganisationUpdated
  { organisation_id, changes: { field: { old, new } } }

OrganisationStatusChanged
  { organisation_id, old_status, new_status, reason }

SubsidiaryLinked
  { parent_entity_id, child_entity_id, relationship_type, ownership_pct, effective_date }

SubsidiaryUnlinked
  { parent_entity_id, child_entity_id, end_date, reason }

OwnershipPercentageChanged
  { parent_entity_id, child_entity_id, old_pct, new_pct, effective_date }
```

### Board & Committee Events

```
BoardCreated
  { board_id, organisation_id, name, board_type, quorum_requirement }

BoardMemberAppointed
  { board_id, user_id, board_role, independence_status, appointed_date, appointment_type }

BoardMemberRoleChanged
  { board_id, user_id, old_role, new_role, effective_date }

BoardMemberResigned
  { board_id, user_id, resigned_date, reason }

BoardMemberTermExpired
  { board_id, user_id, expiry_date }

CommitteeCreated
  { committee_id, board_id, name, committee_type, charter_document_id }

CommitteeMemberAppointed
  { committee_id, user_id, committee_role, appointed_date }

CommitteeMemberRemoved
  { committee_id, user_id, removed_date, reason }

CommitteeCharterUpdated
  { committee_id, old_charter_document_id, new_charter_document_id }
```

### Meeting Lifecycle Events

```
MeetingScheduled
  { meeting_id, board_id, committee_id, title, meeting_type, scheduled_start, scheduled_end, location, virtual_meeting_url }

MeetingRescheduled
  { meeting_id, old_start, new_start, old_end, new_end, reason }

MeetingCancelled
  { meeting_id, reason, cancelled_by }

AttendeeInvited
  { meeting_id, user_id, attendee_role }

AttendeeRSVPReceived
  { meeting_id, user_id, response: 'confirmed' | 'declined', proxy_for_user_id }

AgendaItemAdded
  { meeting_id, agenda_item_id, title, item_type, sort_order, presenter_user_id, allocated_minutes }

AgendaItemReordered
  { meeting_id, agenda_item_id, old_position, new_position }

AgendaItemUpdated
  { meeting_id, agenda_item_id, changes }

AgendaItemRemoved
  { meeting_id, agenda_item_id, reason }

PreReadAttached
  { meeting_id, agenda_item_id, document_id, attachment_type }

MeetingStarted
  { meeting_id, actual_start, quorum_met, quorum_count, attendees_present: [user_id] }

MeetingAdjourned
  { meeting_id, adjourned_at, reason, reconvene_date }

MeetingCompleted
  { meeting_id, actual_end, attendees_present: [user_id], attendees_absent: [user_id] }

AttendeeJoined
  { meeting_id, user_id, joined_at }

AttendeeLeft
  { meeting_id, user_id, left_at }
```

### Voting & Resolution Events

```
VoteProposed
  { vote_id, meeting_id, agenda_item_id, title, description, vote_type, required_threshold }

VoteMovedAndSeconded
  { vote_id, moved_by, seconded_by }

VoteOpened
  { vote_id, opened_at }

VoteCast
  { vote_id, user_id, vote_cast: 'for' | 'against' | 'abstain' | 'recuse', voted_at }

VoteClosed
  { vote_id, closed_at, votes_for, votes_against, votes_abstain, result: 'passed' | 'failed' }

VoteWithdrawn
  { vote_id, withdrawn_by, reason }

ResolutionDrafted
  { resolution_id, meeting_id, vote_id, resolution_number, title, body, resolution_type }

ResolutionPassed
  { resolution_id, passed_at, effective_date }

ResolutionRejected
  { resolution_id, rejected_at, reason }

ResolutionSuperseded
  { resolution_id, superseded_by_id, superseded_at }

ResolutionRescinded
  { resolution_id, rescinded_at, reason, rescinded_by }
```

### Minutes Events

```
MinutesDraftGenerated
  { meeting_id, minutes_id, version, content, ai_generated, generated_from: 'transcript' | 'agenda' | 'notes' }

MinutesDraftEdited
  { meeting_id, minutes_id, version, edited_by, changes_summary }

MinutesSubmittedForReview
  { meeting_id, minutes_id, version, submitted_by }

MinutesApproved
  { meeting_id, minutes_id, version, approved_by, approved_at }

MinutesSigned
  { meeting_id, minutes_id, signed_document_id, signed_by, signed_at }
```

### Action Item Events

```
ActionItemCreated
  { action_item_id, meeting_id, agenda_item_id, resolution_id, title, description, assigned_to, due_date, priority }

ActionItemReassigned
  { action_item_id, old_assignee, new_assignee, reassigned_by }

ActionItemDueDateChanged
  { action_item_id, old_due_date, new_due_date, changed_by }

ActionItemStatusChanged
  { action_item_id, old_status, new_status, changed_by, notes }

ActionItemCompleted
  { action_item_id, completed_at, completed_by, completion_notes }
```

### Document Events

```
DocumentUploaded
  { document_id, organisation_id, folder_id, title, document_type, mime_type, file_size_bytes, checksum_sha256, classification, uploaded_by }

DocumentVersionCreated
  { document_id, parent_document_id, version, title, file_size_bytes, checksum_sha256, uploaded_by }

DocumentMoved
  { document_id, old_folder_id, new_folder_id, moved_by }

DocumentClassificationChanged
  { document_id, old_classification, new_classification, changed_by }

DocumentPermissionGranted
  { document_id, folder_id, grantee_user_id, grantee_role_id, permission_level, granted_by, expires_at }

DocumentPermissionRevoked
  { document_id, folder_id, grantee_user_id, grantee_role_id, revoked_by }

DocumentAccessed
  { document_id, user_id, access_type: 'viewed' | 'downloaded' | 'printed', ip_address, user_agent }

DocumentAnnotated
  { document_id, annotation_id, user_id, annotation_type, page_number, content, is_private }

DocumentDeleted
  { document_id, deleted_by, reason }

SignatureRequested
  { request_id, document_id, requested_by, signers: [{ user_id, sign_order }], due_date }

SignatureCompleted
  { request_id, signer_user_id, signed_at, ip_address }

SignatureRequestCompleted
  { request_id, completed_at, all_signers_signed: true }

SignatureRequestExpired
  { request_id, expired_at, unsigned_signers: [user_id] }
```

### Insider Trading Compliance Events

```
InsiderDesignated
  { designation_id, organisation_id, user_id, insider_type, designation_date }

InsiderDesignationRevoked
  { designation_id, user_id, revocation_date, reason }

TradingWindowOpened
  { window_id, organisation_id, name, window_type, open_date, applies_to_all }

TradingWindowClosed
  { window_id, close_date, blackout_start, reason }

TradingWindowBlackoutEnded
  { window_id, blackout_end }

PreclearanceRequested
  { request_id, organisation_id, requester_user_id, insider_designation_id, security_type, security_name, ticker_symbol, transaction_type, requested_shares, estimated_price, transaction_date, broker_name }

PreclearanceApproved
  { request_id, reviewed_by, reviewed_at, approval_expires_at, conditions }

PreclearanceDenied
  { request_id, reviewed_by, reviewed_at, denial_reason }

PreclearanceExpired
  { request_id, expired_at }

PreclearanceExecuted
  { request_id, executed_at, actual_shares, actual_price }

InsiderHoldingReported
  { holding_id, user_id, security_type, security_name, shares_held, direct_indirect, as_of_date, source }

TradingViolationDetected
  { violation_id, organisation_id, user_id, violation_type, description, detected_at, related_preclearance_id, related_window_id }
```

### Disclosure & Conflict of Interest Events

```
InterestDisclosed
  { disclosure_id, organisation_id, user_id, disclosure_type, entity_name, description, monetary_value, currency, effective_from }

InterestDisclosureUpdated
  { disclosure_id, changes }

InterestDisclosureExpired
  { disclosure_id, effective_to }

InterestDisclosureWithdrawn
  { disclosure_id, withdrawn_by, reason }

ConflictOfInterestFlagged
  { conflict_id, organisation_id, user_id, agenda_item_id, disclosure_id, description, flagged_by }

ConflictOfInterestResolved
  { conflict_id, resolution: 'recusal' | 'waiver' | 'no_conflict', resolved_by, resolution_notes }

QuestionnaireIssued
  { questionnaire_id, organisation_id, title, questionnaire_type, recipients: [user_id], due_date }

QuestionnaireResponseSubmitted
  { questionnaire_id, user_id, submitted_at }

QuestionnaireResponseReviewed
  { questionnaire_id, user_id, reviewed_by, reviewed_at, status }
```

### Board Effectiveness Events

```
SkillDefined
  { skill_id, organisation_id, category, skill_name, is_critical }

DirectorSkillAssessed
  { user_id, skill_id, proficiency_level, years_experience, assessed_by }

BoardEvaluationStarted
  { evaluation_id, board_id, evaluation_year, evaluation_type }

BoardEvaluationCompleted
  { evaluation_id, overall_rating, summary, report_document_id }

BoardCompositionGapIdentified
  { board_id, skill_id, gap_description, severity, identified_at }
```

### User & Access Events

```
UserRegistered
  { user_id, email, first_name, last_name }

UserProfileUpdated
  { user_id, changes }

UserInvitedToOrganisation
  { organisation_id, user_id, role, invited_by }

UserJoinedOrganisation
  { organisation_id, user_id, role, start_date }

UserRemovedFromOrganisation
  { organisation_id, user_id, end_date, reason }

UserLoggedIn
  { user_id, ip_address, user_agent, mfa_used, login_method: 'password' | 'sso' | 'webauthn' }

UserLoggedOut
  { user_id, session_duration_seconds }

UserMFAEnabled
  { user_id, mfa_method }

UserSuspended
  { user_id, suspended_by, reason }

UserReactivated
  { user_id, reactivated_by }

RoleAssigned
  { user_id, role_id, scope_type, scope_id, granted_by }

RoleRevoked
  { user_id, role_id, scope_type, scope_id, revoked_by, reason }
```

### Messaging Events

```
MessageThreadCreated
  { thread_id, organisation_id, subject, classification, created_by, participants: [user_id] }

MessageSent
  { message_id, thread_id, sender_id, body_preview, is_encrypted }

MessageRead
  { thread_id, user_id, last_read_message_id, read_at }

ParticipantAddedToThread
  { thread_id, user_id, added_by }

ParticipantRemovedFromThread
  { thread_id, user_id, removed_by }
```

---

## Read Model Projections (PostgreSQL)

Read models are denormalized tables optimized for specific query patterns. They are rebuilt from the event store whenever needed.

### Director Dashboard Projection

```sql
-- Optimized for the director's landing page: "What do I need to do?"
CREATE TABLE rm_director_dashboard (
    user_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,

    -- Upcoming meetings
    next_meeting_id     UUID,
    next_meeting_title  VARCHAR(500),
    next_meeting_date   TIMESTAMPTZ,
    next_meeting_board  VARCHAR(500),
    unread_preread_count INTEGER NOT NULL DEFAULT 0,

    -- Action items
    open_action_items   INTEGER NOT NULL DEFAULT 0,
    overdue_action_items INTEGER NOT NULL DEFAULT 0,

    -- Pending votes
    pending_votes       INTEGER NOT NULL DEFAULT 0,

    -- Unread messages
    unread_messages     INTEGER NOT NULL DEFAULT 0,

    -- Pending signature requests
    pending_signatures  INTEGER NOT NULL DEFAULT 0,

    -- Pending questionnaires
    pending_questionnaires INTEGER NOT NULL DEFAULT 0,

    -- Last login
    last_login_at       TIMESTAMPTZ,

    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, organisation_id)
);

-- Projector: listens to MeetingScheduled, ActionItemCreated, ActionItemCompleted,
-- VoteOpened, VoteClosed, MessageSent, MessageRead, SignatureRequested, SignatureCompleted,
-- QuestionnaireIssued, QuestionnaireResponseSubmitted, UserLoggedIn
```

### Meeting List Projection

```sql
-- Optimized for meeting list views and calendar integration
CREATE TABLE rm_meetings (
    meeting_id          UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    board_id            UUID,
    board_name          VARCHAR(500),
    committee_id        UUID,
    committee_name      VARCHAR(500),
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL,
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    location            VARCHAR(500),
    virtual_meeting_url VARCHAR(1000),
    quorum_met          BOOLEAN,
    attendee_count      INTEGER NOT NULL DEFAULT 0,
    confirmed_count     INTEGER NOT NULL DEFAULT 0,
    agenda_item_count   INTEGER NOT NULL DEFAULT 0,
    vote_count          INTEGER NOT NULL DEFAULT 0,
    resolution_count    INTEGER NOT NULL DEFAULT 0,
    action_item_count   INTEGER NOT NULL DEFAULT 0,
    minutes_status      VARCHAR(30),
    created_by_name     VARCHAR(400),
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_meetings_org ON rm_meetings(organisation_id, scheduled_start);
CREATE INDEX idx_rm_meetings_board ON rm_meetings(board_id, scheduled_start);
CREATE INDEX idx_rm_meetings_status ON rm_meetings(status);
```

### Entity Hierarchy Projection

```sql
-- Denormalized entity tree for fast hierarchy display
CREATE TABLE rm_entity_hierarchy (
    entity_id           UUID NOT NULL,
    parent_entity_id    UUID,
    root_entity_id      UUID NOT NULL,                        -- Ultimate parent
    entity_name         VARCHAR(500) NOT NULL,
    legal_name          VARCHAR(500),
    lei                 CHAR(20),
    jurisdiction        CHAR(2) NOT NULL,
    organisation_type   VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    relationship_type   VARCHAR(50),
    ownership_pct       DECIMAL(5,2),
    depth               INTEGER NOT NULL DEFAULT 0,           -- Depth in hierarchy (0 = root)
    path                VARCHAR(2000) NOT NULL,               -- Materialized path: '/root-uuid/parent-uuid/this-uuid'
    director_count      INTEGER NOT NULL DEFAULT 0,
    officer_count       INTEGER NOT NULL DEFAULT 0,
    subsidiary_count    INTEGER NOT NULL DEFAULT 0,
    next_filing_due     DATE,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (entity_id, parent_entity_id)
);

CREATE INDEX idx_rm_entity_root ON rm_entity_hierarchy(root_entity_id);
CREATE INDEX idx_rm_entity_path ON rm_entity_hierarchy USING gist (path gist_trgm_ops);
```

### Insider Compliance Projection

```sql
-- Optimized for compliance officer dashboard
CREATE TABLE rm_insider_compliance (
    user_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    user_name           VARCHAR(400) NOT NULL,
    insider_type        VARCHAR(50) NOT NULL,
    designation_date    DATE NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- Current holdings summary
    total_direct_shares DECIMAL(18,4) NOT NULL DEFAULT 0,
    total_indirect_shares DECIMAL(18,4) NOT NULL DEFAULT 0,
    holdings_as_of      DATE,

    -- Trading window status
    current_window_status VARCHAR(20),                        -- 'open', 'closed', 'blackout'
    current_window_name VARCHAR(300),
    blackout_end_date   TIMESTAMPTZ,

    -- Preclearance summary
    pending_preclearances INTEGER NOT NULL DEFAULT 0,
    active_approvals    INTEGER NOT NULL DEFAULT 0,
    total_requests_ytd  INTEGER NOT NULL DEFAULT 0,
    total_denied_ytd    INTEGER NOT NULL DEFAULT 0,

    -- Compliance flags
    has_violations      BOOLEAN NOT NULL DEFAULT FALSE,
    last_violation_date DATE,
    outstanding_disclosures INTEGER NOT NULL DEFAULT 0,

    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, organisation_id)
);

CREATE INDEX idx_rm_insider_org ON rm_insider_compliance(organisation_id);
CREATE INDEX idx_rm_insider_violations ON rm_insider_compliance(has_violations) WHERE has_violations = TRUE;
```

### Resolution Register Projection

```sql
-- Searchable register of all resolutions
CREATE TABLE rm_resolution_register (
    resolution_id       UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    resolution_number   VARCHAR(50) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    body_preview        TEXT,                                  -- First 500 chars
    resolution_type     VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    meeting_id          UUID,
    meeting_title       VARCHAR(500),
    meeting_date        TIMESTAMPTZ,
    board_name          VARCHAR(500),
    committee_name      VARCHAR(500),
    passed_at           TIMESTAMPTZ,
    effective_date      DATE,
    expiry_date         DATE,
    superseded_by       VARCHAR(50),                          -- Resolution number of superseding resolution
    votes_for           INTEGER,
    votes_against       INTEGER,
    votes_abstain       INTEGER,
    action_item_count   INTEGER NOT NULL DEFAULT 0,
    search_vector       TSVECTOR,                             -- Full-text search
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_resolutions_org ON rm_resolution_register(organisation_id, passed_at);
CREATE INDEX idx_rm_resolutions_search ON rm_resolution_register USING gin(search_vector);
```

### Document Access Audit Projection

```sql
-- Optimized for compliance reporting: who accessed what, when
CREATE TABLE rm_document_access_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    document_id         UUID NOT NULL,
    document_title      VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50) NOT NULL,
    classification      VARCHAR(30) NOT NULL,
    user_id             UUID NOT NULL,
    user_name           VARCHAR(400) NOT NULL,
    access_type         VARCHAR(20) NOT NULL,                 -- 'viewed', 'downloaded', 'printed', 'annotated'
    ip_address          INET,
    user_agent          VARCHAR(500),
    accessed_at         TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_doc_access_org ON rm_document_access_log(organisation_id, accessed_at);
CREATE INDEX idx_rm_doc_access_doc ON rm_document_access_log(document_id, accessed_at);
CREATE INDEX idx_rm_doc_access_user ON rm_document_access_log(user_id, accessed_at);
```

### Compliance Calendar Projection

```sql
-- Aggregated view of all upcoming compliance deadlines
CREATE TABLE rm_compliance_calendar (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    entity_id           UUID,
    entity_name         VARCHAR(500),
    jurisdiction        CHAR(2) NOT NULL,
    filing_type         VARCHAR(100) NOT NULL,
    regulatory_body     VARCHAR(200) NOT NULL,
    due_date            DATE NOT NULL,
    status              VARCHAR(30) NOT NULL,
    days_until_due      INTEGER,                              -- Computed on projection refresh
    is_overdue          BOOLEAN NOT NULL DEFAULT FALSE,
    assigned_to_name    VARCHAR(400),
    recurrence          VARCHAR(30),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_compliance_cal_org ON rm_compliance_calendar(organisation_id, due_date);
CREATE INDEX idx_rm_compliance_cal_overdue ON rm_compliance_calendar(is_overdue) WHERE is_overdue = TRUE;
```

---

## Command Handlers (Domain Logic Examples)

### Meeting Aggregate

```python
class MeetingAggregate:
    """Domain aggregate for meeting lifecycle management."""

    def schedule(self, cmd: ScheduleMeeting) -> List[Event]:
        # Validate: board/committee exists, no scheduling conflicts
        # Validate: scheduled_start < scheduled_end
        # Validate: user has permission to schedule
        return [MeetingScheduled(
            meeting_id=cmd.meeting_id,
            board_id=cmd.board_id,
            committee_id=cmd.committee_id,
            title=cmd.title,
            meeting_type=cmd.meeting_type,
            scheduled_start=cmd.scheduled_start,
            scheduled_end=cmd.scheduled_end,
            location=cmd.location,
            virtual_meeting_url=cmd.virtual_meeting_url,
        )]

    def start(self, cmd: StartMeeting) -> List[Event]:
        # Validate: meeting is in 'scheduled' state
        # Validate: quorum check against board/committee requirements
        if not self._is_scheduled():
            raise InvalidStateError("Meeting must be scheduled before starting")

        events = [MeetingStarted(
            meeting_id=self.id,
            actual_start=cmd.timestamp,
            quorum_met=cmd.quorum_met,
            quorum_count=cmd.quorum_count,
            attendees_present=cmd.attendees_present,
        )]
        return events

    def cast_vote(self, cmd: CastVote) -> List[Event]:
        # Validate: meeting is in progress
        # Validate: vote is open
        # Validate: user is an eligible voter (board/committee member)
        # Validate: user has not already voted
        # Check: if user has a disclosed conflict, auto-flag for recusal
        vote = self._find_vote(cmd.vote_id)
        if vote.status != 'open':
            raise InvalidStateError("Vote is not open")
        if cmd.user_id in vote.votes_cast:
            raise BusinessRuleViolation("User has already voted")

        return [VoteCast(
            vote_id=cmd.vote_id,
            user_id=cmd.user_id,
            vote_cast=cmd.vote_cast,
            voted_at=cmd.timestamp,
        )]
```

### Preclearance Aggregate

```python
class PreclearanceAggregate:
    """Domain aggregate for insider trade pre-clearance."""

    def request(self, cmd: RequestPreclearance) -> List[Event]:
        # Validate: user is a designated insider
        # Validate: no active blackout window
        # Validate: no pending preclearance for same security
        return [PreclearanceRequested(
            request_id=cmd.request_id,
            organisation_id=cmd.organisation_id,
            requester_user_id=cmd.requester_user_id,
            insider_designation_id=cmd.insider_designation_id,
            security_type=cmd.security_type,
            security_name=cmd.security_name,
            ticker_symbol=cmd.ticker_symbol,
            transaction_type=cmd.transaction_type,
            requested_shares=cmd.requested_shares,
            estimated_price=cmd.estimated_price,
            transaction_date=cmd.transaction_date,
            broker_name=cmd.broker_name,
        )]

    def approve(self, cmd: ApprovePreclearance) -> List[Event]:
        if self.status != 'pending':
            raise InvalidStateError("Only pending requests can be approved")
        # Validate: reviewer is not the requester (segregation of duties)
        if cmd.reviewed_by == self.requester_user_id:
            raise BusinessRuleViolation("Reviewer cannot be the requester")
        # Validate: no blackout window is now active (could have changed since request)

        return [PreclearanceApproved(
            request_id=self.id,
            reviewed_by=cmd.reviewed_by,
            reviewed_at=cmd.timestamp,
            approval_expires_at=cmd.timestamp + timedelta(days=cmd.expiry_days or 3),
            conditions=cmd.conditions,
        )]

    def deny(self, cmd: DenyPreclearance) -> List[Event]:
        if self.status != 'pending':
            raise InvalidStateError("Only pending requests can be denied")
        return [PreclearanceDenied(
            request_id=self.id,
            reviewed_by=cmd.reviewed_by,
            reviewed_at=cmd.timestamp,
            denial_reason=cmd.denial_reason,
        )]
```

---

## Event Reactors (Side Effects)

```python
class NotificationReactor:
    """Sends notifications in response to domain events."""

    @handles(MeetingScheduled)
    async def on_meeting_scheduled(self, event):
        # Send calendar invites to all board/committee members
        # Trigger email notifications
        # Create in-app notifications

    @handles(VoteOpened)
    async def on_vote_opened(self, event):
        # Notify all eligible voters
        # Send push notifications to mobile

    @handles(PreclearanceRequested)
    async def on_preclearance_requested(self, event):
        # Notify compliance officer(s)
        # Start SLA timer for response

    @handles(PreclearanceApproved)
    async def on_preclearance_approved(self, event):
        # Notify requester
        # Set expiry timer
        # Log for Section 16 reporting

    @handles(TradingWindowClosed)
    async def on_trading_window_closed(self, event):
        # Notify all designated insiders
        # Cancel any pending preclearance approvals that overlap with blackout
        # Send reminder emails

    @handles(ActionItemCreated)
    async def on_action_item_created(self, event):
        # Notify assignee
        # Add to assignee's task list
        # Schedule due date reminder


class ComplianceReactor:
    """Enforces compliance rules in response to domain events."""

    @handles(PreclearanceRequested)
    async def check_trading_window(self, event):
        # Verify no active blackout window
        # If blackout active, auto-deny with reason

    @handles(PreclearanceApproved)
    async def schedule_expiry(self, event):
        # Schedule a PreclearanceExpired event if not executed within window

    @handles(TradingViolationDetected)
    async def escalate_violation(self, event):
        # Alert compliance officer, general counsel
        # Create audit record
        # Optionally freeze insider's trading permissions


class CalendarSyncReactor:
    """Syncs meetings to external calendars."""

    @handles(MeetingScheduled)
    async def sync_to_calendars(self, event):
        # Create iCalendar event (RFC 5545)
        # Push to Outlook via Graph API
        # Push to Google Calendar via API

    @handles(MeetingRescheduled)
    async def update_calendars(self, event):
        # Update external calendar events

    @handles(MeetingCancelled)
    async def cancel_calendar_events(self, event):
        # Cancel external calendar events
```

---

## Pros and Cons

### Pros

1. **Governance-native audit trail**: Every change is an immutable event with a timestamp, user attribution, and causation chain. This is not an "audit log bolted on" — it IS the data model. Compliance officers can reconstruct the complete decision chain for any governance action: "Show me every event from the moment this insider requested pre-clearance through to the executed trade and the Form 4 filing." This is the gold standard for SOC 2, ISO 27001, and SEC compliance.

2. **Temporal queries are first-class**: "What was the board composition on March 15, 2025?" Just replay the Board aggregate's event stream up to that date. "What disclosures did Director X have active when they voted on that agenda item?" Replay the disclosure stream to the vote timestamp. No temporal table extensions needed — the event stream IS the temporal history.

3. **Complete, tamper-evident record**: Events can be cryptographically chained (each event's checksum includes the previous event's checksum), creating a blockchain-like tamper-evident log without the overhead of a blockchain. This provides legal-grade evidence for regulatory investigations.

4. **Domain model clarity**: Event naming forces precision about what happened: `PreclearanceApproved` is unambiguous; an UPDATE to a status column is not. The event catalogue serves as living domain documentation.

5. **Independent read model optimization**: Each UI view gets its own denormalized projection optimized for exactly its query pattern. The director dashboard projection is completely different from the compliance officer's insider trading projection, and neither compromises the other's performance.

6. **Retroactive projections**: When new reporting requirements emerge (e.g., a new regulatory body requires a specific report format), you can create a new projector and replay the entire event history to populate it — without touching the source data.

7. **Testing and debugging**: You can replay any sequence of events to reproduce bugs. You can take a production event stream and replay it in a test environment to reproduce exact state.

### Cons

1. **Complexity overhead**: Event sourcing is architecturally complex. The team must understand aggregates, event streams, projections, eventual consistency, idempotency, and schema evolution. This is a significant learning curve for a team that may be more comfortable with CRUD.

2. **Eventual consistency in read models**: There is always a delay (typically milliseconds, but potentially seconds under load) between when a command is processed and when read models reflect the change. A director casting a vote may not immediately see the updated tally. For a governance platform where actions are deliberate and measured (not real-time trading), this is usually acceptable — but it requires careful UX design (optimistic updates, loading states).

3. **Event schema evolution is hard**: Once an event type is published, its schema becomes a contract. Changing `PreclearanceRequested` to add a new required field means dealing with both old and new event versions forever. Upcasters and version negotiation add complexity.

4. **Storage growth**: Every change produces an event. A busy board with frequent document access logging will generate many events. The event store grows monotonically. Snapshotting mitigates read performance but does not reduce storage. Archival strategies are required for multi-year governance records.

5. **Projection rebuild time**: If a projector has a bug and needs to be rebuilt, replaying millions of events can take significant time. During rebuilds, read models may be stale or unavailable. This requires operational procedures (blue-green projection deployments).

6. **Debugging production issues**: While event replay is powerful for reproducing bugs, understanding the current state of a complex aggregate requires mentally (or programmatically) replaying potentially thousands of events. There is no simple "SELECT * FROM meetings WHERE id = X" to see current state — you must look at the read model, which may be out of date.

7. **Limited ecosystem support**: Event sourcing libraries and frameworks are less mature than traditional ORMs. PostgreSQL does not have native event store support (unlike EventStoreDB), so you must build or adopt a library.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|----------------|
| **Event Store** | PostgreSQL 16+ (primary) or EventStoreDB (dedicated) |
| **Event Bus** | NATS JetStream or Apache Kafka for event distribution |
| **Application Framework** | Node.js with Sequent, or Python with eventsourcing library |
| **Read Model Database** | PostgreSQL for transactional read models; Redis for dashboard caches |
| **Projection Engine** | Custom projectors consuming from event store or message bus |
| **Serialization** | JSON with schema registry (Confluent Schema Registry or custom) |
| **Snapshotting** | Every 100-500 events per aggregate, stored in snapshots table |
| **Idempotency** | Command IDs + idempotency keys for at-least-once delivery |
| **Monitoring** | Projection lag monitoring, dead letter queue alerting, event store growth metrics |

---

## Migration & Scaling Considerations

### Initial Deployment

- Start with PostgreSQL as both event store and read model database (single database, separate schemas: `events` and `read_models`)
- Implement event store with optimistic concurrency control on stream_position
- Build 5-7 core projections for MVP: director dashboard, meeting list, resolution register, insider compliance, document access log, compliance calendar
- Use in-process projectors (same application process) for simplicity

### Growth Phase

- Move projectors to separate worker processes for isolation
- Introduce NATS JetStream or Kafka as an event distribution layer between event store and projectors
- Add Redis for caching frequently-accessed read models (director dashboard)
- Implement event archival: move events older than 2 years to cold storage while keeping snapshots for aggregate reconstitution
- Partition the event_store table by organisation_id or by creation date

### Scale Phase

- Evaluate EventStoreDB as a dedicated event store if PostgreSQL event throughput becomes a bottleneck
- Implement parallel projection rebuilds across multiple workers
- Consider separate read model databases per projection category (compliance projections on a different instance than meeting projections)
- Implement event store compaction: for aggregates with thousands of events, create periodic snapshots and archive events before the snapshot point
- Add a data lake (S3 + Parquet) for long-term governance analytics across the full event history

### Event Schema Evolution Strategy

- All events carry a `schema_version` field
- Upcasters transform old event versions to current versions during replay
- Never delete or rename event types — deprecate and introduce new types
- Maintain a schema registry documenting all event types, their versions, and upcaster chains
- Test projection rebuilds from event zero as part of CI/CD pipeline
