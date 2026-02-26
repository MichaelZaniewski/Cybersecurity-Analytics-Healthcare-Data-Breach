# Data Cleaning and Exploratory Data Analysis

## Excel 
### Data Formatting and EDA
- Are there any invlid formats?
- Are there impossible values (negatve charges, dates out of range, blank PK IDs)
- Duplicate Hunting 
- Formatting Date

## SQL Exploratory Data Analysis

#### 1. Do the exposed columns look more like a billing vendor export, a clearinghouse submission, or an internal report?
```
SELECT * FROM claims
```
#### 2. How many total records and unique patients are there?
![Q2](https://github.com/user-attachments/assets/f9762f20-ca87-4400-977b-3acfc6bade66)
```
SELECT  COUNT(*) AS TOTAL_LEAKED_RECORDS,
        COUNT(DISTINCT PATIENT_ID) AS TOTAL_PATIENTS_AFFECTED
FROM CLAIMS
```
#### 3. What is the time frame of the leak?
![Q3](https://github.com/user-attachments/assets/5b974fec-7553-4a85-94c2-1492a62022ca)
```
SELECT  MIN(billing_date) AS oldest_record,
	     	MAX(billing_date) AS newest_record
FROM claims
```
#### 4. What percentage of patients have at least one record with an address to contact them by mail?
![Q4](https://github.com/user-attachments/assets/5eb3fa1b-2ce1-4206-8445-bce1cde8e19a)
```
SELECT  COUNT(DISTINCT patient_id) AS total_patients,
 	     TO_CHAR(ROUND(COUNT(DISTINCT patient_id) FILTER(WHERE address IS NOT NULL)
     / (COUNT(DISTINCT patient_id)::numeric)*100,2),'999D99%') AS contact_feasability
FROM claims
```
#### 5. What percentage of records include "high-risk combos" like name, SSN, insurance policy number?
![Q5](https://github.com/user-attachments/assets/51645b32-27c0-4a3d-8d99-4c4211d08f81)
```
SELECT  COUNT(*) AS total_records,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE SSN_full IS NOT NULL) / COUNT(*)::numeric),2),'999D99%') AS "ssn_alone",
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name IS NOT NULL AND ssn_full IS NOT NULL) / COUNT(*)::numeric),2),'999D99%') AS "name+ssn",
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name IS NOT NULL AND card_number IS NOT NULL) / COUNT(*)::numeric),2),'999D99%') AS "name+card_info",
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name IS NOT NULL AND insurance_policy_number IS NOT NULL) / COUNT(*)::numeric),2),'999D99%') AS "name+insurance_num",
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name IS NOT NULL AND address IS NOT NULL AND ssn_full IS NOT NULL) / COUNT(*)::numeric),2),'999D99%') AS "name+address+ssn"
FROM claims
```
#### 6. How many patients have had enough exposed data to warrant risk of identity theft vs low-risk exposure vs no exposure
![Q6](https://github.com/user-attachments/assets/a432a1fe-9888-4a77-9b03-66855e2487df)
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
#### 7. Which payment statuses dominate?
![Q7](https://github.com/user-attachments/assets/3e78e8a0-1c6d-4e73-a652-7b1c515dcd77)
```
SELECT COUNT(payment_status) AS total_bills,
	     COUNT(payment_status) FILTER(WHERE payment_status = 'In-progress') as in_progress,
	     COUNT(payment_status) FILTER(WHERE payment_status = 'Paid') AS paid,
	     COUNT(payment_status) FILTER(WHERE payment_status = 'Late-Paid') AS late_paid,
	     COUNT(payment_status) FILTER(WHERE payment_status = 'Late-Unpaid') AS late_unpaid
FROM claims
```

