![Image](https://github.com/user-attachments/assets/fd3d8b2e-69ce-4073-8550-6cf19c46591d)
# Cybersecurity-Analytics-Healthcare-Data-Breach

A cybersecurity analysis of simulated breached insurance claims data for exposed PII patterns, risk levels, and notification readiness with SQL and Tableau.

## Background and Overview
Cybersecurity workflows are becoming increasingly critical as healthcare operations continue shifting to digital systems and third-party software tools. When a breach occurs, teams are not just answering “what was stolen.” They have to determine what was exposed, how severe it is, who needs to be notified, and how to contact them. That requires reliable analytics around Personally Identifiable Information (PII) and Protected Health Information (PHI), plus an understanding of how real breach artifacts look. Partial exports, missing fields, duplicates, and inconsistent formatting are common, and they directly impact breach assessment and notification readiness under requirements like the Health Insurance Portability and Accountability Act (HIPAA).

For this project, **exposure** is defined using a breach-response threshold based on practical claims review workflows. A record is only counted as exposed if it contains **more than basic identifiers**. Rows with only a full name, address, or other publicly available information, are not treated as exposed. A record becomes exposed only when at least one additional sensitive **data element** is present (for example: insurance policy number, SSN fields, or card payment fields). Once that threshold is met, all available fields in that record are treated as exposed for severity scoring and notification readiness analysis. This prevents inflated exposure counts driven by information that is often publicly discoverable on its own and keeps metrics aligned with breach response decision-making.

The purpose of this project is to simulate a realistic insurance claims style breach and analyze it the way a breach response team would. Instead of treating the incident as a clean database dump, the breached dataset is intentionally incomplete and inconsistent at the record level to mirror how real leaks often surface, fragmented exports, missing fields, and uneven exposure across patients. The breach is modeled as a revenue-cycle and claims style export, so the leaked data is centered on billing and payer workflows: patient identifiers and basic demographics, contact details, insurance provider and policy fields, and claim-level billing elements like service dates, charges, balances, and payment status. Because these fields can be combined to enable identity theft and billing fraud, the analysis focuses on exposure severity, risk tiers, and notification readiness to support fast, defensible response decisions.

The scope of this project is one hypothetical NYC-based hospital with 50,000 patients over the past 5 years

**Insights and analysis are provided on the following key areas:**
- **Exposure severity:** what share of affected patients have high-risk PII and PHI combinations
- **Identity theft risk tiers:** which records contain enough exposed data for meaningful misuse versus lower-risk exposure
- **Notification readiness:** how many records are incomplete enough to complicate outreach (missing contact info, ambiguous identifiers)
- **Breach patterning:** which data elements co-occur most often in a claims and billing style export
- **Operational context:** what the exposed fields imply about which workflow or system produced the export, and what that means for containment and remediation

## Dataset Generator and Structure Overview

### Generator Logic
This project utilizes version 2.0 of my [Healthcare Dataset Generator](https://github.com/MichaelZaniewski/Healthcare-Dataset-Generator/tree/main) that programmatically creates realistic, U.S.-based healthcare data that mirrors hospital operations with **no real or sensitive data**. That data was further augmented by joining the `billing` and `patient` tables, generating additional PII (SSNs, credit card info), and randomly nulling records with an additional python script that can be found [here](https://github.com/MichaelZaniewski/Cybersecurity-Analytics-Healthcare-Data-Breach/blob/main/Dataset%20Generation%20Information.md#additional-pii-generation-script). The culimnation of these steps produced the final dataset to be analyzed, which can be found [here](https://github.com/MichaelZaniewski/Cybersecurity-Analytics-Healthcare-Data-Breach/blob/main/Dataset.zip). 

### Data Structure and Type
| column name                   | data type           | column name                 | data type           |
| ----------------------------- | ------------------- | --------------------------- | ------------------- |
| billing_id                    | integer             | visit_id                    | integer             |
| patient_id                    | integer             | billing_date                | date                |
| insurance_coverage_amount     | numeric(10,2)       | patient_responsibility_amount | numeric(10,2)     |
| payment_plan                  | character varying   | expected_payment_date       | date                |
| actual_payment_date           | date                | payment_status              | character varying   |
| total_charge                  | numeric(10,2)       | name                        | character varying   |
| date_of_birth                 | date                | address                     | character varying   |
| city                          | character varying   | state                       | character varying(2)|
| zipcode                       | character varying(5)| insurance_provider          | character varying   |
| insurance_policy_number       | character varying   | phone_number                | character varying   |
| ssn_full                      | character varying   | ssn_last4                   | character varying   |
| card_brand                    | character varying   | card_number                 | character varying   |
| cvv                           | character varying(3)| exp_date                    | character varying   |




## Executive Summary
### Overview of Findings

### Findings

## Recommendations




### EXTRA CATEGORY WRAP-UP

#### CATEGORY Explanation



## Assumptions, Limitations, and Caveats
### Assumptions

### Limitations
- Synthetic data, despite realistic rules, may not capture true clinical variation like seasonal spikes. Real‑world behavior (patients, providers, payers) can differ.
### Caveats


## Glossary
- **Data Element:** A discrete unit of sensitive information used in breach review to determine exposure severity (example: insurance policy number, SSN fields, payment card fields). In this project, exposure is only counted when at least one qualifying data element is present beyond basic identifiers like name and address.
- **Claims (insurance claims data):** Records used to request and process payment from an insurer for healthcare services. Typically includes patient identifiers, dates of service, provider/facility info, charges, amounts paid, and claim status.
- **Revenue Cycle (Revenue Cycle Management, RCM):** The end-to-end process of capturing patient/insurance info, billing, submitting claims, receiving payments/denials, and collecting any remaining patient balance.
- **Personally Identifiable Information (PII):** Data that can identify a specific person on its own or when combined with other fields. Examples: name, address, date of birth, SSN, government ID.
- **Protected Health Information (PHI):** Health-related information tied to an individual and created/used for care or payment. Examples: treatments, insurance policy numbers, test results.
- **Breach Notification:** The process of notifying affected individuals after exposure of sensitive data including what was exposed, how many people were affected, and applicable guidelines.
- **Notification Readiness:** How actionable the leaked records are for contacting affected individuals. High readiness means at least one reliable contact path (name, full address + PII). Low readiness means missing or conflicting contact fields.
- **Exposure:** A field or record being present in the leaked dataset
- **Co-exposure:** Two or more sensitive fields being exposed together in the same record (example: name + DOB + policy number).
- **Risk Tier:** A severity bucket used to classify records or individuals based on exposure (low, medium, high).
