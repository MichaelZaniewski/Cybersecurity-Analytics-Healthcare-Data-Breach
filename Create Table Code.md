-- PostgreSQL


```
CREATE TABLE claims_export (
  billing_id                      INTEGER PRIMARY KEY,
  visit_id                        INTEGER NOT NULL,
  patient_id                      INTEGER NOT NULL,

  billing_date                    DATE NOT NULL,

  insurance_coverage_amount        NUMERIC(10,2) NOT NULL,
  patient_responsibility_amount    NUMERIC(10,2) NOT NULL,
  total_charge                     NUMERIC(10,2) NOT NULL,

  payment_plan                    VARCHAR(30),
  expected_payment_date           DATE,
  actual_payment_date             DATE,
  payment_status                  VARCHAR(30),

  name                            VARCHAR(120),
  date_of_birth                   DATE,
  address                         VARCHAR(200),
  city                            VARCHAR(100),
  state                           CHAR(2),
  zipcode                         CHAR(5),

  insurance_provider              VARCHAR(120),
  insurance_policy_number         VARCHAR(60),
  phone_number                    VARCHAR(25),

  SSN_full                        CHAR(11),   -- 123-45-6789
  SSN_last4                       CHAR(11),   -- ***-**-6789

  card_brand                      VARCHAR(20),
  card_number                     CHAR(19),   -- ####-####-####-#### (19 chars incl dashes)
  CVV                             CHAR(3),    -- 3 digits
  exp_date                        CHAR(5)     -- MM/YY
);

CREATE INDEX idx_claims_export_visit_id        ON claims_export (visit_id);
CREATE INDEX idx_claims_export_patient_id      ON claims_export (patient_id);
CREATE INDEX idx_claims_export_billing_date    ON claims_export (billing_date);
```
