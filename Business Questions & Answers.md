# Business Questions & SQL Query Results

### 1) What percentage of breached records included highly sensitive fields like Social Security Numbers or financial data?
![Q1](https://github.com/user-attachments/assets/aaecad39-b7fc-4e64-a0d0-e6e030c71ee9)
```
SELECT COUNT(*) AS total_records,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE ssn_full <> '') / COUNT(*)::numeric),2),'999D99%') AS has_full_ssn,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE card_number <> '' AND cvv <> '' AND exp_date <> '') / COUNT(*)::numeric),2),'999D99%') as has_financial_info,
		TO_CHAR(ROUND(100*(COUNT(*) FILTER(WHERE name <> '' AND date_of_birth IS NOT NULL AND insurance_policy_number <> '') / COUNT(*)::numeric),2),'999D99%') AS has_insurance_info
FROM claims
```
### 2) Are certain types of PII (e.g., name, address) more frequently exposed together?
![Q2]()
```

```
### 3) How many affected individuals have enough exposed information to pose a serious identity theft risk vs low risk exposure vs no exposure
![Q3](https://github.com/user-attachments/assets/a432a1fe-9888-4a77-9b03-66855e2487df)
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
### 4) Are there patterns in the types of data that are more likely to be missing?
![Q4]()
```

```
### 5) What proportion of records are incomplete to the point where patient notification would be difficult?
![Q5](https://github.com/user-attachments/assets/8855d402-a01e-420d-806c-737f0685b0f3)
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
