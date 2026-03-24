---
description: "Check regulatory compliance (GDPR, SOC2, PCI-DSS, HIPAA). Use when user says 'compliance check', 'GDPR kontrol', 'KVKK', 'SOC 2', 'PCI-DSS', 'HIPAA', 'uyumluluk kontrol', 'regulatory check'."
---

# Nazar Compliance Check Skill

You are performing a regulatory compliance audit at the code level. Follow these instructions systematically.

## Step 1: Determine Compliance Framework

**Option A: User specifies a framework**
If the user explicitly mentions GDPR, KVKK, SOC 2, PCI-DSS, or HIPAA, use that framework.

**Option B: Auto-detect from project context**
If the user says "compliance check" without specifying, detect the relevant framework:

1. **GDPR/KVKK indicators:**
   - European user base (`.eu`, `.de`, `.fr`, `.tr` domains)
   - Cookie consent implementation
   - Privacy policy references
   - Data export/deletion features
   - Turkish language content (KVKK)
   - EU/TR payment processors

2. **SOC 2 indicators:**
   - SaaS application
   - Multi-tenant architecture
   - Enterprise customer features
   - Audit logging
   - SSO/RBAC implementation

3. **PCI-DSS indicators:**
   - Payment processing (Stripe, Braintree, PayPal)
   - Credit card data handling
   - E-commerce functionality
   - Card number fields/patterns

4. **HIPAA indicators:**
   - Health/medical data fields
   - Patient records
   - HL7/FHIR references
   - Healthcare terminology
   - PHI/ePHI references

If multiple frameworks are relevant, ask the user which to prioritize or offer to run all applicable frameworks.

## Step 2: Load the Framework Reference

Read the corresponding framework file:

- **GDPR/KVKK:** Read `skills/compliance/frameworks/gdpr.md`
- **SOC 2:** Read `skills/compliance/frameworks/soc2.md`
- **PCI-DSS:** Read `skills/compliance/frameworks/pci-dss.md`
- **HIPAA:** Read `skills/compliance/frameworks/hipaa.md`

## Step 3: Systematic Compliance Check

For each check item in the loaded framework:

1. **Run all grep patterns** from the framework item against the project codebase using the Grep tool.
2. **For each match**, use Read to examine context (at least 15 lines).
3. **Apply pass/fail criteria** from the framework to determine compliance status.
4. **Classify each item:**
   - **COMPLIANT**: All criteria met
   - **PARTIAL**: Some criteria met, gaps identified
   - **NON-COMPLIANT**: Criteria not met, significant gaps
   - **NOT APPLICABLE**: Check does not apply to this project

## Step 4: Generate Compliance Report

Present findings in this format:

```
## Compliance Audit Report

**Framework:** [GDPR/KVKK | SOC 2 Type II | PCI-DSS v4.0 | HIPAA]
**Project:** [project name]
**Audit Date:** [current date]

### Compliance Summary

| # | Check Item | Status | Risk Level |
|---|-----------|--------|------------|
| 1 | [item name] | COMPLIANT/PARTIAL/NON-COMPLIANT/N/A | HIGH/MEDIUM/LOW |
| 2 | ... | ... | ... |

**Overall Score:** X/Y checks compliant (Z%)

---

### Detailed Findings

#### 1. [Check Item Name]

**Status:** COMPLIANT / PARTIAL / NON-COMPLIANT / NOT APPLICABLE

**Evidence Found:**
- [What was found in the codebase, with file:line references]

**Gaps Identified:**
- [What is missing or insufficient]

**Risk Assessment:**
- Likelihood: [HIGH/MEDIUM/LOW]
- Impact: [HIGH/MEDIUM/LOW]
- Risk Level: [CRITICAL/HIGH/MEDIUM/LOW]

**Remediation:**
- [Specific steps to close the gap]
- [Code example if applicable]

**Priority:** P1/P2/P3/P4

---

[Repeat for each check item]

### Compliance Gap Analysis

| Gap | Risk | Remediation | Effort | Priority |
|-----|------|-------------|--------|----------|
| ... | ... | ... | Low/Med/High | P1-P4 |

### Recommendations

1. **Immediate (P1):** [Critical gaps that must be addressed now]
2. **Short-term (P2):** [High-risk gaps for next sprint]
3. **Medium-term (P3):** [Medium-risk improvements]
4. **Long-term (P4):** [Best practice enhancements]
```

## Step 5: Offer Next Steps

After the report, suggest:
- "Use `/nazar:security-audit` for a deeper OWASP security analysis"
- "Use `/nazar:fix` to auto-fix applicable compliance gaps"
- Specific third-party tools for advanced compliance testing

## Important Notes

- Compliance is not just a code-level concern. Note when organizational/process controls are needed beyond code.
- Be specific about what constitutes compliance vs. non-compliance for each check.
- Consider the project's deployment environment (cloud provider, region) for compliance context.
- For GDPR/KVKK: consider both the technical implementation and the legal requirements.
- For HIPAA: clearly distinguish between Required and Addressable specifications.
- Do not report compliance for items you cannot verify from code alone; note them as "Requires organizational verification".
- Provide actionable, specific remediation steps with code examples where possible.
