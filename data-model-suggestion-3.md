# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Insurance Policy Management · Created: 2026-05-20

## Philosophy

This model takes a pragmatic middle path: core structural relationships (policies, claims, parties, billing) are modelled as traditional relational tables with foreign keys and constraints, while domain-specific variation is handled by JSONB columns on those same tables. The insight is that insurance policies share a common skeletal structure (every policy has a policyholder, coverages, premiums, and dates) but the details vary enormously by line of business, jurisdiction, and carrier.

A homeowners policy has construction type, roof material, and fire protection class. An auto policy has VIN, driver records, and garaging ZIP. A commercial general liability policy has class codes, employee counts, and revenue bands. A cyber policy has network size, security posture, and breach history. In a normalized model, you either create separate tables for every line (explosion of tables) or add nullable columns for every possible field (wide, sparse tables). With JSONB, the core relational skeleton stays clean and constrained, while the `details` JSONB column on `policies`, `insured_items`, and `claims` holds line-specific fields that vary by product.

This approach is proven in production at scale — BriteCore's cloud-native platform uses a similar pattern, and Salesforce Financial Services Cloud stores insurance-specific fields as custom objects with flexible schemas. PostgreSQL's JSONB support (GIN indexes, containment queries, JSON path expressions) makes this performant for read-heavy insurance workloads.

**Best for:** Carriers needing rapid multi-line deployment, multi-jurisdiction flexibility, and fast MVP development without sacrificing relational integrity on core entities.

**Trade-offs:**
- Pro: Clean relational core for queries, reporting, and ACORD mapping
- Pro: JSONB fields absorb line-of-business and jurisdiction variation without schema changes
- Pro: Fewer tables than fully normalized (~25 vs ~40+)
- Pro: Adding a new line of business requires no schema migration — just new JSONB field conventions
- Pro: PostgreSQL JSONB indexing supports efficient queries into variable fields
- Con: JSONB fields lack database-level type constraints (validated at application layer)
- Con: JSONB field documentation must be maintained outside the schema
- Con: Complex reporting on JSONB fields requires JSON path queries (less familiar to SQL analysts)
- Con: Risk of JSONB becoming a dumping ground without governance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ACORD P&C Data Standards | Core relational tables map to ACORD aggregate types; JSONB `details` fields carry line-specific ACORD elements |
| ACORD Next-Gen Digital Standards | JSON API responses blend relational fields with JSONB details seamlessly |
| ISO 3166 | `jurisdiction` columns use ISO 3166-1/2 codes |
| ISO 4217 | `currency` columns use ISO 4217 codes |
| ISO 20022 | Payment transaction fields align with ISO 20022 message elements |
| NAIC Model Laws | `regulatory_config` JSONB on tenants captures state-specific filing requirements |
| PCI DSS v4.0.1 | Payment tokens only — no raw card data stored |
| HL7 FHIR R5 | Health line policy `details` JSONB can include FHIR-aligned fields (member_id, group_number, plan_type) |
| JSON Schema | JSONB `details` fields validated against JSON Schema definitions stored in `product_schemas` |

---

## Core Identity & Configuration

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tenant_type VARCHAR(50) NOT NULL CHECK (tenant_type IN ('carrier', 'mga', 'broker', 'reinsurer')),
    naic_code VARCHAR(20),
    lei VARCHAR(20),
    jurisdiction_country CHAR(2) NOT NULL,
    -- JSONB for carrier-specific configuration that varies widely
    regulatory_config JSONB DEFAULT '{}',
    -- Example regulatory_config:
    -- {
    --   "filing_states": ["CA", "TX", "NY"],
    --   "naic_reporting": true,
    --   "solvency_ii": false,
    --   "surplus_lines_states": ["CA"],
    --   "rate_approval_required": ["NY"]
    -- }
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
    role VARCHAR(50) NOT NULL CHECK (role IN (
        'admin', 'underwriter', 'claims_adjuster', 'agent', 'customer_service', 'billing', 'auditor'
    )),
    permissions JSONB DEFAULT '[]',
    -- Example permissions: ["policy.write", "claim.approve", "billing.refund"]
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant_role ON users(tenant_id, role);
```

## Party Management

```sql
CREATE TABLE parties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    party_type VARCHAR(20) NOT NULL CHECK (party_type IN ('individual', 'organisation')),
    display_name VARCHAR(255) NOT NULL,       -- Computed: "first last" or org name
    email VARCHAR(255),
    phone VARCHAR(50),
    -- Structured fields for the most common attributes
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    organisation_name VARCHAR(255),
    date_of_birth DATE,
    tax_id_last4 VARCHAR(4),
    lei VARCHAR(20),
    -- JSONB for variable party data
    details JSONB DEFAULT '{}',
    -- Individual example:
    -- {
    --   "gender": "F",
    --   "marital_status": "married",
    --   "occupation": "engineer",
    --   "credit_score_tier": "excellent",
    --   "npn": "12345678"
    -- }
    -- Organisation example:
    -- {
    --   "industry_code": "524210",
    --   "employee_count": 150,
    --   "annual_revenue": 25000000,
    --   "years_in_business": 12,
    --   "sic_code": "6311"
    -- }
    addresses JSONB DEFAULT '[]',
    -- Example addresses:
    -- [
    --   {"type": "mailing", "line1": "123 Main St", "city": "Austin", "state": "TX", "zip": "78701", "country": "US", "is_primary": true},
    --   {"type": "billing", "line1": "PO Box 456", "city": "Austin", "state": "TX", "zip": "78702", "country": "US"}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parties_tenant ON parties(tenant_id);
CREATE INDEX idx_parties_email ON parties(tenant_id, email);
CREATE INDEX idx_parties_name ON parties(tenant_id, display_name);
CREATE INDEX idx_parties_details ON parties USING gin(details jsonb_path_ops);
```

## Product Configuration

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    product_code VARCHAR(50) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    line_of_business VARCHAR(50) NOT NULL,
    iso_class_code VARCHAR(20),
    -- JSON Schema that validates policy `details` for this product
    policy_schema JSONB NOT NULL DEFAULT '{}',
    -- JSON Schema that validates insured_item `details` for this product
    item_schema JSONB NOT NULL DEFAULT '{}',
    -- Rating configuration
    rating_config JSONB DEFAULT '{}',
    -- Example rating_config:
    -- {
    --   "base_rate_table": "ho3_base_2026",
    --   "factor_sequence": ["territory", "protection_class", "construction", "age_of_home", "credit"],
    --   "minimum_premium": 350.00,
    --   "rounding": "nearest_dollar"
    -- }
    -- Coverage definitions for this product
    coverage_definitions JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"code": "COV_A", "name": "Dwelling", "mandatory": true, "default_limit": 300000, "min_limit": 100000, "max_limit": 5000000},
    --   {"code": "COV_B", "name": "Other Structures", "mandatory": true, "default_pct_of_a": 0.10},
    --   {"code": "COV_C", "name": "Personal Property", "mandatory": true, "default_pct_of_a": 0.50},
    --   {"code": "COV_E", "name": "Personal Liability", "mandatory": true, "default_limit": 100000}
    -- ]
    underwriting_questions JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   {"code": "Q1", "text": "Any losses in the past 5 years?", "type": "yes_no", "required": true},
    --   {"code": "Q2", "text": "Is the property owner-occupied?", "type": "yes_no", "required": true},
    --   {"code": "Q3", "text": "Distance to fire hydrant (feet)?", "type": "number", "required": false}
    -- ]
    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_code)
);

-- Rating tables stored as JSONB for flexibility across lines
CREATE TABLE rating_tables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    product_id UUID NOT NULL REFERENCES products(id),
    table_name VARCHAR(255) NOT NULL,
    jurisdiction CHAR(2),
    effective_date DATE NOT NULL,
    expiration_date DATE,
    version INT NOT NULL DEFAULT 1,
    -- The rating data itself — structure varies by table type
    rate_data JSONB NOT NULL,
    -- Example (territory factors):
    -- {
    --   "factor_type": "territory",
    --   "rates": {
    --     "001": 0.85, "002": 0.90, "003": 1.00, "004": 1.15,
    --     "005": 1.30, "006": 1.50, "007": 1.75
    --   }
    -- }
    -- Example (age-of-home factors):
    -- {
    --   "factor_type": "age_of_home",
    --   "rates": [
    --     {"min_age": 0, "max_age": 5, "factor": 0.90},
    --     {"min_age": 6, "max_age": 15, "factor": 1.00},
    --     {"min_age": 16, "max_age": 30, "factor": 1.10},
    --     {"min_age": 31, "max_age": null, "factor": 1.25}
    --   ]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rating_tables_product ON rating_tables(product_id, effective_date);
```

## Policy Management

```sql
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    policy_number VARCHAR(50) NOT NULL,
    product_id UUID NOT NULL REFERENCES products(id),
    policyholder_id UUID NOT NULL REFERENCES parties(id),
    agent_id UUID REFERENCES parties(id),     -- Agent is a party
    status VARCHAR(30) NOT NULL CHECK (status IN (
        'quote', 'application', 'bound', 'issued', 'active',
        'pending_renewal', 'renewed', 'cancelled', 'expired', 'non_renewed'
    )),
    effective_date DATE NOT NULL,
    expiration_date DATE NOT NULL,
    original_inception_date DATE NOT NULL,
    term_months INT NOT NULL DEFAULT 12,
    jurisdiction CHAR(2) NOT NULL,
    total_premium NUMERIC(15,2),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    prior_policy_id UUID REFERENCES policies(id),
    underwriter_id UUID REFERENCES users(id),
    -- Coverages as structured JSONB array
    coverages JSONB NOT NULL DEFAULT '[]',
    -- Example coverages:
    -- [
    --   {"code": "COV_A", "name": "Dwelling", "limit": 500000, "deductible": 2500, "premium": 1800.00, "status": "active"},
    --   {"code": "COV_B", "name": "Other Structures", "limit": 50000, "deductible": 0, "premium": 180.00, "status": "active"},
    --   {"code": "COV_C", "name": "Personal Property", "limit": 250000, "deductible": 500, "premium": 320.00, "status": "active"},
    --   {"code": "COV_E", "name": "Personal Liability", "limit": 300000, "deductible": 0, "premium": 150.00, "status": "active"}
    -- ]
    -- Line-of-business-specific policy details
    details JSONB DEFAULT '{}',
    -- Homeowners example:
    -- {
    --   "form_type": "HO-3",
    --   "territory_code": "005",
    --   "protection_class": "3",
    --   "prior_carrier": "State Farm",
    --   "prior_policy_expiry": "2026-06-30",
    --   "underwriting_answers": {"Q1": "no", "Q2": "yes", "Q3": 200}
    -- }
    -- Commercial auto example:
    -- {
    --   "fleet_size": 25,
    --   "radius_of_operation": 200,
    --   "cargo_types": ["general_freight", "electronics"],
    --   "dot_number": "1234567",
    --   "safety_rating": "satisfactory"
    -- }
    -- Additional interests (named insureds, mortgagees, etc.)
    interests JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   {"party_id": "uuid-...", "type": "named_insured", "name": "Jane Smith"},
    --   {"party_id": "uuid-...", "type": "mortgagee", "name": "First National Bank", "loan_number": "MTG-12345"}
    -- ]
    bound_at TIMESTAMPTZ,
    issued_at TIMESTAMPTZ,
    cancellation_date DATE,
    cancellation_reason VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, policy_number)
);

CREATE INDEX idx_policies_tenant_status ON policies(tenant_id, status);
CREATE INDEX idx_policies_policyholder ON policies(policyholder_id);
CREATE INDEX idx_policies_agent ON policies(agent_id);
CREATE INDEX idx_policies_dates ON policies(effective_date, expiration_date);
CREATE INDEX idx_policies_renewal ON policies(prior_policy_id);
CREATE INDEX idx_policies_coverages ON policies USING gin(coverages jsonb_path_ops);
CREATE INDEX idx_policies_details ON policies USING gin(details jsonb_path_ops);

-- Endorsements
CREATE TABLE endorsements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    endorsement_number INT NOT NULL,
    endorsement_type VARCHAR(50) NOT NULL,
    description TEXT NOT NULL,
    effective_date DATE NOT NULL,
    premium_change NUMERIC(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'pending_approval', 'approved', 'issued', 'rejected')),
    -- What changed — stored as a diff
    changes JSONB NOT NULL,
    -- Example:
    -- {
    --   "coverages": {
    --     "COV_A": {"limit": {"old": 500000, "new": 600000}, "premium": {"old": 1800, "new": 2100}}
    --   },
    --   "details": {
    --     "territory_code": {"old": "005", "new": "003"}
    --   }
    -- }
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, endorsement_number)
);

CREATE INDEX idx_endorsements_policy ON endorsements(policy_id);
```

## Insured Items

```sql
-- Insured items with type-specific details in JSONB
CREATE TABLE insured_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    item_type VARCHAR(30) NOT NULL CHECK (item_type IN (
        'vehicle', 'property', 'equipment', 'watercraft', 'aircraft', 'scheduled_item', 'location'
    )),
    item_number INT NOT NULL,
    description TEXT NOT NULL,
    -- Item-specific coverages (may differ from policy-level)
    coverages JSONB DEFAULT '[]',
    -- Type-specific details
    details JSONB NOT NULL DEFAULT '{}',
    -- Vehicle example:
    -- {
    --   "vin": "1HGBH41JXMN109186",
    --   "year": 2025,
    --   "make": "Honda",
    --   "model": "Civic",
    --   "body_type": "sedan",
    --   "usage": "commute",
    --   "annual_mileage": 12000,
    --   "garaging_zip": "78701",
    --   "anti_theft": true,
    --   "telematics_enrolled": true
    -- }
    -- Property example:
    -- {
    --   "property_type": "single_family",
    --   "construction": "frame",
    --   "year_built": 1985,
    --   "square_footage": 2200,
    --   "stories": 2,
    --   "roof_type": "composition_shingle",
    --   "roof_year": 2020,
    --   "replacement_cost": 450000,
    --   "protection_class": "3",
    --   "territory": "005",
    --   "heating_type": "central_gas",
    --   "electrical_updated": true,
    --   "pool": false,
    --   "trampoline": false
    -- }
    -- Equipment/scheduled item example:
    -- {
    --   "serial_number": "SN-ABC-12345",
    --   "manufacturer": "Caterpillar",
    --   "model": "320F",
    --   "year": 2022,
    --   "appraised_value": 185000,
    --   "location": "Job Site A, Houston TX"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insured_items_policy ON insured_items(policy_id);
CREATE INDEX idx_insured_items_details ON insured_items USING gin(details jsonb_path_ops);

-- Drivers (relational — commonly queried across policies)
CREATE TABLE drivers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    party_id UUID NOT NULL REFERENCES parties(id),
    driver_number INT NOT NULL,
    license_number VARCHAR(50),
    license_state CHAR(2),
    driver_status VARCHAR(30) NOT NULL DEFAULT 'rated',
    details JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "date_licensed": "2010-03-15",
    --   "good_driver_discount": true,
    --   "defensive_driving_course": true,
    --   "violations": [
    --     {"date": "2024-06-01", "type": "speeding", "points": 2}
    --   ],
    --   "accidents": []
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drivers_policy ON drivers(policy_id);
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
    loss_type VARCHAR(50) NOT NULL,
    loss_description TEXT,
    assigned_adjuster_id UUID REFERENCES users(id),
    total_incurred NUMERIC(15,2) DEFAULT 0,
    total_paid NUMERIC(15,2) DEFAULT 0,
    total_reserved NUMERIC(15,2) DEFAULT 0,
    fraud_score NUMERIC(5,2),
    -- Coverage breakdown and loss-type-specific details
    coverage_breakdown JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   {"coverage_code": "COV_A", "reserve": 50000, "paid": 15000, "status": "open"},
    --   {"coverage_code": "COV_C", "reserve": 25000, "paid": 8000, "status": "open"}
    -- ]
    details JSONB DEFAULT '{}',
    -- Fire claim example:
    -- {
    --   "fire_origin": "kitchen",
    --   "fire_cause": "electrical",
    --   "fire_department_responded": true,
    --   "police_report_number": "PD-2026-1234",
    --   "temporary_housing_needed": true,
    --   "estimated_repair_months": 4
    -- }
    -- Auto claim example:
    -- {
    --   "vehicle_id": "uuid-...",
    --   "driver_id": "uuid-...",
    --   "accident_type": "rear_end",
    --   "at_fault": false,
    --   "other_party": {"name": "John Doe", "carrier": "GEICO", "claim_number": "GK-12345"},
    --   "police_report": true,
    --   "tow_required": true,
    --   "airbag_deployed": false
    -- }
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, claim_number)
);

CREATE INDEX idx_claims_policy ON claims(policy_id);
CREATE INDEX idx_claims_tenant_status ON claims(tenant_id, status);
CREATE INDEX idx_claims_adjuster ON claims(assigned_adjuster_id);
CREATE INDEX idx_claims_loss_date ON claims(loss_date);
CREATE INDEX idx_claims_details ON claims USING gin(details jsonb_path_ops);

-- Claim activities (diary/notes/actions)
CREATE TABLE claim_activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID NOT NULL REFERENCES claims(id),
    activity_type VARCHAR(50) NOT NULL,
    description TEXT NOT NULL,
    details JSONB DEFAULT '{}',
    -- Payment activity example:
    -- {"payee": "Acme Restoration", "amount": 15000, "check_number": "CHK-8901", "payment_type": "indemnity"}
    performed_by UUID REFERENCES users(id),
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_activities_claim ON claim_activities(claim_id);
```

## Billing & Payments

```sql
CREATE TABLE billing_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    account_number VARCHAR(50) NOT NULL,
    policyholder_id UUID NOT NULL REFERENCES parties(id),
    billing_plan VARCHAR(30) NOT NULL,
    balance_due NUMERIC(15,2) NOT NULL DEFAULT 0,
    next_due_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    payment_methods JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   {"type": "card", "token": "tok_abc123", "last_four": "4242", "expiry": "12/2028", "is_default": true},
    --   {"type": "ach", "token": "ba_xyz789", "bank_name": "Chase", "last_four": "6789"}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, account_number)
);

CREATE INDEX idx_billing_accounts_policyholder ON billing_accounts(policyholder_id);

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
    line_items JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   {"description": "Dwelling coverage premium", "coverage_code": "COV_A", "amount": 450.00},
    --   {"description": "Personal property premium", "coverage_code": "COV_C", "amount": 80.00},
    --   {"description": "Policy fee", "amount": 25.00}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_billing_account ON invoices(billing_account_id);
CREATE INDEX idx_invoices_policy ON invoices(policy_id);
CREATE INDEX idx_invoices_due_date ON invoices(due_date, status);

CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id UUID REFERENCES invoices(id),
    transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('payment', 'refund', 'chargeback', 'adjustment')),
    amount NUMERIC(15,2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method VARCHAR(20),
    processor_reference VARCHAR(255),
    iso20022_msg_id VARCHAR(35),
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
    treaty_type VARCHAR(30) NOT NULL,
    reinsurer_id UUID NOT NULL REFERENCES parties(id),
    effective_date DATE NOT NULL,
    expiration_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Treaty terms vary significantly by type
    terms JSONB NOT NULL,
    -- Quota share example:
    -- {
    --   "cession_percentage": 30.0,
    --   "commission_rate": 25.0,
    --   "lines_covered": ["homeowners", "commercial_property"],
    --   "excluded_perils": ["flood", "earthquake"],
    --   "maximum_cession": 5000000
    -- }
    -- Excess of loss example:
    -- {
    --   "retention": 500000,
    --   "limit": 10000000,
    --   "rate_on_line": 8.5,
    --   "reinstatements": 2,
    --   "reinstatement_premium_pct": 100,
    --   "annual_aggregate_limit": 30000000
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, treaty_number)
);

CREATE TABLE reinsurance_cessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    treaty_id UUID NOT NULL REFERENCES reinsurance_treaties(id),
    policy_id UUID NOT NULL REFERENCES policies(id),
    ceded_premium NUMERIC(15,2) NOT NULL,
    ceding_commission NUMERIC(15,2) NOT NULL DEFAULT 0,
    cession_date DATE NOT NULL,
    details JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cessions_treaty ON reinsurance_cessions(treaty_id);
CREATE INDEX idx_cessions_policy ON reinsurance_cessions(policy_id);
```

## Documents, Audit & Commissions

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    entity_type VARCHAR(30) NOT NULL,
    entity_id UUID NOT NULL,
    document_type VARCHAR(50) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path VARCHAR(500) NOT NULL,
    metadata JSONB DEFAULT '{}',
    -- Example: {"generated_by": "template_engine", "template_id": "dec_page_ho3_v2", "signed": false}
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_entity ON documents(entity_type, entity_id);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID REFERENCES users(id),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(30) NOT NULL,
    changes JSONB,
    -- Example: {"status": {"old": "quote", "new": "bound"}, "total_premium": {"old": null, "new": 2450.00}}
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant_time ON audit_log(tenant_id, created_at);

-- Agent commissions
CREATE TABLE commission_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    agent_id UUID NOT NULL REFERENCES parties(id),
    policy_id UUID NOT NULL REFERENCES policies(id),
    commission_type VARCHAR(20) NOT NULL,
    premium_basis NUMERIC(15,2) NOT NULL,
    rate NUMERIC(5,4) NOT NULL,
    amount NUMERIC(15,2) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'earned',
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commissions_agent ON commission_entries(agent_id, period_start);
CREATE INDEX idx_commissions_policy ON commission_entries(policy_id);
```

## JSONB Query Examples

```sql
-- Find all homeowners policies with protection class 3 or better
SELECT id, policy_number, total_premium
FROM policies p
JOIN products pr ON p.product_id = pr.id
WHERE p.tenant_id = :tenant_id
  AND pr.line_of_business = 'homeowners'
  AND (p.details->>'protection_class')::int <= 3;

-- Find policies with dwelling coverage over $500k
SELECT id, policy_number
FROM policies
WHERE tenant_id = :tenant_id
  AND coverages @> '[{"code": "COV_A"}]'
  AND EXISTS (
      SELECT 1 FROM jsonb_array_elements(coverages) cov
      WHERE cov->>'code' = 'COV_A'
        AND (cov->>'limit')::numeric > 500000
  );

-- Find all auto claims where airbag deployed
SELECT c.claim_number, c.loss_date, c.total_incurred
FROM claims c
WHERE c.tenant_id = :tenant_id
  AND c.details @> '{"airbag_deployed": true}';

-- Get territory rating factor for a specific territory
SELECT rate_data->'rates'->>:territory_code AS factor
FROM rating_tables
WHERE product_id = :product_id
  AND table_name = 'territory_factors'
  AND effective_date <= :policy_effective_date
  AND (expiration_date IS NULL OR expiration_date > :policy_effective_date)
  AND is_active = true
ORDER BY effective_date DESC
LIMIT 1;

-- Aggregate claims by loss type from JSONB details
SELECT loss_type,
       COUNT(*) AS claim_count,
       SUM(total_incurred) AS total_incurred,
       AVG(fraud_score) AS avg_fraud_score
FROM claims
WHERE tenant_id = :tenant_id
  AND loss_date >= '2026-01-01'
GROUP BY loss_type
ORDER BY total_incurred DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Configuration | 2 | tenants, users |
| Party Management | 1 | parties (addresses in JSONB) |
| Product & Rating | 2 | products (with coverage defs in JSONB), rating_tables (rates in JSONB) |
| Policy Management | 2 | policies (coverages + details in JSONB), endorsements (changes in JSONB) |
| Insured Items | 2 | insured_items (details in JSONB), drivers |
| Claims | 2 | claims (breakdown + details in JSONB), claim_activities |
| Billing & Payments | 3 | billing_accounts, invoices, payment_transactions |
| Reinsurance | 2 | treaties (terms in JSONB), cessions |
| Documents & Audit | 2 | documents, audit_log |
| Commissions | 1 | commission_entries |
| **Total** | **19** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **Coverages as JSONB array on policies, not a separate table** — coverages are always read and written together with the policy. Embedding them as a JSONB array eliminates joins for the most common query (show me this policy). The coverage definitions in the `products` table provide the schema, and application-layer validation ensures consistency.

2. **Addresses as JSONB array on parties** — a party rarely has more than 2-3 addresses, and they are always read together. Embedding them saves a table and a join. If address querying becomes important, a partial GIN index can handle it.

3. **Line-specific details in JSONB `details` column** — rather than creating `insured_vehicles`, `insured_properties`, `insured_watercraft` tables, a single `insured_items` table with a `details` JSONB column absorbs all variation. The `products.item_schema` provides JSON Schema validation.

4. **Rating tables as JSONB** — rating data structures vary dramatically between factor types (lookup table, banded range, matrix). JSONB accommodates all structures in a single table, and the application rating engine knows how to interpret each structure based on `factor_type`.

5. **Endorsement changes as JSONB diff** — storing the old/new values of each changed field in JSONB provides a natural audit trail without a separate history mechanism. To reconstruct the policy at any endorsement, apply changes in sequence.

6. **Reinsurance treaty terms as JSONB** — quota share, surplus, excess-of-loss, and catastrophe treaties have fundamentally different term structures. JSONB avoids a table-per-treaty-type pattern while keeping the relational structure for common fields.

7. **Payment methods as JSONB on billing accounts** — a billing account rarely has more than 2-3 payment methods. Embedding them avoids a separate table. PCI DSS compliance is maintained because only tokens and last-four digits are stored.

8. **JSON Schema for validation governance** — the `products.policy_schema` and `products.item_schema` fields store JSON Schema definitions that the application uses to validate JSONB `details` fields. This provides schema governance without database-level constraints.

9. **19 tables vs 39** — roughly half the table count of the normalized model, achieved by consolidating infrequently-queried or type-variable data into JSONB. Core relationships (policy-to-policyholder, claim-to-policy, invoice-to-policy) remain relational with foreign keys.

10. **GIN indexes on JSONB columns** — `jsonb_path_ops` GIN indexes on `details`, `coverages`, and other JSONB columns enable efficient containment queries (`@>` operator) without sacrificing query performance for the flexible fields.
