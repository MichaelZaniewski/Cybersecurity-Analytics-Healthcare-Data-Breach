-- PostgreSQL


```
-- PostgreSQL

DROP TABLE IF EXISTS public.claims_export;

CREATE TABLE public.claims_export (
  billing_id                   INTEGER PRIMARY KEY,
  visit_id                     INTEGER NOT NULL,
  patient_id                   INTEGER NOT NULL,

  billing_date                 DATE NOT NULL,
  insurance_coverage_amount     NUMERIC(10,2) NOT NULL,
  patient_responsibility_amount NUMERIC(10,2) NOT NULL,
  payment_plan                 VARCHAR(30),
  expected_payment_date        DATE,
  actual_payment_date          DATE,
  payment_status               VARCHAR(30),
  total_charge                 NUMERIC(10,2) NOT NULL,

  name                         VARCHAR(120),
  date_of_birth                DATE,
  address                      VARCHAR(200),
  city                         VARCHAR(100),
  state                        CHAR(2),
  zipcode                      CHAR(5),

  insurance_provider           VARCHAR(120),
  insurance_policy_number      VARCHAR(60),
  phone_number                 VARCHAR(25),

  ssn_full                     CHAR(11),   -- 123-45-6789
  ssn_last4                    CHAR(11),   -- ***-**-6789

  card_brand                   VARCHAR(20),
  card_number                  CHAR(19),   -- ####-####-####-####

  cvv                          CHAR(3),    -- 3 digits
  exp_date                     CHAR(7)     -- MM/YY
);
```
