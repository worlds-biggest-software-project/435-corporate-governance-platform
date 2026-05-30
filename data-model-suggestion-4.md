# Data Model Suggestion 4: Graph Database (Neo4j) + PostgreSQL Polyglot Architecture

> Project: Corporate Governance Platform (Candidate #435)
> Generated: 2026-05-25

## Overview

This model uses a polyglot persistence architecture that combines Neo4j (graph database) as the primary store for the relationship-heavy core of corporate governance with PostgreSQL for transactional data, audit logs, and document metadata. The key insight is that corporate governance is fundamentally a domain of **interconnected relationships** — and these relationships are where the most critical compliance questions live.

Consider the questions a compliance officer or company secretary asks daily:

- "What is the complete ownership chain from our ultimate parent company to this fifth-tier subsidiary in Luxembourg?"
- "Which directors on this board also sit on boards of companies that are our competitors, suppliers, or customers — creating potential conflicts of interest?"
- "If this insider trades, which entities in our corporate group are affected by the trading window?"
- "Which committees does this director serve on, across which entities in our corporate structure, and do any of those committee charters create conflicting authorities?"
- "Show me every person who had access to this document and their relationships to the entities referenced in it."

In a relational database, every one of these queries requires multiple recursive CTEs, self-joins, or materialised path lookups. In a graph database, they are natural traversals — often expressible in a single Cypher query.

Neo4j is the natural choice for the graph component: it dominates the graph database market for financial services (used by 20 of the world's 25 largest financial services firms), has mature enterprise features (RBAC, encryption, clustering), and has extensive tooling for ownership and compliance graph analysis.

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                      API / Application Layer               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Governance   │  │  Document &  │  │   Compliance  │    │
│  │  Graph Engine │  │  Transaction │  │   & Audit     │    │
│  │              │  │   Engine     │  │   Engine      │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                  │             │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐    │
│  │    Neo4j     │  │  PostgreSQL  │  │  PostgreSQL  │    │
│  │  (Graph)     │  │ (Documents,  │  │ (Audit Log,  │    │
│  │              │  │  Meetings,   │  │  Compliance  │    │
│  │  Entities    │  │  Voting,     │  │  Filings)    │    │
│  │  People      │  │  Minutes)    │  │              │    │
│  │  Ownership   │  │              │  │              │    │
│  │  Boards      │  │              │  │              │    │
│  │  Committees  │  │              │  │              │    │
│  │  Conflicts   │  │              │  │              │    │
│  │  Trading     │  │              │  │              │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │           Synchronisation Layer (CDC/Events)      │     │
│  │  Keeps Neo4j and PostgreSQL consistent via        │     │
│  │  change data capture and event-driven sync        │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Neo4j** handles: entity structures, ownership chains, board/committee membership, director relationships, conflict-of-interest detection, insider designation networks, and any query that requires graph traversal.

**PostgreSQL** handles: meeting agendas/minutes/attachments, voting records, document storage metadata, questionnaire responses, audit logs, notifications, and secure messaging — all data that benefits from ACID transactions, relational integrity, and traditional query patterns.

---

## Neo4j Graph Schema

### Node Types

```cypher
// Organisation / Legal Entity
(:Organisation {
    id: 'uuid',
    name: 'Acme Corporation',
    legal_name: 'Acme Corporation Inc.',
    lei: '5493001KJTIIGC8Y1R12',           // ISO 17442
    jurisdiction: 'US',                      // ISO 3166-1
    organisation_type: 'public_company',     // public_company, private_company, nonprofit, etc.
    incorporation_date: date('2005-03-15'),
    status: 'active',
    sec_cik: '0001234567',
    ticker_symbol: 'ACME',
    stock_exchange: 'NYSE',
    fiscal_year_end: '12-31',
    timezone: 'America/New_York',
    created_at: datetime(),
    updated_at: datetime()
})

// Person (director, officer, insider, or any individual in the governance ecosystem)
(:Person {
    id: 'uuid',
    email: 'jane.smith@example.com',
    first_name: 'Jane',
    last_name: 'Smith',
    title: 'Chief Financial Officer',
    status: 'active',
    created_at: datetime(),
    updated_at: datetime()
})

// Board
(:Board {
    id: 'uuid',
    name: 'Board of Directors',
    board_type: 'governing',               // governing, advisory, subsidiary
    quorum_requirement: 5,
    quorum_type: 'count',
    status: 'active',
    created_at: datetime()
})

// Committee
(:Committee {
    id: 'uuid',
    name: 'Audit Committee',
    committee_type: 'audit',
    quorum_requirement: 3,
    status: 'active',
    independence_required: true,
    financial_expert_required: true,
    created_at: datetime()
})

// Skill (for board composition analysis)
(:Skill {
    id: 'uuid',
    category: 'Finance',
    name: 'M&A Experience',
    is_critical: true,
    description: 'Experience leading or overseeing M&A transactions'
})

// TradingWindow
(:TradingWindow {
    id: 'uuid',
    name: 'Q1 2026 Earnings Blackout',
    window_type: 'quarterly_blackout',
    status: 'closed',
    blackout_start: datetime('2026-03-15T00:00:00Z'),
    blackout_end: datetime('2026-04-20T00:00:00Z'),
    created_at: datetime()
})

// Meeting (reference node — detail in PostgreSQL)
(:Meeting {
    id: 'uuid',
    title: 'Q2 Board Meeting',
    meeting_type: 'regular',
    scheduled_start: datetime('2026-06-15T09:00:00Z'),
    status: 'scheduled'
})

// Document (reference node — content metadata in PostgreSQL)
(:Document {
    id: 'uuid',
    title: 'Q1 2026 Financial Statements',
    document_type: 'financial',
    classification: 'confidential',
    created_at: datetime()
})

// Resolution (reference node — full content in PostgreSQL)
(:Resolution {
    id: 'uuid',
    resolution_number: 'RES-2026-015',
    title: 'Approval of Q1 Budget',
    status: 'passed',
    passed_at: datetime('2026-03-15T10:30:00Z')
})

// InterestDisclosure
(:InterestDisclosure {
    id: 'uuid',
    disclosure_type: 'board_membership',
    entity_name: 'Competitor Corp',
    description: 'Non-executive director at Competitor Corp',
    is_material: true,
    disclosed_date: date('2026-01-15'),
    effective_from: date('2024-06-01'),
    status: 'active'
})

// Jurisdiction
(:Jurisdiction {
    code: 'US',
    name: 'United States',
    regulatory_body: 'SEC',
    insider_trading_regime: 'Section 16 / Rule 10b-5',
    filing_system: 'EDGAR'
})

// FilingRequirement
(:FilingRequirement {
    id: 'uuid',
    filing_type: 'annual_return',
    regulatory_body: 'Companies House',
    due_date: date('2026-09-15'),
    recurrence: 'annual',
    status: 'upcoming'
})
```

### Relationship Types

```cypher
// Ownership & Corporate Structure
(:Organisation)-[:OWNS {
    ownership_pct: 75.0,
    voting_rights_pct: 75.0,
    effective_date: date('2020-01-01'),
    end_date: null,
    relationship_type: 'subsidiary',        // subsidiary, branch, joint_venture, affiliate
    consolidation_method: 'full'
}]->(:Organisation)

(:Organisation)-[:INCORPORATED_IN]->(:Jurisdiction)

// Board & Committee Membership
(:Organisation)-[:HAS_BOARD]->(:Board)
(:Board)-[:HAS_COMMITTEE]->(:Committee)

(:Person)-[:MEMBER_OF {
    board_role: 'chair',                    // chair, vice_chair, member, secretary, observer
    independence_status: 'independent',
    appointed_date: date('2022-01-15'),
    term_expiry_date: date('2025-01-15'),
    appointment_type: 'elected',
    is_current: true
}]->(:Board)

(:Person)-[:SERVES_ON {
    committee_role: 'chair',
    appointed_date: date('2022-06-01'),
    is_current: true
}]->(:Committee)

// Employment & Officer Relationships
(:Person)-[:OFFICER_OF {
    officer_type: 'cfo',
    appointed_date: date('2021-03-01'),
    is_current: true,
    section_16_filer: true
}]->(:Organisation)

(:Person)-[:EMPLOYED_BY {
    start_date: date('2018-06-15'),
    department: 'Finance',
    is_current: true
}]->(:Organisation)

// Organisation Membership (platform access)
(:Person)-[:MEMBER_OF_ORG {
    role: 'director',
    start_date: date('2022-01-15'),
    is_insider: true
}]->(:Organisation)

// Insider Trading Relationships
(:Person)-[:DESIGNATED_INSIDER {
    insider_type: 'section_16_director',
    designation_date: date('2022-01-15'),
    is_active: true
}]->(:Organisation)

(:TradingWindow)-[:APPLIES_TO]->(:Organisation)

(:Person)-[:RESTRICTED_BY {
    reason: 'board_member'
}]->(:TradingWindow)

// Conflict of Interest Network
(:Person)-[:HAS_INTEREST]->(:InterestDisclosure)

(:InterestDisclosure)-[:RELATES_TO {
    conflict_type: 'competing_interest'     // competing_interest, supplier, customer, family
}]->(:Organisation)

(:InterestDisclosure)-[:CONFLICTS_WITH]->(:Resolution)

// Skills & Competencies
(:Person)-[:HAS_SKILL {
    proficiency_level: 4,                   // 1-5
    years_experience: 12,
    self_assessed: false,
    assessed_at: datetime()
}]->(:Skill)

(:Board)-[:REQUIRES_SKILL {
    minimum_proficiency: 3,
    minimum_members_with_skill: 2,
    is_critical: true
}]->(:Skill)

// Meeting Relationships (lightweight — detail in PostgreSQL)
(:Meeting)-[:HELD_BY]->(:Board)
(:Meeting)-[:HELD_BY]->(:Committee)
(:Person)-[:ATTENDED {attendance_status: 'attended'}]->(:Meeting)
(:Meeting)-[:PRODUCED]->(:Resolution)

// Document Relationships
(:Document)-[:BELONGS_TO]->(:Organisation)
(:Document)-[:PRESENTED_AT]->(:Meeting)
(:Person)-[:UPLOADED]->(:Document)
(:Person)-[:ACCESSED {
    access_type: 'viewed',
    accessed_at: datetime()
}]->(:Document)
(:Document)-[:REFERENCED_IN]->(:Resolution)

// Filing Relationships
(:FilingRequirement)-[:REQUIRED_FOR]->(:Organisation)
(:FilingRequirement)-[:GOVERNED_BY]->(:Jurisdiction)
(:Person)-[:RESPONSIBLE_FOR]->(:FilingRequirement)

// Cross-reference Relationships (connecting people across entities)
(:Person)-[:FAMILY_MEMBER_OF {
    relationship: 'spouse'                  // spouse, child, parent, sibling
}]->(:Person)

(:Person)-[:BUSINESS_ASSOCIATE_OF {
    nature: 'co-investor'
}]->(:Person)
```

---

## Key Graph Queries (Cypher)

### Corporate Structure Traversal

```cypher
// Get complete ownership tree from ultimate parent to all subsidiaries (any depth)
MATCH path = (parent:Organisation {id: $org_id})-[:OWNS*]->(subsidiary:Organisation)
RETURN parent, subsidiary,
       [r IN relationships(path) | r.ownership_pct] AS ownership_chain,
       reduce(pct = 100.0, r IN relationships(path) | pct * r.ownership_pct / 100) AS effective_ownership_pct
ORDER BY length(path)

// Find all entities where effective ownership exceeds 50% (consolidation candidates)
MATCH path = (parent:Organisation {id: $org_id})-[:OWNS*]->(sub:Organisation)
WITH sub, reduce(pct = 100.0, r IN relationships(path) | pct * r.ownership_pct / 100) AS effective_pct
WHERE effective_pct > 50
RETURN sub.name, sub.jurisdiction, effective_pct
ORDER BY effective_pct DESC

// Find circular ownership (a red flag for governance and tax compliance)
MATCH (org:Organisation {id: $org_id})-[:OWNS*]->(org)
RETURN 'CIRCULAR OWNERSHIP DETECTED' AS warning, [n IN nodes(path) | n.name] AS chain
```

### Conflict of Interest Detection

```cypher
// Find all directors who sit on boards of competing, supplier, or customer organisations
MATCH (d:Person)-[:MEMBER_OF]->(b1:Board)<-[:HAS_BOARD]-(org1:Organisation {id: $org_id})
MATCH (d)-[:MEMBER_OF]->(b2:Board)<-[:HAS_BOARD]-(org2:Organisation)
WHERE org1 <> org2 AND b1 <> b2
RETURN d.first_name + ' ' + d.last_name AS director,
       org2.name AS other_organisation,
       b2.name AS other_board,
       [(d)-[m:MEMBER_OF]->(b2) | m.board_role][0] AS role_on_other_board

// Detect potential conflicts between director interests and pending resolutions
MATCH (d:Person)-[:HAS_INTEREST]->(disc:InterestDisclosure)-[:RELATES_TO]->(ext_org:Organisation)
MATCH (res:Resolution {status: 'proposed'})<-[:PRODUCED]-(m:Meeting)-[:HELD_BY]->(b:Board)
MATCH (d)-[:MEMBER_OF]->(b)
WHERE disc.status = 'active'
  AND (res.title CONTAINS ext_org.name OR res.body CONTAINS ext_org.name)
RETURN d.first_name + ' ' + d.last_name AS director,
       disc.entity_name AS disclosed_interest,
       disc.disclosure_type,
       res.resolution_number,
       res.title AS resolution_title,
       'POTENTIAL CONFLICT' AS flag

// Find all persons with family connections to insiders of this organisation
MATCH (insider:Person)-[:DESIGNATED_INSIDER]->(org:Organisation {id: $org_id})
MATCH (insider)-[:FAMILY_MEMBER_OF*1..2]-(connected:Person)
RETURN insider.first_name + ' ' + insider.last_name AS insider,
       connected.first_name + ' ' + connected.last_name AS connected_person,
       [r IN relationships((insider)-[:FAMILY_MEMBER_OF*1..2]-(connected)) | r.relationship] AS relationship_chain
```

### Insider Trading Compliance

```cypher
// Check if a person is restricted from trading (active blackout window)
MATCH (p:Person {id: $user_id})-[:DESIGNATED_INSIDER]->(org:Organisation)
MATCH (tw:TradingWindow)-[:APPLIES_TO]->(org)
WHERE tw.status = 'closed'
  AND tw.blackout_start <= datetime()
  AND (tw.blackout_end IS NULL OR tw.blackout_end >= datetime())
RETURN tw.name AS active_blackout,
       tw.blackout_start AS blackout_since,
       tw.blackout_end AS expected_end,
       org.name AS restricted_organisation

// Find all insiders affected by a trading window across the corporate group
MATCH (parent:Organisation {id: $org_id})-[:OWNS*0..]->(entity:Organisation)
MATCH (tw:TradingWindow {id: $window_id})-[:APPLIES_TO]->(entity)
MATCH (p:Person)-[:DESIGNATED_INSIDER]->(entity)
RETURN p.first_name + ' ' + p.last_name AS insider,
       p.email,
       entity.name AS entity,
       collect(DISTINCT [(p)-[d:DESIGNATED_INSIDER]->(entity) | d.insider_type]) AS insider_types

// Trace the information chain: who had access to MNPI before a trade
MATCH (doc:Document {id: $material_document_id})-[:PRESENTED_AT]->(m:Meeting)
MATCH (p:Person)-[:ACCESSED]->(doc)
MATCH (p)-[:DESIGNATED_INSIDER]->(org:Organisation)
RETURN p.first_name + ' ' + p.last_name AS person_with_access,
       p.email,
       [(p)-[a:ACCESSED]->(doc) | a.accessed_at][0] AS access_time,
       m.title AS meeting_context,
       org.name AS insider_at
ORDER BY [(p)-[a:ACCESSED]->(doc) | a.accessed_at][0]
```

### Board Composition & Effectiveness

```cypher
// Board skills gap analysis
MATCH (b:Board {id: $board_id})-[:REQUIRES_SKILL]->(required_skill:Skill)
OPTIONAL MATCH (p:Person)-[has:HAS_SKILL]->(required_skill)
  WHERE (p)-[:MEMBER_OF]->(b) AND has.proficiency_level >= 3
WITH required_skill, collect(p) AS qualified_members
RETURN required_skill.category,
       required_skill.name AS skill,
       required_skill.is_critical,
       size(qualified_members) AS members_with_skill,
       [m IN qualified_members | m.first_name + ' ' + m.last_name] AS qualified_directors,
       CASE WHEN size(qualified_members) = 0 THEN 'GAP' ELSE 'COVERED' END AS status

// Board diversity and independence analysis
MATCH (b:Board {id: $board_id})<-[m:MEMBER_OF]-(p:Person)
WHERE m.is_current = true
RETURN count(p) AS total_members,
       count(CASE WHEN m.independence_status = 'independent' THEN 1 END) AS independent_count,
       count(CASE WHEN m.independence_status = 'executive' THEN 1 END) AS executive_count,
       toFloat(count(CASE WHEN m.independence_status = 'independent' THEN 1 END)) / count(p) * 100 AS independence_pct,
       avg(duration.between(m.appointed_date, date()).years) AS avg_tenure_years,
       collect({
           name: p.first_name + ' ' + p.last_name,
           role: m.board_role,
           independence: m.independence_status,
           tenure_years: duration.between(m.appointed_date, date()).years,
           term_expires: m.term_expiry_date
       }) AS member_details

// Cross-board service analysis (director overboarding risk)
MATCH (p:Person)-[m:MEMBER_OF]->(b:Board)
WHERE m.is_current = true
WITH p, count(b) AS board_count, collect(b.name) AS boards_served
WHERE board_count > 1
RETURN p.first_name + ' ' + p.last_name AS director,
       p.email,
       board_count,
       boards_served,
       CASE WHEN board_count > 4 THEN 'OVERBOARDING RISK' ELSE 'OK' END AS risk_flag
ORDER BY board_count DESC
```

### Document Access & Compliance Tracing

```cypher
// Who accessed a confidential document and what are their relationships?
MATCH (p:Person)-[a:ACCESSED]->(doc:Document {id: $doc_id})
OPTIONAL MATCH (p)-[:MEMBER_OF]->(b:Board)<-[:HAS_BOARD]-(org:Organisation)
OPTIONAL MATCH (p)-[:DESIGNATED_INSIDER]->(org2:Organisation)
RETURN p.first_name + ' ' + p.last_name AS person,
       a.access_type,
       a.accessed_at,
       collect(DISTINCT b.name) AS board_memberships,
       collect(DISTINCT org2.name) AS insider_designations

// Compliance filing impact analysis: which entities are affected by a jurisdiction change?
MATCH (j:Jurisdiction {code: $jurisdiction_code})<-[:INCORPORATED_IN]-(org:Organisation)
MATCH (org)<-[:OWNS*0..]-(parent:Organisation)
RETURN org.name AS affected_entity,
       org.lei,
       parent.name AS parent_entity,
       [(fr:FilingRequirement)-[:REQUIRED_FOR]->(org) WHERE fr.status IN ['upcoming', 'in_progress'] |
        {filing_type: fr.filing_type, due_date: fr.due_date}] AS pending_filings
```

---

## PostgreSQL Schema (Transactional Data)

The PostgreSQL side stores the transactional, content-heavy data that does not benefit from graph traversal.

```sql
-- Meeting details (full content — Neo4j only holds reference nodes)
CREATE TABLE meetings (
    id                  UUID PRIMARY KEY,                     -- Same UUID as Neo4j Meeting node
    organisation_id     UUID NOT NULL,
    board_id            UUID,
    committee_id        UUID,
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    location            VARCHAR(500),
    virtual_meeting_url VARCHAR(1000),
    virtual_provider    VARCHAR(50),
    quorum_met          BOOLEAN,
    quorum_count        INTEGER,
    meeting_config      JSONB NOT NULL DEFAULT '{}',
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

-- Agenda items with full content
CREATE TABLE agenda_items (
    id                  UUID PRIMARY KEY,
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    parent_item_id      UUID REFERENCES agenda_items(id),
    sort_order          INTEGER NOT NULL,
    item_number         VARCHAR(20),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    item_type           VARCHAR(50) NOT NULL,
    presenter_user_id   UUID,
    allocated_minutes   INTEGER,
    is_consent_item     BOOLEAN NOT NULL DEFAULT FALSE,
    is_confidential     BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    item_data           JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agenda_meeting ON agenda_items(meeting_id, sort_order);

-- Meeting attendees
CREATE TABLE meeting_attendees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL,
    attendance_status   VARCHAR(30) NOT NULL DEFAULT 'invited',
    attendee_role       VARCHAR(50) DEFAULT 'member',
    proxy_for_user_id   UUID,
    joined_at           TIMESTAMPTZ,
    left_at             TIMESTAMPTZ,
    rsvp_at             TIMESTAMPTZ,
    CONSTRAINT uq_meeting_attendee UNIQUE (meeting_id, user_id)
);

-- Meeting minutes with full rich text
CREATE TABLE meeting_minutes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    version             INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    content             TEXT NOT NULL,
    content_format      VARCHAR(20) NOT NULL DEFAULT 'markdown',
    ai_generated        BOOLEAN NOT NULL DEFAULT FALSE,
    minutes_metadata    JSONB NOT NULL DEFAULT '{}',
    approved_at         TIMESTAMPTZ,
    approved_by         UUID,
    signed_document_id  UUID,
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_minutes_version UNIQUE (meeting_id, version)
);

-- Vote records (relational — critical transactional data)
CREATE TABLE votes (
    id                  UUID PRIMARY KEY,
    meeting_id          UUID NOT NULL REFERENCES meetings(id),
    agenda_item_id      UUID REFERENCES agenda_items(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    vote_type           VARCHAR(50) NOT NULL,
    required_threshold  DECIMAL(5,2),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    opened_at           TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    moved_by            UUID,
    seconded_by         UUID,
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
    user_id             UUID NOT NULL,
    vote_cast           VARCHAR(20) NOT NULL,
    voted_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes               TEXT,
    CONSTRAINT uq_vote_record UNIQUE (vote_id, user_id)
);

-- Resolutions (full text in PostgreSQL; reference in Neo4j)
CREATE TABLE resolutions (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    meeting_id          UUID REFERENCES meetings(id),
    vote_id             UUID REFERENCES votes(id),
    resolution_number   VARCHAR(50) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    body                TEXT NOT NULL,
    resolution_type     VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    effective_date      DATE,
    expiry_date         DATE,
    supersedes_id       UUID REFERENCES resolutions(id),
    passed_at           TIMESTAMPTZ,
    signed_document_id  UUID,
    resolution_metadata JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_resolution_number UNIQUE (organisation_id, resolution_number)
);

-- Action items
CREATE TABLE action_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    meeting_id          UUID REFERENCES meetings(id),
    agenda_item_id      UUID REFERENCES agenda_items(id),
    resolution_id       UUID REFERENCES resolutions(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    assigned_to         UUID NOT NULL,
    assigned_by         UUID,
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium',
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    completed_notes     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Document storage metadata (content in object storage; graph relationships in Neo4j)
CREATE TABLE documents (
    id                  UUID PRIMARY KEY,                     -- Same UUID as Neo4j Document node
    organisation_id     UUID NOT NULL,
    folder_id           UUID,
    title               VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50) NOT NULL,
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
    doc_metadata        JSONB NOT NULL DEFAULT '{}',
    uploaded_by         UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

-- Document annotations
CREATE TABLE document_annotations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL,
    page_number         INTEGER,
    annotation_type     VARCHAR(30) NOT NULL,
    content             TEXT,
    position_data       JSONB,
    colour              VARCHAR(7),
    is_private          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Preclearance requests (transactional detail — relationships in Neo4j)
CREATE TABLE preclearance_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    requester_user_id   UUID NOT NULL,
    security_type       VARCHAR(50) NOT NULL,
    security_name       VARCHAR(300) NOT NULL,
    ticker_symbol       VARCHAR(20),
    transaction_type    VARCHAR(50) NOT NULL,
    requested_shares    DECIMAL(18,4),
    estimated_price     DECIMAL(18,4),
    estimated_value     DECIMAL(18,2),
    transaction_date    DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reviewed_by         UUID,
    reviewed_at         TIMESTAMPTZ,
    review_notes        TEXT,
    approval_expires_at TIMESTAMPTZ,
    executed_at         TIMESTAMPTZ,
    actual_shares       DECIMAL(18,4),
    actual_price        DECIMAL(18,4),
    transaction_details JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Questionnaires and responses (JSONB-driven forms)
CREATE TABLE questionnaires (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    title               VARCHAR(500) NOT NULL,
    questionnaire_type  VARCHAR(50) NOT NULL,
    form_definition     JSONB NOT NULL,
    due_date            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE questionnaire_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    questionnaire_id    UUID NOT NULL REFERENCES questionnaires(id),
    user_id             UUID NOT NULL,
    responses           JSONB NOT NULL DEFAULT '{}',
    review_data         JSONB NOT NULL DEFAULT '{}',
    status              VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    submitted_at        TIMESTAMPTZ,
    reviewed_by         UUID,
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_q_response UNIQUE (questionnaire_id, user_id)
);

-- Secure messaging
CREATE TABLE message_threads (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    subject             VARCHAR(500) NOT NULL,
    classification      VARCHAR(30) NOT NULL DEFAULT 'confidential',
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id           UUID NOT NULL REFERENCES message_threads(id) ON DELETE CASCADE,
    sender_id           UUID NOT NULL,
    body                TEXT NOT NULL,
    body_format         VARCHAR(20) NOT NULL DEFAULT 'plain',
    is_encrypted        BOOLEAN NOT NULL DEFAULT TRUE,
    reply_to_id         UUID REFERENCES messages(id),
    sent_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    edited_at           TIMESTAMPTZ,
    deleted_at          TIMESTAMPTZ
);

-- Notifications
CREATE TABLE notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    user_id             UUID NOT NULL,
    notification_type   VARCHAR(50) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    body                TEXT,
    resource_type       VARCHAR(100),
    resource_id         UUID,
    channel             VARCHAR(20) NOT NULL DEFAULT 'in_app',
    is_read             BOOLEAN NOT NULL DEFAULT FALSE,
    read_at             TIMESTAMPTZ,
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read, created_at);

-- Audit log (append-only, partitioned)
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
    severity            VARCHAR(20) NOT NULL DEFAULT 'info',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Data Synchronisation Strategy

The critical challenge with a polyglot architecture is keeping Neo4j and PostgreSQL in sync.

### Write Flow

```
1. Application receives a write command (e.g., "appoint director to board")

2. PostgreSQL transaction:
   - Write transactional data (if any) to PostgreSQL
   - Write a sync event to the outbox table

3. Outbox processor (CDC):
   - Reads events from outbox table
   - Writes corresponding Neo4j operations
   - Marks outbox events as processed

4. Neo4j update:
   - Creates/updates nodes and relationships
   - Confirms sync
```

```sql
-- Outbox table for reliable sync
CREATE TABLE sync_outbox (
    id                  BIGSERIAL PRIMARY KEY,
    aggregate_type      VARCHAR(100) NOT NULL,                -- 'Board', 'Person', 'Organisation'
    aggregate_id        UUID NOT NULL,
    event_type          VARCHAR(100) NOT NULL,                -- 'BoardMemberAppointed', 'InsiderDesignated'
    event_data          JSONB NOT NULL,
    target_system       VARCHAR(20) NOT NULL DEFAULT 'neo4j',
    status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','processing','completed','failed')),
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at        TIMESTAMPTZ,
    error_message       TEXT
);

CREATE INDEX idx_outbox_pending ON sync_outbox(status, created_at) WHERE status = 'pending';
```

### Consistency Guarantees

- **PostgreSQL is the source of truth** for transactional data (votes, minutes, documents)
- **Neo4j is the source of truth** for relationship queries (but can be rebuilt from PostgreSQL + outbox events)
- **Eventual consistency** between the two stores: typically under 100ms, worst case a few seconds
- **Reconciliation job**: a periodic background job compares entity counts and key relationships between Neo4j and PostgreSQL, alerting on discrepancies

---

## Pros and Cons

### Pros

1. **Natural representation of governance relationships**: Corporate ownership hierarchies, director interlocking, conflict-of-interest networks, and insider designation chains are inherently graph structures. Neo4j represents them as-is, without the impedance mismatch of recursive CTEs or adjacency lists. A query like "find all directors who serve on boards of entities in our corporate group that have active trading restrictions" is a simple graph traversal, not a 5-table JOIN with recursive CTEs.

2. **Real-time conflict detection**: Graph queries can detect complex conflicts of interest that would be computationally expensive or impractical in a relational database. "Does any director have a disclosed interest in any entity that is party to any contract referenced in any resolution proposed at this meeting?" is a single Cypher query.

3. **Visual entity hierarchy**: Neo4j's native graph structure maps directly to the visual corporate structure diagrams that company secretaries work with daily. The database IS the diagram — rendering an interactive organisational chart is a direct graph traversal with no data transformation.

4. **Scalable relationship queries**: As the number of entities, directors, and relationships grows, relational JOIN performance degrades. Graph traversal performance is proportional to the size of the subgraph being traversed, not the total database size. A multinational with 500 subsidiaries and 200 directors benefits enormously from this.

5. **Pattern matching for compliance**: Neo4j's pattern matching capabilities can detect governance anomalies: circular ownership structures, directors serving on too many boards (overboarding), undisclosed related-party connections, and trading during blackout windows by related persons. These are exactly the patterns that regulators investigate.

6. **Best-of-breed for each concern**: PostgreSQL handles what it does best (transactional integrity, content storage, audit logs) while Neo4j handles what it does best (relationship traversal, pattern matching, hierarchy visualization). Neither database is forced into an unnatural role.

### Cons

1. **Operational complexity**: Running two database systems (PostgreSQL + Neo4j) doubles the operational burden: two backup strategies, two monitoring setups, two security configurations, two upgrade processes, two sets of connection pool management. For a security-critical board portal, this is significant.

2. **Data synchronisation risk**: Keeping two databases in sync is an ongoing challenge. The outbox pattern provides eventual consistency, but bugs in the sync layer can cause divergence. A compliance officer querying Neo4j for insider designations while a new designation is still in the PostgreSQL outbox will get stale data. In a compliance-critical system, even brief inconsistency can be problematic.

3. **Team expertise requirements**: The team needs expertise in both PostgreSQL and Neo4j, including Cypher query language, Neo4j data modeling best practices, graph indexing strategies, and Neo4j operational management. This narrows the hiring pool and increases onboarding time.

4. **Transactional boundaries**: Neo4j and PostgreSQL cannot participate in a single distributed transaction without significant complexity (two-phase commit, saga pattern). If the PostgreSQL write succeeds but the Neo4j sync fails, the system is temporarily inconsistent. The outbox pattern mitigates this but adds latency and complexity.

5. **Cost**: Neo4j Enterprise (required for production features like clustering, RBAC, and encryption) requires a commercial licence. The Neo4j Community Edition lacks these features. Aura (Neo4j's managed cloud service) adds significant hosting cost on top of the PostgreSQL hosting cost.

6. **Testing complexity**: Integration tests must verify behavior across both databases. Test setup requires both PostgreSQL and Neo4j instances. CI/CD pipelines become more complex.

7. **Query split decisions**: Every feature requires deciding whether data and queries belong in Neo4j, PostgreSQL, or both. This creates ongoing architectural decision-making overhead that a single-database approach avoids.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|----------------|
| **Graph Database** | Neo4j 5.x Enterprise (or Neo4j Aura Enterprise for managed service) |
| **Relational Database** | PostgreSQL 16+ |
| **Neo4j Driver** | Official Neo4j driver for Node.js, Python, or Java |
| **Sync Layer** | Debezium CDC (PostgreSQL WAL to Neo4j) or custom outbox processor |
| **API Layer** | GraphQL (natural fit for graph + relational hybrid) |
| **Graph Visualization** | Neo4j Browser for admin; Neovis.js or D3.js for end-user entity hierarchy views |
| **Monitoring** | Neo4j Ops Manager + pgwatch2; unified alerting via Grafana |
| **Search** | PostgreSQL full-text search for document content; Neo4j full-text indexes for entity/person name search |
| **Backup** | Neo4j `neo4j-admin database dump` + PostgreSQL `pg_basebackup`, coordinated snapshots |

---

## Migration & Scaling Considerations

### Initial Deployment

- Start with Neo4j Community Edition for development and MVP; upgrade to Enterprise for production
- Use PostgreSQL as the initial source of truth for all data; populate Neo4j from PostgreSQL on startup
- Implement a thin synchronisation layer using the outbox pattern
- Keep the Neo4j schema simple: start with Organisation, Person, Board, Committee nodes and OWNS, MEMBER_OF, SERVES_ON relationships
- Build 3-4 key graph queries first: ownership traversal, conflict detection, board composition analysis

### Growth Phase (100-1,000 organisations)

- Move to Neo4j Aura Enterprise or self-hosted Neo4j Enterprise cluster (3-node causal cluster for HA)
- Implement Debezium for CDC-based synchronisation (more robust than custom outbox)
- Add graph-based analytics: director overboarding reports, ownership concentration analysis, compliance risk scoring
- Cache frequently-accessed graph query results in Redis (e.g., entity hierarchy tree, cached for 5 minutes)

### Scale Phase (1,000+ organisations)

- Neo4j sharding by organisation group (each top-level organisation and its subsidiaries in a dedicated Neo4j database within the same DBMS)
- PostgreSQL horizontal scaling via Citus or read replicas
- Pre-compute complex graph analytics as background jobs and store results in PostgreSQL materialised views
- Consider adding a search engine (Elasticsearch) for cross-database full-text search across both Neo4j entities and PostgreSQL documents

### Disaster Recovery

- Coordinated backup strategy: snapshot Neo4j and PostgreSQL at the same logical point
- Neo4j backup: `neo4j-admin database dump` on a read replica, shipped to S3
- PostgreSQL backup: continuous WAL archiving + periodic base backups
- Recovery procedure: restore PostgreSQL first (source of truth), then rebuild Neo4j graph from PostgreSQL data + outbox replay
- RPO target: < 5 minutes (WAL archiving + Neo4j backup frequency)
- RTO target: < 1 hour (PostgreSQL restore + Neo4j rebuild)
