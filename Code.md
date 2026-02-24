1) Tweaked the healthcare dataset generator to focus on ONE hospital instead of many nationwide. Assigned patient zipcodes according to distance from hospital, weighted for locality. 
2)
3)

 Generated dataset for a regional hospital with 59K patients with 485 distinct patient zipcodes over the past 3 years.

python healthcare_dataset_generator.py   --patients 59000   --today 2026-02-24   --zip-target 485   --zip-pool-file ".\us_zip_pool_10k_with_state.csv"   --outdir ".\Dataset"

2) 
