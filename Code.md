1) Tweaked the healthcare dataset generator functionality to include a focus on ONE hospital instead of many nationwide. Assigned patient zipcodes according to distance from hospital, weighted for locality. 

Generated patient for a regional hospital with 50K unique patients weighted for patients near the hypothetical NYC hospital location over the past 3 years.
 
```
python healthcare_dataset_generator_local_skew.py
--patients 50000
--today 2026-02-24
--zip-pool-file ".\us_zip_pool_10k_with_state.csv"
--outdir ".\Dataset"
--single-hospital
--hospital-state NY
--hospital-zipcode 10036
--local-share 0.83
--local-top-n 400
--local-decay 200
--outstate-neighbor-share 0.80
```

2) 


run with locality - One hospital, local patient zipcodes
```
python healthcare_dataset_generator_local_skew.py --patients 65000 --today 2026-02-24 --zip-pool-file ".\us_zip_pool_10k_with_state.csv" --outdir ".\Dataset" --single-hospital --hospital-state NY --hospital-zipcode 10036 --local-share 0.83 --local-top-n 400 --local-decay 200 --outstate-neighbor-share 0.00
```



run without locality - Multiple hospitals, nationwide patient zipcodes

```
python healthcare_dataset_generator_local_skew.py --patients 50325 --today 2026-02-24 --zip-target 5800 --zip-pool-file ".\us_zip_pool_10k_with_state.csv" --outdir ".\Dataset"
```




- patients:
Number of unique patients generated (drives rows in patients.csv and the volume of visits.csv and billing.csv downstream).

- seed:
Random seed for reproducibility. Same inputs + same seed = same dataset.

- today:
Anchor date for the generator’s timeline and any age calculations. Use the end date of your 3-year window.

- outdir:
Folder where the CSV outputs are written.

- zip-pool-file:
Path to your ZIP pool CSV.
Required column: zipcode.
Optional but preferred: state, city. If city is missing, the script uses a prefix-based city mapping, which is less accurate.

- zip-target:
Intended target for how many distinct ZIP codes you want represented (bounded by the pool size).
In single-hospital local skew mode, the distinct ZIP count is driven mainly by --local-top-n (in-state) plus whatever out-of-state ZIPs get sampled.

- hospital-state:
Turns on single-hospital local skew mode anchored to this state (example NY).
Patients are split into in-state vs out-of-state using --local-share.

- hospital-zipcode:
Sets the anchor hospital ZIP (must exist in the ZIP pool and match --hospital-state).
This ZIP is the reference point for “local” distance weighting.

- single-hospital:
Forces all visits to a single hospital (one organization).
If you also set --hospital-state, you are implicitly in “single hospital + local skew” mode anyway.

- hospital-name:
Optional label for the single hospital used when --single-hospital is active.

- local-share:
Fraction of patients assigned to the local area (same state as the hospital).
Example: 0.83 means 83% in-state, 17% out-of-state.

- local-top-n:
Size of the in-state “catchment pool”: the script finds the closest N ZIPs in the hospital state to the hospital ZIP and samples from that set.
Smaller value = fewer distinct in-state ZIPs and a tighter service area.
Typical range: 200 to 800.

- local-decay:
Controls how sharply patient ZIP probability drops as ZIP distance from the hospital increases.
Smaller value = tighter clustering near the hospital ZIP.
Larger value = flatter distribution across the local-top-n pool.
Typical range: 100 to 600.

- outstate-neighbor-share:
When assigning out-of-state patients, this is the fraction pulled from “neighboring states” (more realistic) versus anywhere else.
Higher value = out-of-state patients mostly from nearby states (NJ, CT, PA, MA, etc).


EXAMPLES
Smaller community hospital feel
--patients 20000
--local-share 0.90
--local-top-n 150 to 250
--local-decay 120 to 200

Large regional / academic feel
--patients 50000
--local-share 0.80 to 0.85
--local-top-n 400 to 800
--local-decay 200 to 450
