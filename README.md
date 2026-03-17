# Sentinel_Framework
Sentinel Framework Design to solve Data Quality issue using n8n AI.
# 🛡️ Sentinel Framework
## AI-Powered Data Quality & Observability for Healthcare Data Pipelines

### Overview
The Sentinel Framework is a comprehensive data quality architecture designed for modern healthcare data platforms. It combines circuit breaker patterns, clinical validation rules, and agentic AI observability to ensure data reliability while accelerating pipeline velocity.

**Built for:** Azure Data Factory → Snowflake → Collibra ecosystems  
**Industry Focus:** Healthcare (HIPAA-compliant, PHI/PII aware)

---

## 🏗️ Architecture Principles

### Core Philosophy
> "Quality isn't a bottleneck—it's an accelerator that allows engineering teams to ship code faster because they trust the safety nets we've built."

The framework operates on three layers:

1. **Ingestion Guard** - Schema drift detection and prevention
2. **Clinical Validator** - Healthcare-specific business rule enforcement
3. **Agentic Observer** - AI-driven anomaly detection with smart alerting

---

## 🔧 Component 1: The Ingestion Guard

### Purpose
Prevent schema drift and data type mismatches before they corrupt downstream analytics.

### Implementation Stack
- **Azure Data Factory**: Metadata-driven pipelines with pre-load validation
- **Snowflake Dynamic Tables**: Automated freshness with quality constraints
- **Circuit Breaker Pattern**: Automatic pipeline suspension on quality failures

### Key Features

#### Schema Drift Detection
```sql
-- Automated column tracking in Snowflake
CREATE OR REPLACE TABLE metadata.schema_registry (
    table_name VARCHAR,
    column_name VARCHAR,
    data_type VARCHAR,
    is_nullable BOOLEAN,
    first_seen TIMESTAMP,
    last_validated TIMESTAMP,
    validation_status VARCHAR
);

-- Detect new columns in raw data
CREATE OR REPLACE PROCEDURE detect_schema_drift(source_table VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    LET drift_count NUMBER;
    
    SELECT COUNT(*) INTO :drift_count
    FROM information_schema.columns c
    WHERE c.table_name = :source_table
    AND NOT EXISTS (
        SELECT 1 FROM metadata.schema_registry sr
        WHERE sr.table_name = c.table_name
        AND sr.column_name = c.column_name
    );
    
    IF (drift_count > 0) THEN
        RETURN 'SCHEMA_DRIFT_DETECTED: ' || drift_count || ' new columns found';
    ELSE
        RETURN 'SCHEMA_VALID';
    END IF;
END;
$$;
```

#### Circuit Breaker Logic
- **Threshold**: >5% row-level failures OR any schema drift
- **Action**: Stop pipeline, alert data engineering team, log incident
- **Recovery**: Manual approval or auto-retry after validation

### Azure Data Factory Integration
```json
{
  "name": "CircuitBreakerValidation",
  "type": "IfCondition",
  "dependsOn": [
    {
      "activity": "SchemaValidation",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "expression": {
      "@equals(activity('SchemaValidation').output.status, 'SCHEMA_VALID')"
    },
    "ifTrueActivities": [
      {
        "name": "LoadToSnowflake",
        "type": "Copy"
      }
    ],
    "ifFalseActivities": [
      {
        "name": "CircuitBreakerTriggered",
        "type": "Fail",
        "typeProperties": {
          "message": "Schema drift detected - pipeline halted",
          "errorCode": "DQ_001"
        }
      }
    ]
  }
}
```

---

## 🏥 Component 2: The Clinical Validator

### Purpose
Enforce healthcare-specific business rules for patient safety and billing accuracy.

### Validation Categories

#### 1. Patient Matching & Identity
```sql
-- Validate MRN (Medical Record Number) uniqueness
CREATE OR REPLACE TABLE dq_rules.patient_identity_checks AS
SELECT 
    mrn,
    COUNT(*) as record_count,
    COUNT(DISTINCT patient_name) as name_variations,
    COUNT(DISTINCT date_of_birth) as dob_variations,
    CASE 
        WHEN COUNT(*) > 1 THEN 'DUPLICATE_MRN'
        WHEN COUNT(DISTINCT date_of_birth) > 1 THEN 'DOB_MISMATCH'
        ELSE 'VALID'
    END as validation_status
FROM raw.patient_demographics
GROUP BY mrn
HAVING validation_status != 'VALID';
```

#### 2. ICD-10 Code Validation
```sql
-- Verify diagnosis codes against master reference
CREATE OR REPLACE TABLE dq_rules.diagnosis_validation AS
SELECT 
    encounter_id,
    diagnosis_code,
    CASE 
        WHEN diagnosis_code REGEXP '^[A-Z][0-9]{2}(\\.?[0-9A-Z]{0,4})?$' 
            AND EXISTS (
                SELECT 1 FROM reference.icd10_codes 
                WHERE code = diagnosis_code
            ) THEN 'VALID'
        WHEN diagnosis_code REGEXP '^[A-Z][0-9]{2}' THEN 'INVALID_FORMAT'
        ELSE 'CODE_NOT_FOUND'
    END as validation_status
FROM raw.clinical_encounters
WHERE validation_status != 'VALID';
```

#### 3. HL7 Message Integrity
```sql
-- Validate HL7 ADT (Admit/Discharge/Transfer) message structure
CREATE OR REPLACE FUNCTION validate_hl7_adt(message VARCHAR)
RETURNS OBJECT
LANGUAGE JAVASCRIPT
AS
$$
    const segments = MESSAGE.split('\r');
    const msh = segments.find(s => s.startsWith('MSH'));
    const pid = segments.find(s => s.startsWith('PID'));
    const pv1 = segments.find(s => s.startsWith('PV1'));
    
    return {
        has_msh: msh !== undefined,
        has_pid: pid !== undefined,
        has_pv1: pv1 !== undefined,
        is_valid: msh && pid && pv1,
        message_type: msh ? msh.split('|')[8] : 'UNKNOWN'
    };
$$;
```

#### 4. Claim Denial Prevention Rules
```sql
-- Pre-submission validation for claims
CREATE OR REPLACE TABLE dq_rules.claims_preflight_check AS
SELECT 
    claim_id,
    service_date,
    procedure_code,
    diagnosis_code,
    provider_npi,
    CASE 
        WHEN service_date > CURRENT_DATE THEN 'FUTURE_DATE_ERROR'
        WHEN DATEDIFF(day, service_date, CURRENT_DATE) > 365 THEN 'TIMELY_FILING_RISK'
        WHEN provider_npi NOT IN (SELECT npi FROM reference.active_providers) THEN 'INVALID_PROVIDER'
        WHEN NOT EXISTS (
            SELECT 1 FROM reference.valid_dx_proc_combos 
            WHERE dx = diagnosis_code AND proc = procedure_code
        ) THEN 'MEDICAL_NECESSITY_FLAG'
        ELSE 'READY_FOR_SUBMISSION'
    END as preflight_status
FROM raw.claims_staging
WHERE preflight_status != 'READY_FOR_SUBMISSION';
```

### PHI/PII Masking Strategy
```sql
-- Dynamic data masking for non-production environments
CREATE OR REPLACE MASKING POLICY phi_mask AS (val STRING) 
RETURNS STRING ->
    CASE 
        WHEN CURRENT_ROLE() IN ('PRODUCTION_ANALYST', 'COMPLIANCE_TEAM') THEN val
        ELSE '***MASKED***'
    END;

-- Apply to sensitive columns
ALTER TABLE patient_demographics MODIFY COLUMN ssn 
    SET MASKING POLICY phi_mask;
ALTER TABLE patient_demographics MODIFY COLUMN patient_name 
    SET MASKING POLICY phi_mask;
```

---

## 🤖 Component 3: The Agentic Observer

### Purpose
Reduce alert fatigue by using AI agents to distinguish real issues from noise.

### Agentic AI Capabilities

#### 1. Autonomous Anomaly Detection
```python
# Conceptual framework - integrates with observability tools
class DataQualityAgent:
    """
    AI Agent that monitors data streams and autonomously 
    creates validation rules when new patterns emerge
    """
    
    def __init__(self, snowflake_conn, llm_client):
        self.conn = snowflake_conn
        self.llm = llm_client
        self.learned_patterns = []
    
    def detect_new_provider_format(self, table_name):
        """
        Example: Detects when a new provider code format appears
        and auto-generates a dbt test
        """
        query = f"""
        SELECT DISTINCT provider_code, COUNT(*) as frequency
        FROM {table_name}
        WHERE load_date >= CURRENT_DATE - 7
        GROUP BY provider_code
        HAVING frequency > 100
        """
        
        new_codes = self.conn.execute(query).fetchall()
        
        # Use LLM to analyze if this is a valid new format
        prompt = f"""
        Analyze these new provider codes: {new_codes}
        Compare to historical patterns: {self.learned_patterns}
        
        Is this a legitimate new format or data corruption?
        If legitimate, generate a dbt test YAML.
        """
        
        analysis = self.llm.generate(prompt)
        
        if analysis['is_legitimate']:
            self.create_dbt_test(analysis['test_yaml'])
            self.learned_patterns.append(analysis['pattern'])
    
    def create_dbt_test(self, test_yaml):
        """Auto-generate and commit dbt tests"""
        # Write to version control
        pass
```

#### 2. Smart Alert Prioritization
```sql
-- AI-scored alert criticality
CREATE OR REPLACE TABLE monitoring.smart_alerts AS
SELECT 
    alert_id,
    table_name,
    rule_violated,
    row_count_affected,
    business_impact_score,
    historical_false_positive_rate,
    -- AI agent adds context
    ai_severity_score,
    ai_recommended_action,
    ai_root_cause_hypothesis
FROM monitoring.raw_alerts
WHERE ai_severity_score > 0.7  -- Only surface high-confidence issues
ORDER BY ai_severity_score DESC;
```

#### 3. Self-Healing Data Pipelines
- **Minor issues**: Agent auto-corrects (e.g., trim whitespace, standardize date formats)
- **Medium issues**: Agent proposes fix, requires human approval
- **Critical issues**: Immediate circuit breaker, escalate to on-call engineer

### Integration with Observability Tools
- **Monte Carlo**: Anomaly detection with ML-powered incident prediction
- **Soda**: Contract testing with AI-enhanced threshold tuning
- **Collibra DQ**: Business glossary enrichment via NLP agents

---

## 📊 Metrics & SLAs

### Data Quality Score
```sql
CREATE OR REPLACE VIEW reporting.dq_scorecard AS
SELECT 
    table_name,
    -- Completeness
    AVG(CASE WHEN column_value IS NOT NULL THEN 1 ELSE 0 END) as completeness_score,
    -- Validity
    AVG(CASE WHEN validation_status = 'VALID' THEN 1 ELSE 0 END) as validity_score,
    -- Timeliness
    AVG(CASE WHEN DATEDIFF(hour, source_timestamp, load_timestamp) <= 4 THEN 1 ELSE 0 END) as timeliness_score,
    -- Overall DQ Score
    (completeness_score + validity_score + timeliness_score) / 3 as overall_dq_score
FROM monitoring.quality_metrics
GROUP BY table_name;
```

### Key Performance Indicators
| Metric | Target | Current Baseline |
|--------|--------|------------------|
| Pipeline Uptime | 99.5% | Measure in first 30 days |
| False Positive Alert Rate | <10% | Reduce via Agentic Observer |
| Mean Time to Detection (MTTD) | <15 min | Real-time monitoring |
| Mean Time to Resolution (MTTR) | <2 hours | Circuit breaker automation |
| Claim Denial Rate | <3% | Track post-implementation |

---

## 🚀 Implementation Roadmap

### Phase 1: Foundation (Months 1-2)
- Deploy Ingestion Guard to top 10 critical tables
- Establish baseline DQ metrics
- Configure circuit breakers in Azure Data Factory

### Phase 2: Clinical Rules (Months 2-3)
- Implement patient matching validation
- Deploy ICD-10 code verification
- Launch claim denial prevention checks

### Phase 3: Agentic AI Pilot (Months 3-4)
- Integrate AI agent for anomaly detection
- Train model on 90 days of historical incidents
- Deploy smart alert prioritization

### Phase 4: Scale & Optimize (Months 5-6)
- Expand to all production pipelines
- Automate self-healing for common issues
- Integrate with Collibra for governance workflow

---

## 🔐 HIPAA Compliance

### PHI Protection Mechanisms
1. **Static Data Masking**: Irreversible for non-prod environments
2. **Dynamic Data Masking**: Role-based access in production
3. **Audit Logging**: All PHI access tracked in Snowflake ACCOUNT_USAGE
4. **Encryption**: TLS 1.2+ in transit, AES-256 at rest

### Compliance Checkpoints
```sql
-- Audit trail for PHI access
CREATE OR REPLACE VIEW compliance.phi_access_log AS
SELECT 
    query_id,
    user_name,
    role_name,
    query_text,
    execution_time,
    rows_produced
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%patient_demographics%'
   OR query_text ILIKE '%ssn%'
ORDER BY execution_time DESC;
```

---

## 📈 Business Impact

### Expected Outcomes
- **Claim Denial Reduction**: 15-20% decrease in first 6 months
- **Data Engineering Velocity**: 30% faster feature releases due to quality confidence
- **Operational Efficiency**: 50% reduction in manual data investigation time
- **Compliance Posture**: Zero HIPAA violations related to data quality

### ROI Calculation
```
Cost Avoidance (Annual):
- Prevented claim denials: $2.5M
- Reduced engineering rework: $800K
- Avoided compliance penalties: $1M+

Investment:
- Framework development: $150K
- Ongoing maintenance: $75K/year

Net Benefit: $4M+ annually
```

---

## 🛠️ Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Ingestion | Azure Data Factory | Orchestration & circuit breakers |
| Storage | Snowflake | Data warehouse with native DQ features |
| Governance | Collibra | Business glossary & data stewardship |
| Observability | Monte Carlo / Soda | Anomaly detection & contract testing |
| AI/ML | OpenAI API / Claude | Agentic observer & root cause analysis |
| Version Control | Git + dbt | Test automation & lineage |

---

## 📚 References

### Industry Standards
- **HIPAA Security Rule**: 45 CFR §164.312
- **HL7 FHIR**: Healthcare data interoperability standards
- **ICD-10-CM**: Clinical diagnosis coding system

### Data Quality Frameworks
- **DAMA-DMBOK**: Data Management Body of Knowledge
- **ISO 8000**: Data quality standards
- **Google SRE Book**: Circuit breaker patterns

---

## 👤 Author

**Dipti** - Sr. Data Quality Manager  
Healthcare Data Engineering Specialist  
[LinkedIn](#) | [GitHub](#) | [Portfolio](#)

---

## 📄 License

This framework documentation is provided as a portfolio demonstration piece for professional purposes.

---

*Built with expertise in QA automation, ETL testing, and healthcare data analytics.*
*Designed for modern cloud data platforms at enterprise scale.*
