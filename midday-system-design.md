# Midday System Design

## Table of Contents
1. [Business Context & Problem Statement](#business-context--problem-statement)
2. [High-Level Architecture](#high-level-architecture)
3. [Technology Stack](#technology-stack)
4. [Core Data Model](#core-data-model)
5. [Database Schema Design](#database-schema-design)
6. [API Layer Design](#api-layer-design)
7. [Banking Integration Architecture](#banking-integration-architecture)
8. [Document Processing Flow](#document-processing-flow)
9. [Authentication & Authorization](#authentication--authorization)
10. [Scalability Considerations](#scalability-considerations)

---

## Business Context & Problem Statement

### Problem Domain
Midday addresses the fragmented business management challenges faced by freelancers, contractors, consultants, and solo entrepreneurs. These users typically juggle multiple tools for:
- Time tracking and project management
- Financial transaction monitoring
- Invoice generation and payment tracking  
- Document storage and organization
- Business expense categorization
- Financial reporting and tax preparation

### Target Users
- **Primary**: Solo entrepreneurs, freelancers, independent contractors
- **Secondary**: Small consulting firms, agencies with <10 employees
- **Use Cases**: Project-based billing, expense tracking, financial oversight, client management

### Core Value Proposition
Midday consolidates essential business operations into a single platform, providing:
- **Unified Financial View**: Real-time transaction monitoring across multiple bank accounts
- **Automated Workflows**: Smart document matching, expense categorization
- **Time-to-Invoice Integration**: Seamless flow from time tracking to billing
- **Financial Intelligence**: AI-powered insights and spending analysis

### Business Requirements
- **Security**: Handle sensitive financial data with bank-level encryption
- **Compliance**: Support tax reporting, financial audit trails
- **Integration**: Connect with major banking providers across US, EU, Canada
- **Scalability**: Support team collaboration and multi-user access
- **Real-time**: Live transaction syncing, collaborative editing

---

## High-Level Architecture

### System Overview
Midday follows a modern microservices architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Web Dashboard │   Desktop App   │        Mobile App           │
│   (Next.js)     │   (Tauri)       │        (Expo)              │
└─────────────────┴─────────────────┴─────────────────────────────┘
                           │
                    ┌─────────────┐
                    │  API Gateway │
                    │   (tRPC)     │
                    └─────────────┘
                           │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Dashboard API │   Engine API    │    Background Jobs          │
│   (Vercel)      │   (Fly.io)      │   (Trigger.dev)            │
└─────────────────┴─────────────────┴─────────────────────────────┘
                           │
                    ┌─────────────┐
                    │  Database   │
                    │ (Supabase)  │
                    └─────────────┘
                           │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Banking APIs   │   AI Services   │    External Services        │
│ GoCardless      │   OpenAI        │    Resend (Email)          │
│ Plaid/Teller    │   Mistral       │    Typesense (Search)      │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

### Core Components

#### 1. **Client Applications**
- **Web Dashboard**: Primary business interface (Next.js)
- **Desktop App**: Native cross-platform app (Tauri + Rust)
- **Mobile App**: iOS/Android support (React Native/Expo)

#### 2. **API Layer**
- **tRPC Router**: Type-safe API contracts
- **REST Endpoints**: External integration points
- **Real-time**: WebSocket connections via Supabase

#### 3. **Core Services**
- **Dashboard Service**: Main business logic, user workflows
- **Engine Service**: Financial data aggregation, bank connections
- **Background Jobs**: Async processing, scheduled tasks

#### 4. **Data Layer**
- **Primary Database**: PostgreSQL via Supabase
- **File Storage**: Supabase Storage for documents/attachments
- **Search Index**: Typesense for document and transaction search
- **Cache**: Redis for session management and API caching

---

## Technology Stack

### Frontend Technologies
| Layer | Technology | Purpose |
|-------|------------|---------|
| **Web App** | Next.js 14+ | Server-side rendering, routing |
| **UI Framework** | React 18+ | Component-based UI |
| **Styling** | TailwindCSS + Shadcn/ui | Design system, responsive layouts |
| **State Management** | Zustand | Client-side state management |
| **Forms** | React Hook Form + Zod | Form validation and handling |
| **Desktop** | Tauri + Rust | Native desktop application |
| **Mobile** | Expo + React Native | Cross-platform mobile |

### Backend Technologies
| Layer | Technology | Purpose |
|-------|------------|---------|
| **API Framework** | tRPC | Type-safe API layer |
| **Runtime** | Node.js + Bun | JavaScript runtime, package management |
| **Database** | PostgreSQL | Primary data storage |
| **ORM** | Drizzle | Type-safe database queries |
| **Authentication** | Supabase Auth | User authentication, JWT tokens |
| **Background Jobs** | Trigger.dev | Async task processing |
| **File Storage** | Supabase Storage | Document and image storage |

### Infrastructure & DevOps
| Layer | Technology | Purpose |
|-------|------------|---------|
| **Web Hosting** | Vercel | Dashboard and website deployment |
| **API Hosting** | Fly.io | Engine API deployment |
| **Database** | Supabase | Managed PostgreSQL + Auth |
| **Monitoring** | OpenPanel + Baselime | Analytics and observability |
| **CI/CD** | GitHub Actions | Automated testing and deployment |
| **Search** | Typesense | Full-text search capabilities |

### External Integrations
| Service | Provider | Purpose |
|---------|----------|---------|
| **Banking (EU)** | GoCardless | Bank account connections |
| **Banking (US/CA)** | Plaid, Teller | Transaction data, account access |
| **Email** | Gmail, Outlook APIs | Inbox document processing |
| **Payments** | Polar | Invoice payment processing |
| **AI/ML** | OpenAI, Mistral, Gemini | Document analysis, categorization |
| **Notifications** | Resend | Transactional email delivery |

---

## Core Data Model

### Entity Relationship Overview
The Midday data model centers around **Teams** as the primary tenant boundary, with key business entities:

```
Team (Tenant)
├── Users (Members)
├── Bank Connections
│   └── Bank Accounts
│       └── Transactions
├── Customers
│   └── Invoices
├── Tracker Projects
│   └── Tracker Entries
├── Documents (Vault)
└── Tags (Categories)
```

### Primary Entities

#### **1. Team (Multi-tenant Core)**
```typescript
Team {
  id: uuid
  name: string
  logo_url?: string
  inbox_email: string         // team@inbox.midday.ai
  base_currency: string       // USD, EUR, etc.
  created_at: timestamp
}
```
- **Purpose**: Primary tenant boundary for multi-tenancy
- **Key Features**: Each team has isolated data, custom inbox email, currency settings

#### **2. User Management**
```typescript
User {
  id: uuid
  email: string
  full_name?: string
  avatar_url?: string
  locale: string              // en, sv, etc.
  timezone: string
}

UserTeam {
  user_id: uuid -> User.id
  team_id: uuid -> Team.id
  role: enum ['owner', 'admin', 'member']
  created_at: timestamp
}
```
- **Purpose**: User identity and team membership management
- **Key Features**: Role-based access control, multi-team support

#### **3. Banking & Transactions**
```typescript
BankConnection {
  id: uuid
  team_id: uuid -> Team.id
  provider: enum ['gocardless', 'plaid', 'teller', 'enablebanking']
  access_token: encrypted_text
  status: enum ['connected', 'disconnected', 'unknown']
  expires_at?: timestamp
}

BankAccount {
  id: uuid
  team_id: uuid -> Team.id
  connection_id: uuid -> BankConnection.id
  account_id: string          // External provider account ID
  name: string
  currency: string
  type: enum ['depository', 'credit', 'other_asset', 'loan']
  balance: decimal
  enabled: boolean
}

Transaction {
  id: uuid
  team_id: uuid -> Team.id
  bank_account_id: uuid -> BankAccount.id
  amount: decimal
  currency: string
  date: date
  description: string
  merchant_name?: string
  category_slug?: string      // Auto-categorization
  status: enum ['posted', 'pending']
  balance: decimal            // Account balance after transaction
}
```
- **Purpose**: Core financial data management
- **Key Features**: Multi-provider support, real-time balance tracking, automatic categorization

#### **4. Customer & Invoice Management**
```typescript
Customer {
  id: uuid
  team_id: uuid -> Team.id
  name: string
  email?: string
  phone?: string
  address_line_1?: string
  city?: string
  country?: string
  currency: string
  vat_number?: string
  created_at: timestamp
}

Invoice {
  id: uuid
  team_id: uuid -> Team.id
  customer_id?: uuid -> Customer.id
  invoice_number: string      // Auto-generated or custom
  status: enum ['draft', 'sent', 'paid', 'overdue', 'cancelled']
  issue_date: date
  due_date: date
  amount: decimal
  tax: decimal
  currency: string
  template: jsonb             // Invoice design/layout
  sent_at?: timestamp
  paid_at?: timestamp
}
```
- **Purpose**: Client billing and payment tracking
- **Key Features**: Customizable invoice templates, payment status tracking

#### **5. Time Tracking**
```typescript
TrackerProject {
  id: uuid
  team_id: uuid -> Team.id
  name: string
  description?: string
  status: enum ['active', 'completed', 'archived']
  rate?: decimal              // Hourly rate
  currency?: string
  customer_id?: uuid -> Customer.id
  created_at: timestamp
}

TrackerEntry {
  id: uuid
  team_id: uuid -> Team.id
  project_id: uuid -> TrackerProject.id
  user_id: uuid -> User.id
  description?: string
  start: timestamp
  stop?: timestamp            // NULL for active tracking
  duration?: integer          // Seconds, calculated when stopped
  created_at: timestamp
}
```
- **Purpose**: Project-based time tracking for billing
- **Key Features**: Live timer support, project-based organization, billable hour calculation

#### **6. Document Management (Vault + Inbox)**
```typescript
Document {
  id: uuid
  team_id: uuid -> Team.id
  name: string
  content_type: string        // application/pdf, image/jpeg, etc.
  size: bigint
  path_tokens: string[]       // Folder hierarchy
  created_at: timestamp
}

Inbox {
  id: uuid
  team_id: uuid -> Team.id
  display_name: string
  file_name?: string
  file_path?: string
  content_type?: string
  status: enum ['processing', 'pending', 'archived', 'new', 'analyzing', 'suggested_match', 'no_match', 'done', 'deleted']
  transaction_id?: uuid -> Transaction.id  // Matched transaction
  amount?: decimal
  currency?: string
  date?: date
  created_at: timestamp
}
```
- **Purpose**: Document storage and intelligent transaction matching
- **Key Features**: Hierarchical organization, AI-powered invoice/receipt matching

### Cross-Cutting Entities

#### **Tags & Categories**
```typescript
Tag {
  id: uuid
  team_id: uuid -> Team.id
  name: string
  color?: string
  created_at: timestamp
}

// Junction tables for many-to-many relationships
TransactionTag {
  transaction_id: uuid -> Transaction.id
  tag_id: uuid -> Tag.id
}

DocumentTag {
  document_id: uuid -> Document.id  
  tag_id: uuid -> Tag.id
}
```

#### **Notifications & Settings**
```typescript
Notification {
  id: uuid
  user_id: uuid -> User.id
  team_id: uuid -> Team.id
  type: string                // 'transaction_created', 'invoice_paid', etc.
  data: jsonb                 // Event-specific payload
  read_at?: timestamp
  created_at: timestamp
}
```

---

## Database Schema Design

### Schema Architecture Principles

#### **Multi-Tenancy Strategy**
- **Team-based Partitioning**: Every major entity includes `team_id` for data isolation
- **Row Level Security (RLS)**: Supabase policies enforce tenant boundaries at the database level
- **Shared Schema**: Single database with logical separation (vs. schema-per-tenant)

#### **Data Integrity Constraints**
```sql
-- Example: Teams table with core constraints
CREATE TABLE teams (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT NOT NULL,
    inbox_email TEXT UNIQUE NOT NULL,
    base_currency TEXT NOT NULL DEFAULT 'USD',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS policy example
CREATE POLICY "Team members can view their team data" ON teams
    FOR SELECT USING (auth.uid() IN (
        SELECT user_id FROM user_teams WHERE team_id = teams.id
    ));
```

### Key Database Design Patterns

#### **1. Soft Deletes & Audit Trails**
```sql
-- Common audit columns across tables
ALTER TABLE transactions ADD COLUMN deleted_at TIMESTAMPTZ;
ALTER TABLE transactions ADD COLUMN created_by UUID REFERENCES users(id);
ALTER TABLE transactions ADD COLUMN updated_by UUID REFERENCES users(id);

-- Soft delete view
CREATE VIEW active_transactions AS 
SELECT * FROM transactions WHERE deleted_at IS NULL;
```

#### **2. Encrypted Sensitive Data**
```sql
-- Bank connection tokens (encrypted at application level)
CREATE TABLE bank_connections (
    id UUID PRIMARY KEY,
    team_id UUID NOT NULL REFERENCES teams(id),
    provider bank_providers NOT NULL,
    access_token_encrypted TEXT NOT NULL,  -- AES encrypted
    refresh_token_encrypted TEXT,
    expires_at TIMESTAMPTZ,
    status connection_status DEFAULT 'unknown'
);
```

#### **3. JSONB for Flexible Schema**
```sql
-- Invoice templates with flexible structure
CREATE TABLE invoices (
    id UUID PRIMARY KEY,
    team_id UUID NOT NULL REFERENCES teams(id),
    template JSONB NOT NULL DEFAULT '{}',  -- Invoice design/layout
    line_items JSONB NOT NULL DEFAULT '[]', -- Dynamic invoice items
    metadata JSONB DEFAULT '{}'  -- Additional invoice data
);

-- GIN index for JSONB queries
CREATE INDEX idx_invoices_template_gin ON invoices USING GIN (template);
CREATE INDEX idx_invoices_metadata_gin ON invoices USING GIN (metadata);
```

### Performance Optimization Strategies

#### **Database Indexes**
```sql
-- High-frequency query indexes
CREATE INDEX idx_transactions_team_date ON transactions(team_id, date DESC);
CREATE INDEX idx_transactions_account_status ON transactions(bank_account_id, status);
CREATE INDEX idx_tracker_entries_project_start ON tracker_entries(project_id, start DESC);

-- Partial indexes for common filters
CREATE INDEX idx_transactions_active ON transactions(team_id, date) 
    WHERE deleted_at IS NULL;
    
-- Composite indexes for complex queries
CREATE INDEX idx_inbox_status_team ON inbox(status, team_id, created_at DESC)
    WHERE status IN ('new', 'pending', 'analyzing');
```

#### **Database Partitioning (Future Scale)**
```sql
-- Time-based partitioning for transaction history
CREATE TABLE transactions (
    -- ... columns
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE transactions_2024_01 PARTITION OF transactions
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### Data Consistency & Integrity

#### **Foreign Key Relationships**
```sql
-- Core entity relationships with cascading
ALTER TABLE bank_accounts 
    ADD CONSTRAINT fk_bank_accounts_team 
    FOREIGN KEY (team_id) REFERENCES teams(id) ON DELETE CASCADE;

ALTER TABLE transactions 
    ADD CONSTRAINT fk_transactions_account 
    FOREIGN KEY (bank_account_id) REFERENCES bank_accounts(id) ON DELETE RESTRICT;
```

#### **Check Constraints for Business Rules**
```sql
-- Business logic validation at DB level
ALTER TABLE transactions 
    ADD CONSTRAINT chk_transaction_amount 
    CHECK (amount != 0);

ALTER TABLE invoices 
    ADD CONSTRAINT chk_invoice_dates 
    CHECK (due_date >= issue_date);

ALTER TABLE tracker_entries 
    ADD CONSTRAINT chk_tracker_duration 
    CHECK (stop IS NULL OR stop > start);
```

#### **Triggers for Derived Data**
```sql
-- Auto-calculate tracker entry duration
CREATE OR REPLACE FUNCTION calculate_tracker_duration()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.stop IS NOT NULL AND NEW.start IS NOT NULL THEN
        NEW.duration := EXTRACT(EPOCH FROM (NEW.stop - NEW.start));
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_tracker_duration
    BEFORE INSERT OR UPDATE ON tracker_entries
    FOR EACH ROW EXECUTE FUNCTION calculate_tracker_duration();
```

### Search & Full-Text Capabilities

#### **PostgreSQL Full-Text Search**
```sql
-- Document search vectors
ALTER TABLE documents ADD COLUMN search_vector TSVECTOR;

-- Auto-update search index
CREATE OR REPLACE FUNCTION update_document_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
                        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_document_search_update
    BEFORE INSERT OR UPDATE ON documents
    FOR EACH ROW EXECUTE FUNCTION update_document_search_vector();

-- GIN index for search performance
CREATE INDEX idx_documents_search_gin ON documents USING GIN (search_vector);
```

#### **External Search Integration (Typesense)**
- **Sync Strategy**: Database triggers push changes to search index
- **Search Scope**: Documents, transactions, customers, projects
- **Real-time Updates**: WebSocket notifications on search result changes

### Backup & Disaster Recovery

#### **Point-in-Time Recovery**
- **Supabase Managed**: Automatic daily backups with 7-day retention
- **Custom Snapshots**: Weekly full database exports to S3
- **Migration Scripts**: Version-controlled schema changes via Drizzle migrations

#### **High Availability Setup**
```sql
-- Read replicas for reporting queries
-- (Managed by Supabase in production)
CREATE PUBLICATION midday_replica FOR ALL TABLES;

-- Connection pooling configuration
-- Max connections: 100 (Supabase default)
-- Pool mode: Transaction-level pooling for API requests
```

---