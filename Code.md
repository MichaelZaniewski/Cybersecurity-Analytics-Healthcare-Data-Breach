I generated the dataset for a regional hospital with 59K patients with 485 distinct patient zipcodes in the dataset.

python healthcare_dataset_generator.py   --patients 59000   --today 2026-02-24   --zip-target 485   --zip-pool-file ".\us_zip_pool_10k_with_state.csv"   --outdir ".\Dataset"
