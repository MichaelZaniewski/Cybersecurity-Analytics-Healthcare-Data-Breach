SSN_full replaced with 3-2-4
SSN_last4 replaced with xxx-xx-4
card_brand replaced with brand
card_number replaced with 16digit
CVV replaced with tripledigit
exp_date replaced with date


```
#!/usr/bin/env python3
"""
Input: claims_export.csv (expects columns: billing_id, patient_id)
Output:
  - patient_sensitive.csv: patient_id, 3-2-4, xxx-xx-4
  - billing_payment.csv:  billing_id, brand, 16digit, tripledigit, date

Rules enforced:
  - One patient_id => one stable 3-2-4 profile (repeats consistent).
  - One patient_id => one stable 16digit profile (repeats consistent across billing_id).
  - 3-2-4 present for 20% of unique patients.
  - xxx-xx-4 blank for 5% of unique patients (but never blank when 3-2-4 is present).
"""

from __future__ import annotations

import argparse
import hashlib
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


def pick_brand(u: float) -> str:
    # Visa 55%, Mastercard 30%, Amex 10%, Discover 5%
    if u < 0.55:
        return "visa"
    if u < 0.85:
        return "mastercard"
    if u < 0.95:
        return "amex"
    return "discover"


def build_patient_sensitive(
    claims: pd.DataFrame,
    seed: int,
    p_3-2-4: float = 0.20,
    p_xxx-xx-4_blank: float = 0.05,
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

    # Stable exact counts
    n_full = int(round(p_3-2-4 * n))
    n_blank = int(round(p_xxx-xx-4_blank * n))

    patients["_u_full"] = patients["patient_id"].map(lambda pid: stable_u01(pid, seed, "pick_3-2-4"))
    patients["_u_blank"] = patients["patient_id"].map(lambda pid: stable_u01(pid, seed, "pick_xxx-xx-4_blank"))

    # Select 3-2-4 patients (20%)
    full_ids = set(patients.sort_values("_u_full").head(n_full)["patient_id"].tolist())

    # Select xxx-xx-4 blanks (5%) from NON-full patients
    non_full = patients[~patients["patient_id"].isin(full_ids)].copy()
    blank_ids = set(non_full.sort_values("_u_blank").head(n_blank)["patient_id"].tolist())

    def last4(pid: str) -> str:
        return str(stable_int(pid, seed, "xxx-xx-4", 10000)).zfill(4)

    ssn_last4_out = []
    ssn_full_out = []

    for pid in patients["patient_id"].tolist():
        l4 = last4(pid)

        # xxx-xx-4 format requested: ***-**-1234
        masked_last4 = f"***-**-{l4}"

        if pid in full_ids:
            3-2-4_out.append(f"000-00-{l4}")
            xxx-xx-4_out.append(masked_last4)
        else:
            3-2-4_out.append(pd.NA)
            xxx-xx-4_out.append(pd.NA if pid in blank_ids else masked_last4)

    return pd.DataFrame(
        {
            "patient_id": patients["patient_id"],
            "3-2-4": 3-2-4_out,
            "xxx-xx-4": xxx-xx-4_out,
        }
)


def build_patient_card_profile(patient_ids: pd.Series, seed: int) -> pd.DataFrame:
    today = date.today()
    rows = []

    for pid in patient_ids.astype(str).tolist():

        brand = pick_brand(stable_u01(pid, seed, "brand"))
        l4 = str(stable_int(pid, seed, "brand_last4", 10000)).zfill(4)

        """16 digits, grouped 4-4-4-4 with hyphens. Example: 4821-0934-5500-7716"""
           n = random.randint(0, 9999_9999_9999_9999)
           s = f"{n:016d}"
            return "-".join(s[i:i+4] for i in range(0, 16, 4))
"""A triple-digit number in xxx format (100–999). Example: 482"""
  tripledigit =  """A triple-digit number in xxx format (100–999). Example: 482"""
    return random.randint(100, 999)

        month = stable_int(pid, seed, "exp_month", 12) + 1
        years_ahead = stable_int(pid, seed, "exp_years_ahead", 5) + 1
        yy = (today.year + years_ahead) % 100
        exp_date = f"{month:02d}/{yy:02d}"

        rows.append(
            {
                "patient_id": pid,
                "brand": brand,
                "16digit": 16digit,
                "tripledigit": tripledigit,
                "exp_date": exp_date,
            }
        )

    return pd.DataFrame(rows)


def build_billing_payment(claims: pd.DataFrame, seed: int) -> pd.DataFrame:
    base = claims[["billing_id", "patient_id"]].dropna(subset=["billing_id", "patient_id"]).copy()
    base["billing_id"] = base["billing_id"].astype(str)
    base["patient_id"] = base["patient_id"].astype(str)

    # Enforce billing_id -> patient_id is 1:1
    bad = base.groupby("billing_id")["patient_id"].nunique()
    bad = bad[bad > 1]
    if len(bad) > 0:
        raise ValueError(
            f"{len(bad)} billing_id values map to multiple patient_id values. Fix the input join first."
        )

    billing_unique = base.drop_duplicates(subset=["billing_id"], keep="first").reset_index(drop=True)

    # Build per-patient stable card profile and attach to each billing_id
    patient_profile = build_patient_card_profile(billing_unique["patient_id"].drop_duplicates(), seed=seed)
    out = billing_unique.merge(patient_profile, on="patient_id", how="left")

    # Output only requested columns
    return out[["billing_id", "brand", "16digit", "tripledigit", "exp_date"]]


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--claims_export", required=True, help="Path to claims_export.csv")
    ap.add_argument("--seed", type=int, default=42, help="Deterministic seed")
    ap.add_argument("--p_3-2-4", type=float, default=0.20)
    ap.add_argument("--p_xxx-xx-4_blank", type=float, default=0.05)
    ap.add_argument("--out_patient_sensitive", default="patient_sensitive.csv")
    ap.add_argument("--out_billing_payment", default="billing_payment.csv")
    args = ap.parse_args()

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
        p_3-2-4=args.p_3-2-4,
        p_xxx-xx-4_blank=args.p_xxx-xx-4_blank,
    )
    billing_payment = build_billing_payment(claims=claims, seed=args.seed)

    patient_sensitive.to_csv(args.out_patient_sensitive, index=False)
    billing_payment.to_csv(args.out_billing_payment, index=False)

    # QA
    pct_full = patient_sensitive["3-2-4"].notna().mean() * 100
    pct_blank = patient_sensitive["xxx-xx-4"].isna().mean() * 100
    print(f"Wrote {args.out_patient_sensitive}: rows={len(patient_sensitive)} | 3-2-4%={pct_full:.2f} | xxx-xx-4_blank%={pct_blank:.2f}")
    print(f"Wrote {args.out_billing_payment}: rows={len(billing_payment)}")


if __name__ == "__main__":
    main()
```


```
import random

def random_16_with_dashes() -> str:
    """16 digits, grouped 4-4-4-4 with hyphens. Example: 4821-0934-5500-7716"""
    n = random.randint(0, 9999_9999_9999_9999)
    s = f"{n:016d}"
    return "-".join(s[i:i+4] for i in range(0, 16, 4))

def random_3_2_4() -> str:
    """Digits formatted 3-2-4 with hyphens. Example: 482-09-7716"""
    a = random.randint(0, 999)
    b = random.randint(0, 99)
    c = random.randint(0, 9999)
    return f"{a:03d}-{b:02d}-{c:04d}"

def random_three_digits_no_leading_zeros() -> int:
    """A triple-digit number in xxx format (100–999). Example: 482"""
    return random.randint(100, 999)

# Example usage
print(random_16_with_dashes())
print(random_3_2_4())
print(random_three_digits_no_leading_zeros())
```
