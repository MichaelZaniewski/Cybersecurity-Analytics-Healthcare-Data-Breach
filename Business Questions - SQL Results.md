# Business Questions & SQL Query Results

### 1) What percentage of breached records included highly sensitive fields like Social Security Numbers or financial data?
![Q1](https://github.com/user-attachments/assets/aaecad39-b7fc-4e64-a0d0-e6e030c71ee9)
- SOME BACKGROUND: Sensitive fields evaluated in this section were `ssn_full`, the financial bundle of `card_number` + `cvv` + `exp_date`, and an insurance-related identity bundle of `name` + `date_of_birth` + `insurance_policy_number`.
- EXPLANATION OF CHART: Insurance-related identity data, defined in this query as `name` + `date_of_birth` + `insurance_policy_number`, appears in 36.61% of all records, with complete financial information close behind at 35.97%. Full Social Security numbers were exposed in 17.86% of records.

```
SELECT COUNT(*) AS total_records,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE ssn_full <> '') / COUNT(*)::numeric),2),'999D99%') AS has_full_ssn,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE card_number <> '' AND cvv <> '' AND exp_date <> '') / COUNT(*)::numeric),2),'999D99%') as has_financial_info,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name <> '' AND date_of_birth IS NOT NULL AND insurance_policy_number <> '') / COUNT(*)::numeric),2),'999D99%') AS has_insurance_info
FROM claims
```

### 2) How many affected individuals have enough exposed information to pose a serious identity theft risk vs low risk exposure vs no exposure
![Q3](https://github.com/user-attachments/assets/a432a1fe-9888-4a77-9b03-66855e2487df)
- EXPLANATION OF CHART: A significant number of patients, nearly 44K out of 50K, have enough information across their leaked records to constitute a serious threat of identity theft or fraud. Almost 5K patients had enough exposed information to create a lower level of risk, while roughly 1K patients experienced no meaningful threat based on the rules used below.
  
- TECHNICAL POINT: Risk buckets were created with a `CASE` expression at the patient level, then rolled up with `MAX(...)` across all records for each `patient_id` so every patient is classified by their highest observed level of exposure. The outer query then counts how many patients fall into each final risk bucket.

Risk was determined as the presence of any one of the following:
- High-risk: `ssn_full`, `card_number`, `insurance_policy_number` + `name`
- Low-risk: `ssn_last4` + `date_of_birth`, `date_of_birth` + `name`
- No risk: any other combination of fields, for instance `phone_number`, `address`, `total_charge`, etc.

```
SELECT  COUNT(*) as count,
	CASE risk_factor 
		WHEN 2 THEN 'high_risk_identity_or_fraud'
		WHEN 1 THEN 'low_risk_exposure'
		ELSE 'no_risk'
		END as risk_bucket
FROM
  (SELECT patient_id,
	MAX(CASE
	WHEN ssn_full IS NOT NULL THEN 2
	WHEN card_number IS NOT NULL THEN 2
	WHEN insurance_policy_number IS NOT NULL 
		AND name IS NOT NULL THEN 2
	WHEN ssn_last4 IS NOT NULL 
		AND date_of_birth IS NOT NULL THEN 1
	WHEN insurance_policy_number IS NOT NULL 
		AND name IS NOT NULL THEN 1
	WHEN name IS NOT NULL 
		AND date_of_birth IS NOT NULL THEN 1
	ELSE 0
	END) as risk_factor
	FROM claims
	GROUP BY patient_id
	)
GROUP BY risk_factor
ORDER BY count DESC
```

### 3) What proportion of records are incomplete to the point where patient notification would be difficult?
![Q5](https://github.com/user-attachments/assets/8855d402-a01e-420d-806c-737f0685b0f3)
- EXPLANATION OF CHART: The majority of patients, 44,077, have enough data present in the breach to be considered easily contactable. Another 5,923 patients have records incomplete enough to make notification difficult.
- TECHNICAL POINT: This logic evaluates contactability at the patient level rather than the row level. The `MAX(CASE ...)` pattern checks all records for a patient and marks them as `easy` if any record contains both `name` and `address`, which avoids understating notification readiness when contact information is spread across multiple claims.
- SOME BACKGROUND: Only `name` + `address` were treated as reliable contact methods. Phone numbers and emails were not counted because the notification method assumed in this scenario is postal mail.

```
SELECT 
	CASE address_flag 
		WHEN 1 THEN 'easy'
		ELSE 'difficult'
		END AS contact_feasability, COUNT(*) AS patient_count
		
FROM
	(SELECT  
	patient_id,
	MAX(CASE 
		WHEN address IS NOT NULL AND name IS NOT NULL then 1
		ELSE 0
		END) AS address_flag
	FROM claims
	GROUP BY patient_id)
	GROUP BY contact_feasability
	ORDER BY patient_count DESC
```

### 4) Are there patterns in the types of data that are more likely to be missing?
#### Step 1: Which type of data is more likely to be missing?
![Q4a](https://github.com/user-attachments/assets/e71e3f2b-f9ed-41c8-b0a3-56f5a1a87a39)
- EXPLANATION OF CHART: `ssn_full` is most likely to be missing at 82.15%. Complete financial information is missing at 64.03%, while insurance policy information, name, address, and date of birth are each missing at rates that fall roughly between 21% and 26%.
- TECHNICAL POINT: This query measures missingness with `COUNT(*) FILTER (WHERE ...)` so each field or field bundle can be compared against the same total record count. The financial and address checks are intentionally treated as bundles, meaning the query only counts them as missing when all related fields in that group are absent together.
```
SELECT count(*) AS total_records,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE ssn_full IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_ssn_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_name_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE card_number IS NULL AND cvv IS NULL AND exp_date IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_financial_info_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE insurance_policy_number IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_policy_num_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE address IS NULL AND city IS NULL and zipcode IS NULL) / COUNT(*)::numeric),2),'999D99%') AS pcnt_address_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE date_of_birth IS NULL) / COUNT(*)::numeric),2),'999D99%') AS pcnt_dob_missing
FROM claims
```
#### Step 2: Are there patterns in the missing data? 
![Q4b](https://github.com/user-attachments/assets/93bdf515-b231-41b9-b3c9-68ae6bfafe57)
- EXPLANATION OF CHART: When filtering to rows where `ssn_full` is NULL, `insurance_policy_number` shows the highest co-missingness among the fields tested. That means those two fields are absent together more often than the other categories in this subset of records.
- TECHNICAL POINT: This second query re-runs the missingness test on only the `ssn_full IS NULL` subset so the output can be compared against the overall rates from Step 1. It is a conditional missingness check, designed to see whether missing SSNs meaningfully change the probability that other fields are also missing.
- SOME BACKGROUND: In a real breach, tandem missingness can be a useful clue about the source system or extraction workflow because it can suggest whether the leak came from a billing feed that strips member identifiers or from a dataset that was partially redacted before release.
```
SELECT  COUNT(*) AS total_rows_ssn_full_null,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_name_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE card_number IS NULL AND cvv IS NULL AND exp_date IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_financial_info_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE insurance_policy_number IS NULL) / COUNT(*)::numeric),2),'999D99%') as pcnt_policy_num_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE address IS NULL AND city IS NULL and zipcode IS NULL) / COUNT(*)::numeric),2),'999D99%') AS pcnt_address_missing,
TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE date_of_birth IS NULL) / COUNT(*)::numeric),2),'999D99%') AS pcnt_dob_missing
FROM claims
WHERE ssn_full IS NULL
```
The results are statistically insignificant for two reasons: 
1) ssn_full is NULL in 82% of rows, so filtering to ssn_full IS NULL barely changes the population mix
2) The Python nulling logic applies most other fields independently of SSN status, so conditional missingness rates stay close to the overall rates.
   
In a real breach dataset, this check matters because correlated missingness often points to specific extraction workflows or masking failures, which changes exposure severity, identity theft risk, and who needs to be notified.

#### 5. What share of affected individuals are in NY vs out-of-state
![Q5](https://github.com/user-attachments/assets/c5f10d91-6dba-4e37-8a06-38df2b954d8d)
- EXPLANATION OF CHART: As expected for a New York City-based hospital, the largest share of affected individuals live in New York. A total of 38,722 affected patients reside in NY.
- SOME BACKGROUND: Because this project models a New York City-based hospital, the majority of patients are expected to live in New York State. At the same time, nearby states should still appear in meaningful numbers because regional hospitals can draw patients from surrounding areas. This matters in breach response because notification requirements and compliance timelines can vary by state.
- Neighboring states have higher patient counts, with 2,438 from New Jersey, 2,285 from Massachusetts, and 1,646 from Pennsylvania, followed by a steep drop-off after that.
- There are 27 distinct states represented in this dataset.
- TECHNICAL POINT: Many patient records do not include an entry in the `state` field. To avoid counting one patient multiple times across multiple claims, the query first collapses all rows to one state per `patient_id` using `COALESCE(MAX(state) FILTER (WHERE state IS NOT NULL), 'no_state_listed')`. Even after that patient-level grouping, 2,454 affected patients still have no state listed.
```
SELECT patient_state, count(affected_flag) as patient_count
FROM (SELECT
	patient_id, COALESCE(MAX(state) FILTER(WHERE state IS NOT NULL), 'no_state_listed') as patient_state,
	MAX(CASE 
		WHEN SSN_full IS NOT NULL
		OR insurance_policy_number IS NOT NULL
		OR (name IS NOT NULL AND date_of_birth IS NOT NULL)
		OR card_number IS NOT NULL
		THEN 1
		ELSE 0
		END) AS affected_flag
	FROM claims
	GROUP BY patient_id
	ORDER BY patient_id ASC)
WHERE affected_flag = 1
GROUP BY patient_state
ORDER BY patient_count DESC
```
