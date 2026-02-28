![Image](https://github.com/user-attachments/assets/fd3d8b2e-69ce-4073-8550-6cf19c46591d)
# Cybersecurity-Analytics-Healthcare-Data-Breach

A cybersecurity analysis of simulated breached insurance claims data for exposed PII patterns, risk levels, and notification readiness with SQL and Tableau.

## Background and Overview
Cybersecurity workflows are becoming increasingly critical as healthcare operations continue shifting to digital systems and third-party software tools. When a breach occurs, teams are not just answering “what was stolen.” They have to determine what was exposed, how severe it is, who needs to be notified, and how to contact them. That requires reliable analytics around Personally Identifiable Information (PII) and Protected Health Information (PHI), plus an understanding of how real breach artifacts look. Partial exports, missing fields, duplicates, and inconsistent formatting are common, and they directly impact breach assessment, breach notification, and notification readiness under requirements like the Health Insurance Portability and Accountability Act (HIPAA).

For this project, **exposure** is defined using a breach-response threshold based on practical claims review workflows. A record is only counted as exposed if it contains **more than basic identifiers**. Rows with only a full name, address, or other publicly available information, are not treated as exposed. A record becomes exposed only when at least one additional sensitive **data element** is present (for example: insurance policy number, SSN fields, or card payment fields). Once that threshold is met, all available fields in that record are treated as exposed for severity scoring and notification readiness analysis. This prevents inflated exposure counts driven by information that is often publicly discoverable on its own and keeps metrics aligned with breach response decision-making.

The purpose of this project is to simulate a realistic insurance claims style breach and analyze it the way a breach response team would. Instead of treating the incident as a clean database dump, the breached dataset is intentionally incomplete and inconsistent at the record level to mirror how real leaks often surface, fragmented exports, missing fields, and uneven exposure across patients. The breach is modeled as a Revenue Cycle Management (RCM) and claims style export, so the leaked data is centered on billing and payer workflows: patient identifiers and basic demographics, contact details, insurance provider and policy fields, and claim-level billing elements like service dates, charges, balances, and payment status. Because these fields can be combined to enable identity theft and billing fraud, the analysis focuses on exposure severity, risk tiers, and notification readiness to support fast, defensible response decisions.

The scope of this project is one hypothetical NYC-based hospital with 50,000 patients over the past 5 years

**Insights and analysis are provided on the following key areas:**
- **Exposure severity:** what share of affected patients have high-risk PII and PHI combinations.
- **Identity theft risk tiers:** which records contain enough exposed data for meaningful misuse versus lower-risk exposure.
- **Notification readiness:** how many records are incomplete enough to complicate outreach (missing contact info, ambiguous identifiers).
- **Breach patterning:** which sensitive fields show the highest co-exposure in a claims and billing style export.
- **Operational context:** what the exposed fields imply about which workflow or system produced the export, and what that means for containment and remediation.

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
This analysis points to a breach shaped like a fragmented claims or revenue-cycle export. The exposed records contain a mix of billing, coverage, and identity-related fields, while missingness remains uneven across the dataset. The combination suggests an operational workflow leak tied to payer and billing activity, rather than a simple demographic file or a clean full-table dump.

The results also show that the incident would create two immediate response priorities:

1) Most affected patients fall into the highest risk tier under the project’s exposure rules, which means the leaked field combinations are serious enough to support identity theft or fraud concerns.
2) At the same time, patient notification remains broadly feasible for most of the affected population, even though the underlying records are incomplete and inconsistent at the row level. 

Geographically, the breach appears concentrated around the hospital’s home market but still extends well beyond it. That pattern fits the project’s single-hospital design and shows how a local operational breach can still create a multi-state response burden. From a business perspective, the dataset reflects an incident where severity assessment, notification readiness, and jurisdictional reach would all need to be evaluated together.

### Findings
- **High patient-level risk concentration:** Nearly 44,000 of 50,000 patients have enough exposed information across their records to be classified as high risk for identity theft or fraud. Almost 5,000 are low risk, and only about 1,000 fall into a no-risk bucket.
- **Sensitive fields are common in the leak:** Insurance-related data appears in 36.61% of records, financial information in 35.97%, and full Social Security numbers in 17.86%.
- **Most patients are still contactable:** 44,077 patients meet the project’s notification readiness standard, while 5,923 have records incomplete enough to make postal notification difficult.
- **The breach looks fragmented, not complete:** ssn_full is missing in 82.15% of rows and financial information is missing in 64.03%, which supports the project assumption (and design) that the breach resembles a partial claims export rather than a full structured database dump.
- **The impact is regionally concentrated but multi-state:** 38,722 affected patients are in New York, but the breach still spans 27 states, with the largest out-of-state counts in New Jersey, Massachusetts, and Pennsylvania

## Recommendations

- **Prioritize response by exposure severity:** Patients in the high-risk bucket should be triaged first for notification and follow-up because their records contain the combinations of fields most associated with identity theft and fraud risk.
- **Split outreach by notification readiness:** Records with complete `name` and `address` information can move directly into a standard notification workflow, while incomplete records should be routed to a separate remediation process.
- **Plan for a multi-state response footprint:** Although the breach is concentrated locally, the affected population spans multiple states, which would increase the coordination burden for notification and compliance handling.


## Assumptions, Limitations, and Caveats
### Assumptions
- The breach is modeled as a claims and revenue-cycle export, not a full database dump. Exposure is only counted when a record contains at least one qualifying sensitive data element beyond basic identifiers, and patient-level risk is based on the highest-risk row tied to that patient.
- The source organization is assumed to be a single hypothetical NYC-based hospital with a mostly local patient population and a smaller out-of-state tail. Notification readiness is evaluated primarily around likely mail contactability using preserved name and address fields.
  
### Limitations
- Synthetic data, despite realistic rules, may not capture true clinical variation like seasonal spikes. Real-world behavior from patients, providers, and payers can differ.
- The dataset’s missingness patterns are generated by project rules, not by an actual breached system, and the risk tiers are a project-defined analytical framework rather than a legal or regulatory standard.

### Caveats
- This project measures exposure potential and breach-response implications, not confirmed fraud, financial loss, or forensic root cause. Similar co-missingness percentages should also be interpreted carefully because ssn_full is missing in most rows, which limits how much the filtered population differs from the full dataset.

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
