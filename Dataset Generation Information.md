# Python Dataset Generation Information

## Foundational Generation 
This project utilizes version 2.0 of my [Healthcare Dataset Generator](https://github.com/MichaelZaniewski/Healthcare-Dataset-Generator/tree/main).

The generator programmatically creates realistic, U.S.-based healthcare data that mirrors hospital operations with **no real or sensitive data**

## Expanded Creation
### The Goal
This projects dataset is designed to simulate an **insurance claims and revenue-cycle export breach**, which is one of the most common versions of healthcare leaks. In real incidents, attackers often obtain a claims-related file pulled for billing or clearinghouse workflows. That export typically contains enough identity and coverage detail to match patients to payers (names, demographics, addresses, insurance provider, policy numbers, billing dates, charges, balances). 

### The Execution
This project models that breach artifact directly. Rather than treating the breach as a clean database dump, I simulated a realistic claims export by joining the `billing` and `patients` tables into a single “claims-style” dataset. I then generated additional synthetic PII fields to reflect what often exists in revenue-cycle workflows, and applied controlled nulling to mimic partial exposure. The result is a dataset where each record is incomplete in different ways (some rows contain coverage and billing amounts but limited identity fields, others include contact fields but missing insurance details), which better mirrors how real breach exports look and how breach response teams have to assess impact.


## Additional PII Generation Script

After creating the claims_export table, this script was ran against the dataset to add increased PII severity and realism through partial/missing data. 

```
"""
Input: claims_export.csv (expects columns: billing_id, patient_id)
"""

from __future__ import annotations

import argparse
import hashlib
import random
from datetime import date
from pathlib import Path

import pandas as pd


def _sha256_hex(s: str) -> str:
    return hashlib.sha256(s.encode("utf-8")).hexdigest()


def stable_u01(key: str, seed: int, salt: str) -> float:
    """Deterministic pseudo-random float in [0, 1)."""
    h = _sha256_hex(f"{seed}|{salt}|{key}")
    x = int(h[:16], 16)  # 64-bit chunk
    return (x % 10**12) / 10**12


def stable_int(key: str, seed: int, salt: str, mod: int) -> int:
    """Deterministic pseudo-random int in [0, mod)."""
    h = _sha256_hex(f"{seed}|{salt}|{key}")
    x = int(h[:16], 16)
    return x % mod



def auto_rate(seed: int, salt: str, lo: float = 0.05, hi: float = 0.25) -> float:
    """Deterministically pick a rate in [lo, hi] from seed + salt."""
    u = stable_u01("default", seed, salt)
    return lo + (hi - lo) * u

def stable_rng(key: str, seed: int, salt: str) -> random.Random:
    """Deterministic RNG per key (patient_id)."""
    s = stable_int(key, seed, salt, 2**63 - 1)
    return random.Random(s)


def pick_brand(u: float) -> str:
    # Visa 55%, Mastercard 30%, Amex 10%, Discover 5%
    if u < 0.55:
        return "visa"
    if u < 0.85:
        return "mastercard"
    if u < 0.95:
        return "amex"
    return "discover"


def luhn_checksum(num: str) -> int:
    """Return checksum mod 10. Valid numbers have checksum == 0."""
    total = 0
    alt = False
    for ch in reversed(num):
        d = ord(ch) - 48
        if alt:
            d *= 2
            if d > 9:
                d -= 9
        total += d
        alt = not alt
    return total % 10


def random_16_with_dashes_invalid(rng: random.Random) -> str:
    """16 digits, grouped 4-4-4-4 with hyphens. Forced invalid (fails Luhn)."""
    digits = "".join(str(rng.randint(0, 9)) for _ in range(16))
    if luhn_checksum(digits) == 0:
        digits = digits[:-1] + str((int(digits[-1]) + 5) % 10)
    return "-".join(digits[i : i + 4] for i in range(0, 16, 4))


def random_3_2_4_invalid(rng: random.Random) -> tuple[str, str]:
    """SSN-like 3-2-4 format. Forced invalid by setting the middle group to 00."""
    a = rng.randint(100, 899)
    b = rng.randint(1, 99)
    c = rng.randint(0, 9999)
    full = f"{a:03d}-{b:02d}-{c:04d}"
    last4 = f"{c:04d}"
    return full, last4


def random_three_digits_no_leading_zeros(rng: random.Random) -> int:
    """A triple-digit number in xxx format (100–999)."""
    return rng.randint(100, 999)


def build_patient_sensitive(
    claims: pd.DataFrame,
    seed: int,
    p_ssn_full: float = 0.20,
    p_ssn_last4_blank: float = 0.05,
) -> pd.DataFrame:
    patients = (
        claims[["patient_id"]]
        .dropna()
        .astype(str)
        .drop_duplicates()
        .reset_index(drop=True)
    )
    n = len(patients)
    if n == 0:
        raise ValueError("No patient_id values found.")

    n_full = int(round(p_ssn_full * n))
    n_blank = int(round(p_ssn_last4_blank * n))

    patients["_u_full"] = patients["patient_id"].map(lambda pid: stable_u01(pid, seed, "pick_ssn_full"))
    patients["_u_blank"] = patients["patient_id"].map(lambda pid: stable_u01(pid, seed, "pick_ssn_last4_blank"))

    full_ids = set(patients.sort_values("_u_full").head(n_full)["patient_id"].tolist())

    non_full = patients[~patients["patient_id"].isin(full_ids)].copy()
    n_blank = min(n_blank, len(non_full))
    blank_ids = set(non_full.sort_values("_u_blank").head(n_blank)["patient_id"].tolist())

    ssn_full_out = []
    ssn_last4_out = []

    for pid in patients["patient_id"].tolist():
        rng = stable_rng(pid, seed, "ssn")
        full, last4 = random_3_2_4_invalid(rng)
        masked_last4 = f"***-**-{last4}"

        if pid in full_ids:
            ssn_full_out.append(full)
            ssn_last4_out.append(masked_last4)
        else:
            ssn_full_out.append(pd.NA)
            ssn_last4_out.append(pd.NA if pid in blank_ids else masked_last4)

    return pd.DataFrame(
        {
            "patient_id": patients["patient_id"],
            "SSN_full": ssn_full_out,
            "SSN_last4": ssn_last4_out,
        }
    )


def build_patient_card_profile(patient_ids: pd.Series, seed: int) -> pd.DataFrame:
    today = date.today()
    rows = []

    for pid in patient_ids.astype(str).tolist():
        rng = stable_rng(pid, seed, "card")

        card_brand = pick_brand(stable_u01(pid, seed, "brand"))
        card_number = random_16_with_dashes_invalid(rng)
        CVV = random_three_digits_no_leading_zeros(rng)

        month = stable_int(pid, seed, "exp_month", 12) + 1
        years_ahead = stable_int(pid, seed, "exp_years_ahead", 5) + 1
        yy = (today.year + years_ahead) % 100
        exp_date = f"{month:02d}/{yy:02d}"

        rows.append(
            {
                "patient_id": pid,
                "card_brand": card_brand,
                "card_number": card_number,
                "CVV": CVV,
                "exp_date": exp_date,
            }
        )

    return pd.DataFrame(rows)


def build_billing_payment(claims: pd.DataFrame, seed: int) -> pd.DataFrame:
    base = claims[["billing_id", "patient_id"]].dropna(subset=["billing_id", "patient_id"]).copy()
    base["billing_id"] = base["billing_id"].astype(str)
    base["patient_id"] = base["patient_id"].astype(str)

    bad = base.groupby("billing_id")["patient_id"].nunique()
    bad = bad[bad > 1]
    if len(bad) > 0:
        raise ValueError(
            f"{len(bad)} billing_id values map to multiple patient_id values. Fix the input join first."
        )

    billing_unique = base.drop_duplicates(subset=["billing_id"], keep="first").reset_index(drop=True)

    patient_profile = build_patient_card_profile(billing_unique["patient_id"].drop_duplicates(), seed=seed)
    out = billing_unique.merge(patient_profile, on="patient_id", how="left")

    return out[["billing_id", "card_brand", "card_number", "CVV", "exp_date"]]




def _make_row_key(df: pd.DataFrame) -> pd.Series:
    """
    Build a per-row key for deterministic row-level masking.
    Prefer billing_id when it is unique; otherwise include the row index.
    """
    if "billing_id" in df.columns:
        bid = df["billing_id"].astype("string")
        nonnull = bid.dropna()
        if len(nonnull) > 0 and nonnull.nunique(dropna=True) == len(nonnull):
            return bid.fillna("").astype(str)
        return (bid.fillna("").astype(str) + "|" + df.index.astype(str))
    return df.index.astype(str)


def _select_exact_mask(keys: pd.Series, seed: int, salt: str, p: float) -> pd.Series:
    """
    Deterministically select an exact fraction p of rows (rounded) based on stable_u01 ranking.
    Returns a boolean mask aligned to keys.index.
    """
    p = max(0.0, min(1.0, float(p)))
    n = int(len(keys))
    k = int(round(p * n))

    if n == 0:
        return pd.Series(dtype=bool)
    if k <= 0:
        return pd.Series(False, index=keys.index)
    if k >= n:
        return pd.Series(True, index=keys.index)

    u = keys.map(lambda x: stable_u01(str(x), seed, salt))
    chosen_idx = u.sort_values().head(k).index
    return pd.Series(keys.index.isin(chosen_idx), index=keys.index)


def apply_row_level_leak_nulls(
    df: pd.DataFrame,
    seed: int,
    p_card_patient_leak: float = 0.43,
    p_card_row_null_given_eligible: float = 0.10,
    p_name_null_rows: float = 0.10,
    p_dob_null_rows: float = 0.10,
    p_address_null_rows: float = 0.10,
    p_ssn_full_null_rows: float = 0.10,
    p_ssn_last4_null_rows: float = 0.10,
    p_card_brand_null_when_number_present_rows: float = 0.0,
) -> pd.DataFrame:
    """
    Row-level leak nulling for:
      - Card fields: select exactly p_card_patient_leak of UNIQUE patients as eligible, then leak per row.
      - Demographics: name/date_of_birth/address bundle nulling, independent per row.
      - SSN fields: SSN_full and SSN_last4 can be nulled per row.

    Card rules:
      - If card_number is null => CVV and exp_date are null.
      - If card_number is present => CVV and exp_date must be present.
      - card_brand is nulled whenever card_number is nulled.
      - card_brand may also be nulled on some rows where card_number is present.

    SSN rule:
      - If SSN_full is present on a row, SSN_last4 is forced to be present on that row.
    """
    out = df.copy()
    keys = _make_row_key(out)

    # Preserve original SSN_last4 so we can restore it when SSN_full is present
    ssn_last4_orig = out["SSN_last4"].copy() if "SSN_last4" in out.columns else None

    # -------------------------
    # Card: 43% of patients eligible, then leak per row
    # -------------------------
    if "patient_id" in out.columns and "card_number" in out.columns:
        pids = (
            out["patient_id"]
            .dropna()
            .astype(str)
            .drop_duplicates()
            .tolist()
        )
        n_pat = len(pids)
        n_leak = int(round(max(0.0, min(1.0, float(p_card_patient_leak))) * n_pat))

        ranked = sorted(pids, key=lambda pid: stable_u01(pid, seed, "pick_card_patient_leak"))
        leak_patients = set(ranked[:n_leak])

        pid_series = out["patient_id"].astype("string").astype(str)
        eligible_rows = pid_series.isin(leak_patients)

        # Independent per row (within eligible patients)
        u_keep = keys.map(lambda x: stable_u01(str(x), seed, "card_row_keep"))
        keep_card = eligible_rows & (u_keep >= float(p_card_row_null_given_eligible))

        # Guarantee at least one leaked row per eligible patient
        if leak_patients:
            forced_u = keys.map(lambda x: stable_u01(str(x), seed, "card_force_keep"))
            any_kept = keep_card.groupby(pid_series).any()
            # Patients that are eligible but currently have no kept rows
            need_force = [pid for pid in leak_patients if not bool(any_kept.get(pid, False))]
            if need_force:
                for pid in need_force:
                    idxs = out.index[eligible_rows & (pid_series == pid)]
                    if len(idxs) == 0:
                        continue
                    best_idx = forced_u.loc[idxs].idxmin()
                    keep_card.loc[best_idx] = True

        drop_card = ~keep_card

        for col in ["card_number", "CVV", "exp_date"]:
            if col in out.columns:
                out.loc[drop_card, col] = pd.NA

        if "card_brand" in out.columns:
            out.loc[drop_card, "card_brand"] = pd.NA

            # Optional: brand can be missing even when number is present
            if p_card_brand_null_when_number_present_rows > 0:
                brand_null = _select_exact_mask(
                    keys[keep_card],
                    seed,
                    "null_card_brand_when_number_present_row",
                    p_card_brand_null_when_number_present_rows,
                )
                out.loc[brand_null[brand_null].index, "card_brand"] = pd.NA

        # Enforce/validate card rules
        num_null = out["card_number"].isna()
        for col in ["CVV", "exp_date"]:
            if col in out.columns:
                out.loc[num_null, col] = pd.NA

        if "CVV" in out.columns and "exp_date" in out.columns:
            bad = out["card_number"].notna() & (out["CVV"].isna() | out["exp_date"].isna())
            if bool(bad.any()):
                raise ValueError("Card rule violated: card_number present but CVV and/or exp_date missing.")

    # -------------------------
    # Demographics: null per row, independent of patient_id
    # -------------------------
    if "name" in out.columns:
        null_name = _select_exact_mask(keys, seed, "null_name_row", p_name_null_rows)
        out.loc[null_name, "name"] = pd.NA

    if "date_of_birth" in out.columns:
        null_dob = _select_exact_mask(keys, seed, "null_dob_row", p_dob_null_rows)
        out.loc[null_dob, "date_of_birth"] = pd.NA

    addr_cols = [c for c in ["address", "city", "state", "zipcode"] if c in out.columns]
    if addr_cols:
        null_addr = _select_exact_mask(keys, seed, "null_address_row", p_address_null_rows)
        out.loc[null_addr, addr_cols] = pd.NA

    # -------------------------
    # SSN: null per row, independent of patient_id
    # -------------------------
    if "SSN_full" in out.columns:
        null_full = _select_exact_mask(keys, seed, "null_ssn_full_row", p_ssn_full_null_rows)
        out.loc[null_full, "SSN_full"] = pd.NA

    if "SSN_last4" in out.columns:
        null_l4 = _select_exact_mask(keys, seed, "null_ssn_last4_row", p_ssn_last4_null_rows)
        out.loc[null_l4, "SSN_last4"] = pd.NA

        # If SSN_full present, force SSN_last4 present (restore from original)
        if "SSN_full" in out.columns and ssn_last4_orig is not None:
            need_restore = out["SSN_full"].notna() & out["SSN_last4"].isna()
            if bool(need_restore.any()):
                out.loc[need_restore, "SSN_last4"] = ssn_last4_orig.loc[need_restore]

    return out


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--claims_export", required=True, help="Path to claims_export.csv")
    ap.add_argument("--seed", type=int, default=42, help="Deterministic seed")
    ap.add_argument("--p_ssn_full", type=float, default=0.20)
    ap.add_argument("--p_ssn_last4_blank", type=float, default=0.05)
    ap.add_argument("--p_card_patient_leak", type=float, default=0.43)
    ap.add_argument("--p_card_row_null_given_eligible", type=float, default=-1.0)
    ap.add_argument("--p_name_null_rows", type=float, default=-1.0)
    ap.add_argument("--p_dob_null_rows", type=float, default=-1.0)
    ap.add_argument("--p_address_null_rows", type=float, default=-1.0)
    ap.add_argument("--p_ssn_full_null_rows", type=float, default=-1.0)
    ap.add_argument("--p_ssn_last4_null_rows", type=float, default=-1.0)
    ap.add_argument("--p_card_brand_null_when_number_present_rows", type=float, default=0.0)

    # Single output file (enriched copy of the input)
    ap.add_argument("--out_enriched", default="claims_export_enriched.csv")

    # Deprecated args kept for compatibility (ignored; no extra CSVs will be written)
    ap.add_argument("--out_patient_sensitive", default="patient_sensitive.csv", help=argparse.SUPPRESS)
    ap.add_argument("--out_billing_payment", default="billing_payment.csv", help=argparse.SUPPRESS)

    args = ap.parse_args()

    # Auto-pick per-row null rates when set to -1 (deterministic by seed)
    if args.p_card_row_null_given_eligible < 0:
        args.p_card_row_null_given_eligible = auto_rate(args.seed, "p_card_row_null_given_eligible")
    if args.p_name_null_rows < 0:
        args.p_name_null_rows = auto_rate(args.seed, "p_name_null_rows", lo=0.10, hi=0.30)
    if args.p_dob_null_rows < 0:
        args.p_dob_null_rows = auto_rate(args.seed, "p_dob_null_rows", lo=0.15, hi=0.40)
    if args.p_address_null_rows < 0:
        args.p_address_null_rows = auto_rate(args.seed, "p_address_null_rows", lo=0.20, hi=0.45)
    if args.p_ssn_full_null_rows < 0:
        args.p_ssn_full_null_rows = auto_rate(args.seed, "p_ssn_full_null_rows")
    if args.p_ssn_last4_null_rows < 0:
        args.p_ssn_last4_null_rows = auto_rate(args.seed, "p_ssn_last4_null_rows")


    in_path = Path(args.claims_export)
    if not in_path.exists():
        raise FileNotFoundError(f"Input not found: {in_path}")

    claims = pd.read_csv(in_path)

    for col in ["billing_id", "patient_id"]:
        if col not in claims.columns:
            raise ValueError(f"Missing required column '{col}'. Found: {list(claims.columns)}")

    patient_sensitive = build_patient_sensitive(
        claims=claims,
        seed=args.seed,
        p_ssn_full=args.p_ssn_full,
        p_ssn_last4_blank=args.p_ssn_last4_blank,
    )
    billing_payment = build_billing_payment(claims=claims, seed=args.seed)

    # Merge generated columns back into the original claims rows
    claims_out = claims.copy()
    claims_out["patient_id"] = claims_out["patient_id"].astype("string")
    claims_out["billing_id"] = claims_out["billing_id"].astype("string")

    patient_sensitive["patient_id"] = patient_sensitive["patient_id"].astype("string")
    billing_payment["billing_id"] = billing_payment["billing_id"].astype("string")

    enriched = claims_out.merge(
        patient_sensitive,
        on="patient_id",
        how="left",
        validate="m:1",  # many claim rows per one patient row
    ).merge(
        billing_payment,
        on="billing_id",
        how="left",
        validate="m:1",  # many claim rows per one billing row
    )

    enriched = apply_row_level_leak_nulls(
        enriched,
        seed=args.seed,
        p_card_patient_leak=args.p_card_patient_leak,
        p_card_row_null_given_eligible=args.p_card_row_null_given_eligible,
        p_name_null_rows=args.p_name_null_rows,
        p_dob_null_rows=args.p_dob_null_rows,
        p_address_null_rows=args.p_address_null_rows,
        p_ssn_full_null_rows=args.p_ssn_full_null_rows,
        p_ssn_last4_null_rows=args.p_ssn_last4_null_rows,
        p_card_brand_null_when_number_present_rows=args.p_card_brand_null_when_number_present_rows,
    )

    enriched.to_csv(args.out_enriched, index=False)

    # QA
    pct_full = enriched["SSN_full"].notna().mean() * 100
    pct_blank = enriched["SSN_last4"].isna().mean() * 100
    pct_card = enriched["card_number"].notna().mean() * 100 if "card_number" in enriched.columns else float("nan")
    print(
        f"Wrote {args.out_enriched}: rows={len(enriched)} | SSN_full%={pct_full:.2f} | SSN_last4_blank%={pct_blank:.2f} | card_number%={pct_card:.2f}"
    )


if __name__ == "__main__":
    main()
```

## SQL Create Table Code
Code used to create the table in SQL before data import
```
CREATE TABLE public.claims (
  billing_id                   INTEGER PRIMARY KEY,
  visit_id                     INTEGER NOT NULL,
  patient_id                   INTEGER NOT NULL,

  billing_date                 DATE NOT NULL,
  insurance_coverage_amount     NUMERIC NOT NULL,
  patient_responsibility_amount NUMERIC NOT NULL,
  payment_plan                 VARCHAR(30),
  expected_payment_date        DATE,
  actual_payment_date          DATE,
  payment_status               VARCHAR(30),
  total_charge                 NUMERIC NOT NULL,

  name                         VARCHAR(150),
  date_of_birth                DATE,
  address                      VARCHAR(250),
  city                         VARCHAR(100),
  state                        CHAR(2),
  zipcode                      CHAR(5),

  insurance_provider           VARCHAR(150),
  insurance_policy_number      VARCHAR(60),
  phone_number                 VARCHAR(30),

  ssn_full                     CHAR(11),   -- 123-45-6789
  ssn_last4                    CHAR(11),   -- ***-**-6789

  card_brand                   VARCHAR(20),
  card_number                  CHAR(19),   -- ####-####-####-####

  cvv                          CHAR(3),    -- 3 digits
  exp_date                     CHAR(7)     -- MM/YYYY
);
```

