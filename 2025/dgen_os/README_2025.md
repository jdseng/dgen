# dGen 2025 Fixes and Improvements

This document consolidates all fixes and improvements made to the dGen codebase for 2025 runs, removing machine-specific paths for broader applicability.

---

## Fix 1: NumPy 2.0 Compatibility

- **File:** `dgen_os/python/agent_mutation/elec.py`
- **Lines:** 853-854
- **Change:** Replace `np.in1d` with `np.isin`
- **Reason:** `numpy.in1d` was deprecated in NumPy 1.24 and removed in NumPy 2.0
- **Impact:** Drop-in replacement with identical behavior

```python
# Before
np.in1d(county_ids, ba_county_mapping['county_id'])

# After  
np.isin(county_ids, ba_county_mapping['county_id'])
```

---

## Fix 2: Input Sheet Markets Field

- **File:** Input scenario sheet
- **Field:** `markets`
- **Change:** Use `All` instead of `Residential + Commercial`
- **Reason:** Database view only recognizes `All`, `Only Residential`, `Only Commercial`, `Only Industrial`

---

## Fix 3: Agent ID Preservation in Combined Pickles

- **File:** Agent pickle combination script
- **Issue:** `agent_id` was lost during concatenation
- **Fix:** Reset index to preserve `agent_id`, assign unique sequential IDs
- **Impact:** Required for combined residential + commercial agent files

```python
# Reset index to preserve agent_id
df1 = df1.reset_index()
df2 = df2.reset_index()
# Concat with unique IDs
combined = pd.concat([df1, df2], ignore_index=True)
combined['agent_id'] = range(len(combined))
combined.set_index('agent_id', inplace=True)
```

---

## Fix 4: Load Growth Configuration

- **File:** Input scenario sheet
- **Field:** `load_growth`
- **Change:** Set to `User Defined` with appropriate CSV file
- **Reason:** Database view only supports AEO 2018/2019 scenarios
- **Solution:** Use CSV fallback for AEO25 data

---

## Fix 5: Input Data Updates

- **Files:** Various CSV files in `input_data/`
- **Updates:** AEO25, ATB24, Cambium24 data files
- **Impact:** Supports 2025-2036 modeling timeframe

---

## Fix 6: Load Growth Query Enhancement

- **File:** `dgen_os/python/dgen_model.py`
- **Issue:** Missing `stacked_sectors` and `county_id` in load growth query
- **Fix:** Add missing fields to SELECT statement
- **Impact:** Ensures proper load growth data mapping

```sql
-- Added to load growth query
, stacked_sectors
, county_id
```

---

## Fix 7: Missing Input Tables

- **Files:** Various input CSV files
- **Issue:** Missing required input tables for specific scenarios
- **Fix:** Add missing CSV files to `input_data/` directories
- **Impact:** Completes input data requirements

---

## Fix 8: Wholesale Price BA Mapping

- **Files:** 
  - `input_data/wholesale_electricity_prices/wholesale_price_projections_cambium24_*.csv`
  - `input_data/county_to_ba_mapping.csv`
- **Issue:** BA mismatches causing NaN values
- **Fixes:**
  - z122 → p122 rename in Cambium24 CSVs
  - p119 → p122 remap for 5 PA counties
- **Impact:** Resolves wholesale electricity price NaNs

---

## Fix 9: SAM Utility Rate Tier Structure

- **File:** `dgen_os/python/financial_functions.py`
- **Issue:** SAM utilityrate5 requires uniform tier max usage values
- **Fix:** Normalize energy tiers across TOU periods, set last tier to unlimited (1e9)
- **Impact:** Prevents SAM tier cross-period inconsistency errors

```python
# Normalize energy tier max usage
e_levels = np.tile(e_levels[0:1], (len(e_levels), 1))
e_levels[-1, -1] = 1e9  # Last tier unlimited
```

---

## Fix 10: Demand Tier Structure

- **File:** `dgen_os/python/financial_functions.py`
- **Issue:** SAM requires uniform demand tier max usage values
- **Fix:** Similar normalization for demand tiers
- **Impact:** Prevents demand tier inconsistency errors

---

## Fix 11: Energy Unit Mapping

- **File:** `dgen_os/python/financial_functions.py`
- **Issue:** Missing `'kWh/kVA': 1` mapping in `max_usage_dict`
- **Fix:** Add mapping for demand-normalized energy units
- **Reason:** SAM's `ur_ec_tou_mat` unit column only accepts 0-3, mapping to option 1 (kWh/kW)

```python
max_usage_dict = {
    'kWh/kW': 0,
    'kWh/kVA': 1,  # Added this mapping
    'kW/kW': 2,
    'kVA/kVA': 3
}
```

---

## Fix 12: SAM Cashloan System Capacity

- **File:** `dgen_os/python/financial_functions.py`
- **Lines:** ~227-228
- **Issue:** SAM cashloan requires positive `system_capacity`
- **Fix:** Set minimum positive value (0.001 kW) to prevent crashes

```python
# Before
loan.FinancialParameters.system_capacity = kw

# After
loan.FinancialParameters.system_capacity = max(kw, 0.001)
```

---

## Fix 13: Model Years Validation

- **File:** `dgen_os/python/settings.py`
- **Issue:** Validation required strict 2014 start year
- **Fix:** Allow start year at or after 2014
- **Impact:** More flexible year range configuration

```python
# Before
"Must begin with 2014"

# After
"Must begin at 2014 or later"
```

---

## Development and Deployment Notes

### VM Environment Considerations
- VM is the authoritative working copy on `seia` branch at `github.com/jdseng/dgen`
- Staging machine edits are uploaded to VM via `gcloud compute scp`, then committed and pushed from the VM
- Staging machine git state is not used for deployment
- Always verify VM file state before patching

### File Upload Best Practices
- Use `/tmp/` as staging area for uploads
- Verify target directories exist
- Use absolute paths for remote operations

### Database Schema Management
- Drop existing schemas before new runs to avoid conflicts
- Schema naming: `diffusion_results_YYYYMMDD_*_sheet`
- Use `-h 127.0.0.1` for TCP connections to avoid auth issues

### Monitoring and Logging
- Use tmux for long-running processes
- Implement background monitoring with `nohup`
- Check progress via database queries and log files

---

## Testing and Validation

Each fix should be tested with:
1. Small test runs before full deployment
2. Verification that original issues are resolved
3. No regression in existing functionality
4. Proper error handling and logging

---

## National Run Attempt (Sep 11-12, 2026)

### Configuration (Initial Attempt — Sep 11)
- **VM:** `instance-20260410-first`, `c3d-highmem-16` (16 vCPU, 128 GB RAM), `us-east4-a`
- **Input sheet:** `2026_input_sheet.xlsm` with `markets='All'`, `scenario_name='national_reference_110926'`
- **Config:** `local_cores=14`, `start_year=2020`
- **Agent file:** `agent_df_ref_res_and_com_national.pkl` (376,886 agents, res+com combined)
- **Model years:** 2020–2050, every 2 years (~16 years)

### Observations
- Run started at 19:44 UTC Sep 11, 2026
- 14 worker processes confirmed at ~93–97% CPU utilization
- Memory usage: ~122–134 GB (tight against 128 GB limit; COW-shared pages inflate RSS)
- After ~19.7 hours: still on first model year (2020), 60% of agents complete
- Estimated per-year runtime: ~32 hours (first year includes startup overhead)
- Estimated total runtime: ~500 hours (~21 days)
- Estimated cost at $0.98/hr: ~$490

### Optimizations Applied (Sep 14)
- **Tolerance coarsened** from 0.5 kW to 1.0 kW in `financial_functions.py` (VM-side `sed` patch)
- **VM resized** from `c3d-highmem-16` to `c2d-highmem-32` (32 vCPU, 256 GB RAM, AMD EPYC Milan)
- **`local_cores` updated** from 14 to 30
- **Quota constraint:** N2D_CPUS limited to 16 in us-east4; C2D_CPUS quota is 400
- **Disk constraint:** C4D machine types require `pd-extreme` (existing disk is `pd-balanced`)

### Conclusion
Full national run at current scale is feasible but expensive. Run was stopped after confirming it progresses correctly. Optimized configuration reduces estimated runtime from ~500 hrs / ~$490 to ~107–118 hrs / ~$210–$231 (with 9 model years through 2036). Future runs should consider:
- **Tier 1 optimizations** (reduce model year frequency, coarsen optimization tolerance): ~55-70% runtime reduction
- **Agent sampling** (Tier 3): 75-90% fewer agents, largest available speedup
- See `dgen_performance_optimization_brief.md` for full machine type comparison and quota notes

### Files Prepared
- `runs/reference_NTL_110926/2026_input_sheet.xlsm` — national input sheet with Fix 2 applied
- `runs/reference_NTL_110926/config.py` — config with `local_cores=14`, `start_year=2020`
- Both uploaded to VM and verified via MD5 hash

---

## Fix 14: Division-by-Zero Guard in NAEP Calculation

- **File:** `dgen_os/python/financial_functions.py`
- **Function:** `calc_system_size_and_performance`
- **Change:** Guard `naep` calculation against `system_kw = 0`
- **Reason:** When the optimizer returns `system_kw = 0`, `naep = annual_energy_production_kwh / system_kw` produces NaN, which propagates to `capacity_factor`. Runtime validation in `agents.py` raises `ValueError`, crashing the run.
- **Impact:** 2 commercial agents (county_id 2465, bin_id 491) in national run `reference_NTL_150926`
- **Run README ref:** Fix 9

```python
# Before
naep = annual_energy_production_kwh / system_kw

# After
if system_kw > 0:
    naep = annual_energy_production_kwh / system_kw
else:
    naep = 0
```

- **Also includes:** Tolerance coarsening from 0.5 kW to 1.0 kW (previously VM-side only, now committed to `seia` branch)
- **Commit:** `4724ee7` on `seia` branch
- **Date:** 2026-09-17

---

## Fix 15: OOM Prevention — Batched Result Concatenation + Memory Reclamation

- **Files:** `dgen_os/python/agents.py`, `dgen_os/python/dgen_model.py`
- **Problem:** Run 3 (`ntl-180926a`) was killed by Linux OOM killer at ~35 hrs into the run. Main process RSS reached 121.8 GB (year 2022), combined with 30 workers (~240 GB) exceeded 256 GB VM limit.
- **Root cause:** Three compounding factors:
  1. `apply_chunk_on_row` holds all 376,886 result Series simultaneously before `pd.concat`
  2. `run_with_runtime_tests` creates a full merge copy of the dataframe
  3. glibc malloc does not return freed memory to OS between model years
- **Fix A:** Batched concatenation in `apply_chunk_on_row` (5,000 results per batch) — `agents.py`
- **Fix B:** `gc.collect()` + `ctypes.CDLL('libc.so.6').malloc_trim(0)` between model years — `dgen_model.py`
- **Fix C (optional):** Reduce `local_cores` from 30 to 24 (saves ~48 GB worker memory)
- **Run README ref:** Fix 10
- **Status:** Applied — committed to `seia` branch (`74b3802`), pushed to GitHub. Run 4 (`ntl-200926a`) crashed at year 2024 (115.8 GB main RSS). See `runs/reference_NTL_150926/README.md` for full OOM troubleshooting report and time/cost matrix.
- **Date:** 2026-09-20

---

## Fix 16: OOM Prevention — Drop Expendable Array Columns Before Merge

- **Files:** `dgen_os/python/config.py`, `dgen_os/python/agents.py`
- **Problem:** Run 4 (`ntl-200926a`) was killed by OOM killer at ~48 hrs into the run (year 2024, 85% of `chunk_on_row`). Main process RSS reached 115.8 GB. Fix 15 addressed factors 1 and 3 of the root cause, but factor 2 (`run_with_runtime_tests` creates a full merge copy) was left unfixed.
- **Root cause:** `pd.merge(self.df, results_df, on='agent_id')` in `run_with_runtime_tests` duplicates the full 376,886-row DataFrame including large 8,760-element array columns (`consumption_hourly`, `solar_cf_profile`, `tariff_dict`, `deprec_sch`, `state_incentives`) and large result arrays (`cash_flow`, `batt_dispatch_profile`, `cbi`/`ibi`/`pbi`). At peak, `self.df` + `results_df` + merged result all coexist.
- **Fix:**
  - `config.py`: Added `EXPENDABLE_INPUT_COLUMNS` and `EXPENDABLE_RESULT_COLUMNS` lists. Updated `MISSING_COLUMN_EXCEPTIONS` to include expendable input columns so runtime tests don't flag them.
  - `agents.py`: In `run_with_runtime_tests`, after the function completes but before `pd.merge`, drop expendable columns from both `self.df` and `results_df`. Fixed `post_dtypes` and dtype check loop to handle intentionally-dropped columns.
- **Impact:** Reduces merge peak memory by ~65–70 GB (8,760-element arrays × 376,886 agents × 2 copies). Main process RSS peak estimated to drop from ~115 GB to ~45–50 GB.
- **Verification:** None of the dropped columns are used by any subsequent function (`calc_max_market_share`, `calculate_developable_customers_and_load`, `calc_diffusion_solar`, `estimate_total_generation`). All are explicitly dropped before DB write in `dgen_model.py`.
- **Run README ref:** Fix 11
- **Status:** Partially applied in `80feea7` / `8e2453c`. Run 5 (`ntl-230926a`) completed year 2020 but crashed at year 2022 with `KeyError: 'tariff_dict'`. Dropping base agent columns from `self.df` permanently removed `tariff_dict` needed in subsequent years. Resolved by Fix 17.
- **Date:** 2026-09-23

---

## Fix 17: Retain Base Agent Columns in self.df (Fix KeyError: 'tariff_dict')

- **Files:** `dgen_os/python/config.py`, `dgen_os/python/agents.py`
- **Problem:** Run 5 (`ntl-230926a`) completed year 2020 successfully (~17.5 hrs), but crashed immediately at year 2022 with `KeyError: 'tariff_dict'` in `financial_functions.calc_system_size_and_performance`.
- **Root cause:** Fix 16 dropped `EXPENDABLE_INPUT_COLUMNS` (including `tariff_dict`) from `self.df` before `pd.merge` in `run_with_runtime_tests`. Because `chunk_on_row` updates `self.df = results_df`, `tariff_dict` was permanently lost after year 2020. `tariff_dict` is an immutable base agent attribute loaded once at startup (`cols_base`) and used every model year by `calc_system_size_and_performance`. Furthermore, VM inspection showed `self.df` is only 860 MB (<0.4% VM memory) with `tariff_dict` accounting for 316 MB. The true memory spike is in `results_df` (specifically `batt_dispatch_profile` at 8,760 floats/row = ~26.4 GB per DataFrame copy).
- **Fix:**
  - `agents.py`: Removed `self.df.drop(config.EXPENDABLE_INPUT_COLUMNS, ...)` before `pd.merge`. Only drop `EXPENDABLE_RESULT_COLUMNS` from `results_df`. Restored standard `initial_columns` checks in runtime tests.
  - `config.py`: Removed `EXPENDABLE_INPUT_COLUMNS`, kept `EXPENDABLE_RESULT_COLUMNS`, and restored `MISSING_COLUMN_EXCEPTIONS = []`.
- **Impact:** `self.df` retains all base agent attributes across all model years. Memory optimization remains fully effective by stripping the 26+ GB array columns from `results_df` before the merge copy.
- **Run README ref:** Fix 12
- **Date:** 2026-09-24

---

## Version History

- **2025-04-25:** Initial fixes 1-5 (NumPy compatibility, input data)
- **2025-04-26:** Fixes 6-8 (load growth, BA mapping)
- **2025-04-30:** Fixes 9-12 (SAM utility rates, cashloan)
- **2025-05-02:** Fix 13 (model years validation)
- **2026-09-11:** National run attempt, config.py updated to `local_cores=14`
- **2026-09-14:** VM resized to c2d-highmem-32, tolerance coarsened to 1.0 kW, `local_cores=30`
- **2026-09-16:** `reference_NTL_150926` run with ATB25/AEO26 data — crashed on NaN naep (Fix 14)
- **2026-09-17:** Fix 14 applied (division-by-zero guard), committed to `seia` branch (`4724ee7`)
- **2026-09-18:** Run 3 OOM crash; `enable-linger` applied to prevent systemd process cleanup
- **2026-09-19:** Run 3 killed by OOM at year 2022 (121.8 GB main RSS)
- **2026-09-20:** Fix 15 applied (batched concat + malloc_trim), committed to `seia` (`74b3802`); Run 4 (`ntl-200926a`) started; OOM troubleshooting documented
- **2026-09-22:** Run 4 OOM crash at year 2024 (115.8 GB main RSS) — Fix 15 addressed factors 1 and 3 but not factor 2 (merge copy)
- **2026-09-23:** Fix 16 applied (drop expendable columns before merge), committed to `seia` (`80feea7`, `8e2453c`, `49e66de`); Run 5 (`ntl-230926a`) started
- **2026-09-24:** Run 5 crashed at year 2022 (`KeyError: 'tariff_dict'`). Fix 17 applied (retain `self.df` base columns, drop from `results_df` only)

---

## Contributing

When adding new fixes:
1. Document the issue, solution, and impact
2. Include code snippets showing before/after
3. Test thoroughly before deployment
4. Update this README with machine-agnostic paths
5. Consider backward compatibility implications
