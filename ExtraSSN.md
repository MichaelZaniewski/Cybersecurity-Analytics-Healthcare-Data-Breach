# Python Additional PII Generation

```
#!/usr/bin/env python3
"""
Input: claims_export.csv (expects columns: billing_id, patient_id)

Output:
  - claims_export_enriched.csv: original columns + SSN_full, SSN_last4, card_brand, card_number, CVV, exp_date

Rules enforced:
  - One patient_id => one stable SSN_full profile (repeats consistent).
  - One patient_id => one stable card_number + CVV + exp_date profile (repeats consistent).
  - SSN_full present for 20% of unique patients.
  - SSN_last4 blank for 5% of unique patients (but never blank when SSN_full is present).
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
    """SSN-like 3-2-4 format."""
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
    # Partial breach: 43% of UNIQUE patients have card_number present.
    p_card_leak = 0.43

    # card_brand is allowed to be null even when card_number exists.
    # This sets brand to null for 15% of leaked-card patients (deterministic).
    p_brand_null_when_leaked = 0.15

    # Normalize to unique patient list
    pids = (
        pd.Series(patient_ids)
        .dropna()
        .astype(str)
        .drop_duplicates()
        .tolist()
    )
    n = len(pids)
    if n == 0:
        return pd.DataFrame(columns=["patient_id", "card_brand", "card_number", "CVV", "exp_date"])

    # Deterministically pick an exact count of leaked-card patients
    n_leak = int(round(p_card_leak * n))
    ranked = sorted(pids, key=lambda pid: stable_u01(pid, seed, "pick_card_leak"))
    leaked_ids = set(ranked[:n_leak])

    # Deterministically pick leaked patients where brand is missing
    n_brand_null = int(round(p_brand_null_when_leaked * n_leak))
    leaked_ranked_for_brand = sorted(leaked_ids, key=lambda pid: stable_u01(pid, seed, "pick_brand_null"))
    brand_null_ids = set(leaked_ranked_for_brand[:n_brand_null])

    today = date.today()
    rows = []

    for pid in pids:
        if pid in leaked_ids:
            rng = stable_rng(pid, seed, "card")

            # If card_number is present, CVV and exp_date must be present
            card_number = random_16_with_dashes_invalid(rng)
            CVV = random_three_digits_no_leading_zeros(rng)

            month = stable_int(pid, seed, "exp_month", 12) + 1
            years_ahead = stable_int(pid, seed, "exp_years_ahead", 5) + 1
            yy = (today.year + years_ahead) % 100
            exp_date = f"{month:02d}/{yy:02d}"

            card_brand = pd.NA if pid in brand_null_ids else pick_brand(stable_u01(pid, seed, "brand"))
        else:
            # Not breached for card info
            card_brand = pd.NA
            card_number = pd.NA
            CVV = pd.NA
            exp_date = pd.NA

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


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--claims_export", required=True, help="Path to claims_export.csv")
    ap.add_argument("--seed", type=int, default=42, help="Deterministic seed")
    ap.add_argument("--p_ssn_full", type=float, default=0.20)
    ap.add_argument("--p_ssn_last4_blank", type=float, default=0.05)

    # Single output file (enriched copy of the input)
    ap.add_argument("--out_enriched", default="claims_export_enriched.csv")

    # Deprecated args kept for compatibility (ignored; no extra CSVs will be written)
    ap.add_argument("--out_patient_sensitive", default="patient_sensitive.csv", help=argparse.SUPPRESS)
    ap.add_argument("--out_billing_payment", default="billing_payment.csv", help=argparse.SUPPRESS)

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

    enriched.to_csv(args.out_enriched, index=False)

    # QA
    pct_full = enriched["SSN_full"].notna().mean() * 100
    pct_blank = enriched["SSN_last4"].isna().mean() * 100
    print(
        f"Wrote {args.out_enriched}: rows={len(enriched)} | SSN_full%={pct_full:.2f} | SSN_last4_blank%={pct_blank:.2f}"
    )


if __name__ == "__main__":
    main()
```
