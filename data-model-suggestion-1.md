# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Insurance Policy Management · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form relational design, giving every insurance concept its own table with explicit foreign key relationships. Every coverage, endorsement, rating factor, and billing installment is a discrete row in a dedicated table, enforcing referential integrity at the database level. This approach mirrors how regulators and auditors think about insurance data: every field is typed, every relationship is explicit, and every constraint is enforced.

The ACORD data standards themselves are fundamentally entity-relational in structure — policies contain coverages, coverages contain limits and deductibles, claims reference policies and coverages. A normalized model maps naturally to ACORD XML import/export because each XML element corresponds to a table. Guidewire's PolicyCenter and BriteCore both use heavily normalized relational schemas internally, validating this as the industry default.

The trade-off is table count and join complexity. A single policy view may require joining 8-12 tables. But for a regulatory-heavy domain where data integrity, auditability, and standards compliance are paramount, this is the proven approach.

**Best for:** Carriers prioritising regulatory compliance, ACORD standards alignment, and long-term data integrity over development speed.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Natural mapping to ACORD XML/JSON schemas
- Pro: Easy to audit — every field has a defined type and constraint
- Pro: Well-understood by insurance domain experts and DBAs
- Con: High table count (~60-80 tables) increases join complexity
- Con: Schema changes require migrations for every new field
- Con: Jurisdiction-specific fields lead to many nullable columns or subtypes
- Con: Complex queries for common views (full policy summary)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ACORD P&C Data Standards | Direct table mapping — Policy, Coverage, Claim, ClaimActivity mirror ACORD XML element hierarchy |
| ACORD Next-Gen Digital Standards | JSON API response shapes derived directly from table joins |
| ISO 3166 | `jurisdictions` reference table for country/subdivision codes |
| ISO 20022 | `payment_transactions` table fields align with ISO 20022 payment instruction messages |
| NAIC Model Laws | `regulatory_filings` table supports state-specific filing workflows |
| PCI DSS v4.0.1 | Payment card data isolated in `payment_methods` with tokenised storage |
| ISO 27001 | Row-level audit columns (`created_by`, `updated_by`, timestamps) on every table |
| HL7 FHIR R5 | `health_coverage` table fields map to FHIR Coverage and Claim resources |

---

## Core Identity & Multi-Tenancy

```sql
-- Tenant isolation for multi-carrier deployment
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tenant_type VARCHAR(50) NOT NULL CHECK (tenant_type IN ('carrier', 'mga', 'broker', 'reinsurer')),
    naic_code VARCHAR(20),                    -- NAIC company code (US carriers)
    lei VARCHAR(20),                          -- ISO 17442 Legal Entity Identifier
    jurisdiction_country CHAR(2) NOT NULL,    -- ISO 3166-1 alpha-2
    settings JSONB DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    role VARCHAR(50) NOT NULL CHECK (role IN ('admin', 'underwriter', 'claims_adjuster', 'agent', 'customer_service', 'billing', 'auditor')),
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant_role ON users(tenant_id, role);
```

## Party Management (Policyholders, Agents, Third Parties)

```sql
-- Unified party model — a party can be a person or an organisation
CREATE TABLE parties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    party_type VARCHAR(20) NOT NULL CHECK (party_type IN ('individual', 'organisation')),
    -- Individual fields
    first_name VARCHAR(100),
    middle_name VARCHAR(100),
    last_name VARCHAR(100),
    date_of_birth DATE,
    ssn_last4 VARCHAR(4),                     -- Last 4 of SSN (masked storage)
    gender VARCHAR(20),
    -- Organisation fields
    organisation_name VARCHAR(255),
    tax_id VARCHAR(50),                       -- EIN or equivalent
    lei VARCHAR(20),                          -- ISO 17442 Legal Entity Identifier
    -- Common fields
    email VARCHAR(255),
    phone VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parties_tenant ON parties(tenant_id);
CREATE INDEX idx_parties_email ON parties(tenant_id, email);
CREATE INDEX idx_parties_org_name ON parties(tenant_id, organisation_name) WHERE party_type = 'organisation';

CREATE TABLE party_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES parties(id),
    address_type VARCHAR(30) NOT NULL CHECK (address_type IN ('mailing', 'billing', 'property', 'business')),
    line1 VARCHAR(255) NOT NULL,
    line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country CHAR(2) NOT NULL DEFAULT 'US',    -- ISO 3166-1
    is_primary BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_addresses_party ON party_addresses(party_id);

-- Agents and agencies
CREATE TABLE agencies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    party_id UUID NOT NULL REFERENCES parties(id),   -- Links to organisation party
    agency_code VARCHAR(50) NOT NULL,
    agency_name VARCHAR(255) NOT NULL,
    license_number VARCHAR(100),
    license_state CHAR(2),                             -- ISO 3166-2 subdivision
    appointment_status VARCHAR(30) NOT NULL DEFAULT 'active',
    commission_schedule_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, agency_code)
);

CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    party_id UUID NOT NULL REFERENCES parties(id),   -- Links to individual party
    agency_id UUID REFERENCES agencies(id),
    user_id UUID REFERENCES users(id),               -- Portal login
    agent_code VARCHAR(50) NOT NULL,
    license_number VARCHAR(100),
    license_state CHAR(2),
    npn VARCHAR(20),                                  -- National Producer Number (US)
    appointment_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, agent_code)
);
```

## Product & Rating Configuration

```sql
-- Insurance product definitions (e.g., "Homeowners HO-3", "Commercial General Liability")
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    product_code VARCHAR(50) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    line_of_business VARCHAR(50) NOT NULL CHECK (line_of_business IN (
        'personal_auto', 'commercial_auto', 'homeowners', 'renters',
        'commercial_property', 'general_liability', 'professional_liability',
        'workers_comp', 'umbrella', 'inland_marine', 'cyber', 'health', 'life'
    )),
    iso_class_code VARCHAR(20),               -- ISO classification code
    description TEXT,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_code)
);

-- Coverage types available for each product
CREATE TABLE coverage_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    coverage_code VARCHAR(50) NOT NULL,
    coverage_name VARCHAR(255) NOT NULL,
    description TEXT,
    is_mandatory BOOLEAN NOT NULL DEFAULT false,
    default_limit NUMERIC(15,2),
    default_deductible NUMERIC(15,2),
    min_limit NUMERIC(15,2),
    max_limit NUMERIC(15,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_types_product ON coverage_types(product_id);

-- Rating tables — base rates, factors, and rules
CREATE TABLE rating_tables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    product_id UUID NOT NULL REFERENCES products(id),
    table_name VARCHAR(255) NOT NULL,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    jurisdiction CHAR(2),                     -- State/province; NULL = all
    version INT NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rating_factors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rating_table_id UUID NOT NULL REFERENCES rating_tables(id),
    factor_name VARCHAR(100) NOT NULL,        -- e.g., 'territory', 'age_band', 'credit_tier'
    factor_value VARCHAR(255) NOT NULL,       -- e.g., 'Territory 5', '25-34', 'Excellent'
    multiplier NUMERIC(10,6) NOT NULL DEFAULT 1.000000,
    flat_amount NUMERIC(15,2) DEFAULT 0.00,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rating_factors_table ON rating_factors(rating_table_id);
```

## Policy Management

```sql
-- Core policy record
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    policy_number VARCHAR(50) NOT NULL,
    product_id UUID NOT NULL REFERENCES products(id),
    policyholder_id UUID NOT NULL REFERENCES parties(id),
    agent_id UUID REFERENCES agents(id),
    agency_id UUID REFERENCES agencies(id),
    status VARCHAR(30) NOT NULL CHECK (status IN (
        'quote', 'application', 'bound', 'issued', 'active',
        'pending_renewal', 'renewed', 'cancelled', 'expired', 'non_renewed'
    )),
    effective_date DATE NOT NULL,
    expiration_date DATE NOT NULL,
    cancellation_date DATE,
    cancellation_reason VARCHAR(100),
    original_inception_date DATE NOT NULL,    -- First policy in chain
    term_months INT NOT NULL DEFAULT 12,
    prior_policy_id UUID REFERENCES policies(id),   -- Renewal chain
    jurisdiction CHAR(2) NOT NULL,            -- ISO 3166-2 state/province
    total_premium NUMERIC(15,2),
    currency CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    underwriter_id UUID REFERENCES users(id),
    bound_at TIMESTAMPTZ,
    issued_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, policy_number)
);

CREATE INDEX idx_policies_tenant_status ON policies(tenant_id, status);
CREATE INDEX idx_policies_policyholder ON policies(policyholder_id);
CREATE INDEX idx_policies_agent ON policies(agent_id);
CREATE INDEX idx_policies_dates ON policies(effective_date, expiration_date);
CREATE INDEX idx_policies_renewal_chain ON policies(prior_policy_id);

-- Policy coverages (specific limits/deductibles per coverage on a policy)
CREATE TABLE policy_coverages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    coverage_type_id UUID NOT NULL REFERENCES coverage_types(id),
    coverage_limit NUMERIC(15,2) NOT NULL,
    deductible NUMERIC(15,2) NOT NULL DEFAULT 0,
    premium NUMERIC(15,2) NOT NULL,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_coverages_policy ON policy_coverages(policy_id);

-- Endorsements (mid-term policy changes)
CREATE TABLE endorsements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    endorsement_number INT NOT NULL,
    endorsement_type VARCHAR(50) NOT NULL CHECK (endorsement_type IN (
        'addition', 'deletion', 'modification', 'correction', 'renewal', 'cancellation'
    )),
    description TEXT NOT NULL,
    effective_date DATE NOT NULL,
    premium_change NUMERIC(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'pending_approval', 'approved', 'issued', 'rejected')),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, endorsement_number)
);

CREATE INDEX idx_endorsements_policy ON endorsements(policy_id);

-- Named insureds and additional interests on a policy
CREATE TABLE policy_interests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    party_id UUID NOT NULL REFERENCES parties(id),
    interest_type VARCHAR(50) NOT NULL CHECK (interest_type IN (
        'named_insured', 'additional_insured', 'loss_payee', 'mortgagee',
        'certificate_holder', 'lienholder'
    )),
    effective_date DATE NOT NULL,
    expiration_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_interests_policy ON policy_interests(policy_id);
```

## Insured Assets & Risk Items

```sql
-- Risk items insured under a policy (vehicles, properties, equipment)
CREATE TABLE insured_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    item_type VARCHAR(30) NOT NULL CHECK (item_type IN (
        'vehicle', 'property', 'equipment', 'watercraft', 'aircraft', 'scheduled_item'
    )),
    item_number INT NOT NULL,
    description TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insured_items_policy ON insured_items(policy_id);

-- Vehicle-specific details
CREATE TABLE insured_vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    insured_item_id UUID NOT NULL UNIQUE REFERENCES insured_items(id),
    vin VARCHAR(17),
    year INT,
    make VARCHAR(100),
    model VARCHAR(100),
    body_type VARCHAR(50),
    usage_type VARCHAR(50),               -- 'pleasure', 'commute', 'business'
    annual_mileage INT,
    garaging_zip VARCHAR(10),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Property-specific details
CREATE TABLE insured_properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    insured_item_id UUID NOT NULL UNIQUE REFERENCES insured_items(id),
    property_type VARCHAR(50),            -- 'single_family', 'condo', 'commercial'
    construction_type VARCHAR(50),        -- 'frame', 'masonry', 'fire_resistive'
    year_built INT,
    square_footage INT,
    number_of_stories INT,
    roof_type VARCHAR(50),
    replacement_cost NUMERIC(15,2),
    protection_class VARCHAR(10),         -- ISO fire protection class
    territory_code VARCHAR(20),           -- ISO territory
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Drivers (for auto policies)
CREATE TABLE policy_drivers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    party_id UUID NOT NULL REFERENCES parties(id),
    driver_number INT NOT NULL,
    license_number VARCHAR(50),
    license_state CHAR(2),
    date_licensed DATE,
    driver_status VARCHAR(30) NOT NULL DEFAULT 'rated',  -- 'rated', 'excluded', 'listed_only'
    good_driver_discount BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_drivers_policy ON policy_drivers(policy_id);
```

## Underwriting

```sql
CREATE TABLE underwriting_submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    policy_id UUID NOT NULL REFERENCES policies(id),
    submission_type VARCHAR(30) NOT NULL CHECK (submission_type IN ('new_business', 'renewal', 'endorsement')),
    status VARCHAR(30) NOT NULL CHECK (status IN (
        'submitted', 'in_review', 'info_requested', 'approved', 'declined', 'referred'
    )),
    assigned_to UUID REFERENCES users(id),
    risk_score NUMERIC(5,2),                  -- AI-generated risk score (0-100)
    risk_tier VARCHAR(20),                    -- 'preferred', 'standard', 'substandard'
    premium_indication NUMERIC(15,2),
    notes TEXT,
    submitted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    decision_at TIMESTAMPTZ,
    decided_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uw_submissions_policy ON underwriting_submissions(policy_id);
CREATE INDEX idx_uw_submissions_status ON underwriting_submissions(tenant_id, status);

-- Underwriting questions and answers
CREATE TABLE underwriting_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    question_code VARCHAR(50) NOT NULL,
    question_text TEXT NOT NULL,
    answer_type VARCHAR(20) NOT NULL CHECK (answer_type IN ('yes_no', 'text', 'number', 'date', 'select')),
    is_required BOOLEAN NOT NULL DEFAULT true,
    display_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE underwriting_answers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submission_id UUID NOT NULL REFERENCES underwriting_submissions(id),
    question_id UUID NOT NULL REFERENCES underwriting_questions(id),
    answer_value TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uw_answers_submission ON underwriting_answers(submission_id);
```

## Claims Management

```sql
CREATE TABLE claims (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    claim_number VARCHAR(50) NOT NULL,
    policy_id UUID NOT NULL REFERENCES policies(id),
    claimant_id UUID NOT NULL REFERENCES parties(id),
    status VARCHAR(30) NOT NULL CHECK (status IN (
        'fnol', 'open', 'under_investigation', 'reserved', 'approved',
        'denied', 'closed', 'reopened', 'subrogation'
    )),
    loss_date DATE NOT NULL,
    reported_date DATE NOT NULL,
    loss_type VARCHAR(50) NOT NULL,           -- 'collision', 'fire', 'theft', 'liability', 'weather'
    loss_description TEXT,
    loss_location TEXT,
    loss_jurisdiction CHAR(2),
    assigned_adjuster_id UUID REFERENCES users(id),
    total_incurred NUMERIC(15,2) DEFAULT 0,
    total_paid NUMERIC(15,2) DEFAULT 0,
    total_reserved NUMERIC(15,2) DEFAULT 0,
    deductible_applied NUMERIC(15,2) DEFAULT 0,
    fraud_score NUMERIC(5,2),                 -- AI fraud detection score (0-100)
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, claim_number)
);

CREATE INDEX idx_claims_policy ON claims(policy_id);
CREATE INDEX idx_claims_tenant_status ON claims(tenant_id, status);
CREATE INDEX idx_claims_adjuster ON claims(assigned_adjuster_id);
CREATE INDEX idx_claims_loss_date ON claims(loss_date);

-- Claim coverages (which policy coverages apply to this claim)
CREATE TABLE claim_coverages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID NOT NULL REFERENCES claims(id),
    policy_coverage_id UUID NOT NULL REFERENCES policy_coverages(id),
    reserve_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    paid_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_coverages_claim ON claim_coverages(claim_id);

-- Claim activities / diary entries
CREATE TABLE claim_activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID NOT NULL REFERENCES claims(id),
    activity_type VARCHAR(50) NOT NULL CHECK (activity_type IN (
        'note', 'phone_call', 'email', 'inspection', 'appraisal',
        'payment', 'reserve_change', 'status_change', 'document_received',
        'subrogation_action', 'litigation_update'
    )),
    description TEXT NOT NULL,
    performed_by UUID REFERENCES users(id),
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_activities_claim ON claim_activities(claim_id);

-- Claim payments
CREATE TABLE claim_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID NOT NULL REFERENCES claims(id),
    claim_coverage_id UUID REFERENCES claim_coverages(id),
    payee_id UUID NOT NULL REFERENCES parties(id),
    payment_type VARCHAR(30) NOT NULL CHECK (payment_type IN (
        'indemnity', 'expense', 'medical', 'legal', 'salvage_recovery', 'subrogation_recovery'
    )),
    amount NUMERIC(15,2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    check_number VARCHAR(50),
    payment_method VARCHAR(20),               -- 'check', 'eft', 'wire'
    payment_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_payments_claim ON claim_payments(claim_id);
```

## Billing & Payments

```sql
CREATE TABLE billing_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    account_number VARCHAR(50) NOT NULL,
    policyholder_id UUID NOT NULL REFERENCES parties(id),
    billing_plan VARCHAR(30) NOT NULL CHECK (billing_plan IN (
        'annual', 'semi_annual', 'quarterly', 'monthly', 'pay_as_you_go'
    )),
    payment_method_id UUID,
    balance_due NUMERIC(15,2) NOT NULL DEFAULT 0,
    next_due_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, account_number)
);

CREATE INDEX idx_billing_accounts_policyholder ON billing_accounts(policyholder_id);

-- Invoices generated for premium installments
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_account_id UUID NOT NULL REFERENCES billing_accounts(id),
    policy_id UUID NOT NULL REFERENCES policies(id),
    invoice_number VARCHAR(50) NOT NULL,
    invoice_date DATE NOT NULL,
    due_date DATE NOT NULL,
    amount_due NUMERIC(15,2) NOT NULL,
    amount_paid NUMERIC(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'sent', 'paid', 'partial', 'overdue', 'cancelled', 'written_off')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_billing_account ON invoices(billing_account_id);
CREATE INDEX idx_invoices_policy ON invoices(policy_id);
CREATE INDEX idx_invoices_due_date ON invoices(due_date, status);

-- Payment methods (PCI DSS: only store tokens, never raw card data)
CREATE TABLE payment_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES parties(id),
    method_type VARCHAR(20) NOT NULL CHECK (method_type IN ('card', 'ach', 'wire', 'check')),
    token VARCHAR(255),                       -- Payment processor token (Stripe, etc.)
    last_four VARCHAR(4),
    expiry_month INT,
    expiry_year INT,
    bank_name VARCHAR(100),
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Payment transactions
CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID REFERENCES invoices(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('payment', 'refund', 'chargeback', 'adjustment')),
    amount NUMERIC(15,2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    processor_reference VARCHAR(255),         -- External payment processor ID
    iso20022_msg_id VARCHAR(35),              -- ISO 20022 message identifier
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'completed', 'failed', 'reversed')),
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_transactions_invoice ON payment_transactions(invoice_id);
```

## Reinsurance

```sql
CREATE TABLE reinsurance_treaties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    treaty_number VARCHAR(50) NOT NULL,
    treaty_name VARCHAR(255) NOT NULL,
    treaty_type VARCHAR(30) NOT NULL CHECK (treaty_type IN (
        'quota_share', 'surplus', 'excess_of_loss', 'catastrophe', 'facultative', 'stop_loss'
    )),
    reinsurer_id UUID NOT NULL REFERENCES parties(id),
    effective_date DATE NOT NULL,
    expiration_date DATE NOT NULL,
    cession_percentage NUMERIC(5,2),          -- For proportional treaties
    retention_amount NUMERIC(15,2),           -- For excess-of-loss
    limit_amount NUMERIC(15,2),
    commission_rate NUMERIC(5,4),             -- Ceding commission rate
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, treaty_number)
);

-- Cessions — individual policy premiums ceded to reinsurers
CREATE TABLE reinsurance_cessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    treaty_id UUID NOT NULL REFERENCES reinsurance_treaties(id),
    policy_id UUID NOT NULL REFERENCES policies(id),
    ceded_premium NUMERIC(15,2) NOT NULL,
    ceding_commission NUMERIC(15,2) NOT NULL DEFAULT 0,
    cession_date DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cessions_treaty ON reinsurance_cessions(treaty_id);
CREATE INDEX idx_cessions_policy ON reinsurance_cessions(policy_id);
```

## Documents & Communications

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    entity_type VARCHAR(30) NOT NULL,         -- 'policy', 'claim', 'endorsement', 'submission'
    entity_id UUID NOT NULL,
    document_type VARCHAR(50) NOT NULL,       -- 'dec_page', 'application', 'endorsement_form', 'claim_photo', 'correspondence'
    file_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path VARCHAR(500) NOT NULL,       -- S3/GCS path
    version INT NOT NULL DEFAULT 1,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_entity ON documents(entity_type, entity_id);

-- Communication log (emails, letters, SMS)
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    entity_type VARCHAR(30) NOT NULL,
    entity_id UUID NOT NULL,
    channel VARCHAR(20) NOT NULL CHECK (channel IN ('email', 'sms', 'letter', 'portal_message', 'phone')),
    direction VARCHAR(10) NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    recipient_id UUID REFERENCES parties(id),
    subject VARCHAR(500),
    body TEXT,
    sent_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_communications_entity ON communications(entity_type, entity_id);
```

## Audit & Regulatory

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID REFERENCES users(id),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(30) NOT NULL CHECK (action IN ('create', 'update', 'delete', 'view', 'approve', 'reject', 'export')),
    field_changes JSONB,                      -- {"field": {"old": "x", "new": "y"}, ...}
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant_time ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at);

-- Regulatory filings (NAIC annual statements, rate filings, etc.)
CREATE TABLE regulatory_filings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    filing_type VARCHAR(50) NOT NULL,         -- 'rate_filing', 'form_filing', 'annual_statement', 'quarterly_financial'
    jurisdiction CHAR(2) NOT NULL,
    filing_reference VARCHAR(100),
    status VARCHAR(30) NOT NULL CHECK (status IN ('draft', 'submitted', 'approved', 'rejected', 'withdrawn')),
    submitted_at TIMESTAMPTZ,
    response_at TIMESTAMPTZ,
    response_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_regulatory_filings_tenant ON regulatory_filings(tenant_id, filing_type);
```

## Commission Management

```sql
CREATE TABLE commission_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    schedule_name VARCHAR(255) NOT NULL,
    product_id UUID REFERENCES products(id),
    new_business_rate NUMERIC(5,4) NOT NULL,  -- e.g., 0.1500 = 15%
    renewal_rate NUMERIC(5,4) NOT NULL,
    contingency_rate NUMERIC(5,4) DEFAULT 0,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE commission_statements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES agents(id),
    statement_period_start DATE NOT NULL,
    statement_period_end DATE NOT NULL,
    total_earned NUMERIC(15,2) NOT NULL DEFAULT 0,
    total_paid NUMERIC(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE commission_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id UUID NOT NULL REFERENCES commission_statements(id),
    policy_id UUID NOT NULL REFERENCES policies(id),
    commission_type VARCHAR(20) NOT NULL,     -- 'new_business', 'renewal', 'contingency'
    premium_basis NUMERIC(15,2) NOT NULL,
    rate NUMERIC(5,4) NOT NULL,
    amount NUMERIC(15,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commission_entries_statement ON commission_entries(statement_id);
CREATE INDEX idx_commission_entries_policy ON commission_entries(policy_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-tenancy | 2 | tenants, users |
| Party Management | 5 | parties, addresses, agencies, agents |
| Product & Rating | 4 | products, coverage_types, rating_tables, rating_factors |
| Policy Management | 4 | policies, policy_coverages, endorsements, policy_interests |
| Insured Assets | 4 | insured_items, vehicles, properties, drivers |
| Underwriting | 3 | submissions, questions, answers |
| Claims | 4 | claims, claim_coverages, claim_activities, claim_payments |
| Billing & Payments | 4 | billing_accounts, invoices, payment_methods, payment_transactions |
| Reinsurance | 2 | treaties, cessions |
| Documents & Communications | 2 | documents, communications |
| Audit & Regulatory | 2 | audit_log, regulatory_filings |
| Commissions | 3 | schedules, statements, entries |
| **Total** | **39** | |

---

## Key Design Decisions

1. **Unified party model** — individuals and organisations share a `parties` table with type discrimination rather than separate tables, because the same party can be a policyholder, claimant, agent, loss payee, or reinsurer. This mirrors ACORD's Party aggregate.

2. **Policy renewal as a linked list** — `prior_policy_id` creates a chain of policies through renewal cycles, preserving the full history while keeping each term as an independent record with its own coverages and premiums.

3. **Endorsement as a first-class entity** — endorsements are not "edits to the policy" but separate records with their own approval workflow, effective date, and premium change. This preserves the audit trail of every mid-term change.

4. **Insured item subtyping via separate detail tables** — a base `insured_items` table with `insured_vehicles` and `insured_properties` as one-to-one extensions avoids JSONB for structured, well-known fields while remaining extensible to new item types.

5. **PCI DSS compliance by design** — the `payment_methods` table stores only tokens and last-four digits, never raw card numbers. All payment processing is delegated to an external processor.

6. **ISO 20022 alignment in payments** — `payment_transactions` includes an `iso20022_msg_id` field for cross-referencing with ISO 20022 payment messages, anticipating the industry's migration to that standard.

7. **Claim coverage breakdown** — claims link to specific `policy_coverages` through `claim_coverages`, enabling per-coverage reserve and payment tracking as required by NAIC reporting.

8. **Audit log with field-level change tracking** — the `audit_log` table uses JSONB for `field_changes` to capture old/new values per field, providing the granular audit trail regulators expect without adding dozens of history tables.

9. **Multi-tenant with row-level isolation** — `tenant_id` on all major tables enables shared-database multi-tenancy with row-level security policies, suitable for a SaaS deployment serving multiple carriers.

10. **ACORD-aligned entity hierarchy** — the table structure (Policy → Coverage → Limit/Deductible; Claim → ClaimCoverage → Payment) mirrors the ACORD XML schema hierarchy, making import/export mapping straightforward.
