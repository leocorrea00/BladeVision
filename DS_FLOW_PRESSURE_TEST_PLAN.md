# DS Flow Rate Pressure Test Plan (0-2000 gpm)

## Test Objective
Verify that pressure at TD (Total Depth) increases monotonically with DS flow rate across the full operating range (0-2000 gpm) when Booster MW equals DS MW, and that physics is respected without artificial cutoffs.

---

## Test Setup

### Initial Conditions
1. **Open**: `Losses V26.txt` in your p5.js environment
2. **Set Booster MW = DS MW**: Use the same mud weight for both pumps
   - Recommended: `MW_DS = MW_BO = 9.5 ppg`
3. **Set Booster Flow = 0 gpm**: Turn off booster pump
   - `Q_BO = 0 gpm`
4. **Disable P-Only Corrections**: Turn off compressibility effect
   - Click "P-only Corrections: OFF" button
5. **Disable MPD/SBP**: Ensure Surface Back Pressure = 0
   - `SBP = 0 psi`
6. **Disable Loss Zones**: Turn off all loss zones
   - Losses toggle: OFF
7. **Note your rheology**: Record the current rheology model and parameters
   - Check console for: `ACTIVE_RHEO_MODEL`, `TAUY_Pa`, `K_PaSn`, `N_dim`

---

## Test Procedure

### Phase 1: Zero Flow Test (Critical)
**Expected**: When pumps are OFF, ECD at TD should equal MW (no friction).

1. **Set Q_DS = 0 gpm, Q_BO = 0 gpm**
2. **Wait 5 seconds** for simulation to stabilize
3. **Open browser console** (F12)
4. **Record** the following values from console output:
   ```
   === TD PRESSURE BREAKDOWN ===
   Q_DS: 0 gpm | Q_BO: 0 gpm
   MW_DS: 9.50 ppg | MW_BO: 9.50 ppg
   Rheology: HB | τ_y: X.XX Pa | K: X.XXXX | n: X.XXX
   Hydrostatic: XXXX.X psi
   Friction: X.X psi          ← Should be ~0 psi
   BHP_TD: XXXX.X psi
   ECD_TD: X.XXX ppg          ← Should equal MW_DS
   ```

**Pass Criteria**:
- ✅ Friction < 0.5 psi (ideally 0.0 psi)
- ✅ ECD_TD ≈ MW_DS (within 0.01 ppg)
- ❌ If friction > 0.5 psi → **PHYSICS VIOLATION**
- ❌ If ECD_TD ≠ MW_DS → **PHYSICS VIOLATION**

---

### Phase 2: Low Flow Test (500 gpm)
**Expected**: Friction should appear, BHP increases.

1. **Set Q_DS = 500 gpm** (keep Q_BO = 0)
2. **Wait 10 seconds** for flow to stabilize
3. **Record from console**:
   ```
   BHP_TD: XXXX.X psi
   ECD_TD: X.XXX ppg
   Friction: XXX.X psi        ← Should be > 0
   ```
4. **Check friction debug logs** (printed every 3 seconds):
   ```
   ========== FRICTION DEBUG [XXX REGIME] ==========
   SECTION:   RISER/CASING/OPENHOLE
   FLOW:      Q=X.XXXXX m³/s = XXX.X gpm
   Reynolds:  XXXX (LAM<2000, TURB>2300)
   Yield:     τ_y=X.XX Pa, boost=XX.X Pa/m ✓
              dp_before=XXX.X, dp_after=XXX.X, actual_boost=XX.X
   RESULT:    XXX.X Pa/m
   ```

**Pass Criteria**:
- ✅ Friction > 0 psi
- ✅ BHP_TD > BHP_TD at Q=0
- ✅ Yield boost shows `✓` (matching section)
- ✅ `dp_after > dp_before` if τ_y > 0

---

### Phase 3: Monotonic Increase Test (0 → 2000 gpm)
**Expected**: BHP should NEVER decrease as flow increases.

Test the following flow rates in sequence:
```
Q_DS (gpm):  0, 250, 500, 750, 1000, 1250, 1500, 1750, 2000
```

For each flow rate:
1. **Set Q_DS** to test value
2. **Wait 10-15 seconds** for stabilization
3. **Record**: Q_DS, BHP_TD, ECD_TD, Friction
4. **Check**: BHP[i] > BHP[i-1] for each increase

**Data Table**:
```
Q_DS (gpm) | BHP_TD (psi) | ECD_TD (ppg) | Friction (psi) | ΔBHP (psi) | Status
-----------|--------------|--------------|----------------|------------|--------
     0     |              |              |                |     -      |
   250     |              |              |                |            |
   500     |              |              |                |            |
   750     |              |              |                |            |
  1000     |              |              |                |            |
  1250     |              |              |                |            |
  1500     |              |              |                |            |
  1750     |              |              |                |            |
  2000     |              |              |                |            |
```

**Pass Criteria**:
- ✅ ΔBHP > 0 for every row (BHP increases)
- ✅ Friction increases with flow
- ❌ If ΔBHP < 0 → **PHYSICS VIOLATION**
- ❌ If BHP console shows error: `❌ PHYSICS VIOLATION: BHP DECREASED` → **CRITICAL FAILURE**

---

### Phase 4: Friction Scaling Test
**Expected**: In turbulent regime, friction should scale approximately as Q².

Calculate friction ratios:
- F(1000) / F(500) ≈ 4 (doubling flow → 4× friction)
- F(2000) / F(1000) ≈ 4

**Pass Criteria**:
- ✅ Ratio between 3-5 (acceptable for non-Newtonian fluids)
- ⚠️  Ratio < 2 or > 6 → **WARNING: Check rheology**

---

### Phase 5: Rheology Verification
**Expected**: User-input rheology parameters are being used.

1. **Record your input readings** (R600, R300, R200, R100, R6, R3)
2. **Check console** for fitted parameters:
   ```
   [RHEO_FIT_DEBUG] Published to globals: K_PaSn=X.XXXX, N_dim=X.XXX, TAUY_Pa=X.XX
   ```
3. **Verify** these match your expected values from fann data

**Pass Criteria**:
- ✅ `window.TAUY_Pa`, `window.K_PaSn`, `window.N_dim` are non-zero
- ✅ Values match your input data
- ❌ If all zeros or default values → **Rheology NOT applied**

---

## Known Issues to Check

### Issue 1: Yield Stress Not Applied (from previous conversation)
**Symptom**: Friction too low at high flow rates
**Check**: Look for yield boost in friction logs:
```
Yield:     τ_y=4.92 Pa, boost=67.8 Pa/m ✓
           dp_before=150.7, dp_after=218.5, actual_boost=67.8
```
- ✅ `dp_after > dp_before` by amount = `boost`
- ✅ Section match shows `✓` (not `❌`)
- ❌ If `dp_after = dp_before` → yield stress not being added

### Issue 2: Section Flow Mismatch
**Symptom**: Yield boost shows `❌(Dh=X.XXX)` instead of `✓`
**Cause**: `window.__lastYieldBoost` from wrong section
**Check**: Dh (hydraulic diameter) should match current section:
- RISER: Dh ≈ 0.35 m (21" ID - 5" OD)
- CASING: Dh ≈ 0.09 m (9.625" ID - 5" OD)
- OPENHOLE: Dh ≈ 0.09 m (8.5" OD - 5" OD)

### Issue 3: Flow Lag
**Symptom**: Friction changes one frame after Q changes
**Fixed**: V26 already has this fix (line 720: calls `recomputeAnnulusSectionFlowsForLosses()`)

---

## Expected Results Summary

### At Q = 0 gpm (Pumps OFF)
```
Friction:  0.0 psi
BHP_TD:    ≈ Hydrostatic only
ECD_TD:    = MW_DS (9.5 ppg)
```

### At Q = 500 gpm (Low Flow, likely Turbulent)
```
Friction:  ~50-150 psi (depends on rheology)
BHP_TD:    Hydrostatic + Friction
ECD_TD:    > MW_DS (friction boost)
Reynolds:  > 2300 (turbulent)
```

### At Q = 2000 gpm (High Flow, Turbulent)
```
Friction:  ~500-1500 psi (depends on rheology)
BHP_TD:    Much higher than Q=0
ECD_TD:    Significantly > MW_DS
Reynolds:  >> 2300 (fully turbulent)
```

### Friction Trend
```
Friction vs Flow should look like:
  ^
  |                                    *  (2000 gpm)
  |                           *  (1500 gpm)
F |                   *  (1000 gpm)
r |           *  (500 gpm)
i |     *  (250 gpm)
c |  *  (0 gpm - zero)
  |________________________
         Flow Rate (gpm) →

- Smooth monotonic increase
- NO sudden drops or jumps
- Approximately parabolic (Q²) in turbulent regime
```

---

## Physics Checks (Built into V26)

The code automatically checks and logs warnings:

### ⚠️  Warning Messages:
```javascript
⚠️  PHYSICS VIOLATION: Friction = XX.X psi at ZERO flow!
⚠️  PHYSICS VIOLATION: ECD = X.XXX ppg but expected X.XXX ppg at zero flow
```

### ❌ Error Messages:
```javascript
❌ PHYSICS VIOLATION: BHP DECREASED from XXXX.X to XXXX.X psi
   when Q increased from XXX to XXX gpm
```

**If you see any of these → STOP and report the issue with full console logs.**

---

## Troubleshooting

### Problem: Friction = 0 at ALL flow rates
**Cause**: Rheology not loaded or Q=0 everywhere
**Fix**:
1. Check `window.Q_DS` in console
2. Check `window.__qAnnAbove_m3s` and `window.__qAnnBelow_m3s`
3. Click "Fit Rheology" button to reload rheology
4. Refresh page and try again

### Problem: ECD ≠ MW at Q=0
**Cause**: Residual friction or compressibility effect ON
**Fix**:
1. Turn OFF "P-only Corrections"
2. Set Q_DS = 0, Q_BO = 0
3. Wait 30 seconds for all flows to settle
4. Check `window.__qAnnAbove_m3s` in console (should be 0)

### Problem: BHP decreases when Q increases
**Critical**: This is a physics violation
**Actions**:
1. **Take screenshot** of console logs
2. **Note the Q_DS values** where BHP decreased
3. **Check friction logs** for that flow rate
4. **Check if yield boost** shows `❌` instead of `✓`
5. **Report** with full details

### Problem: Yield boost shows ❌ (wrong section)
**Cause**: `window.__lastYieldBoost` contamination
**Check**:
- Compare `Dh` in yield log vs section
- If mismatch, yield stress from different section being logged
- **This may be cosmetic** (log issue) or **real** (calculation issue)
- To verify: Check if `actual_boost` matches `boost` value

---

## Reporting Results

Please provide:

1. **Test Summary**:
   - ✅ or ❌ for each phase
   - Any warnings/errors from console

2. **Data Table** from Phase 3 (filled out)

3. **Console Logs** for:
   - Q=0 gpm (Phase 1)
   - Q=500 gpm (Phase 2)
   - Any flow rate where BHP decreased or physics violated

4. **Screenshots**:
   - Main window showing gauges at Q=0
   - Main window showing gauges at Q=2000
   - Console logs if any errors

5. **Rheology Confirmation**:
   - Your R600, R300 values
   - Fitted τ_y, K, n from console

---

## Success Criteria

The test PASSES if:
- ✅ At Q=0: Friction=0, ECD=MW
- ✅ BHP increases monotonically from 0 to 2000 gpm
- ✅ Friction increases monotonically with flow
- ✅ No physics violation warnings/errors in console
- ✅ Yield stress boost applied correctly (✓ in logs)
- ✅ User rheology parameters confirmed in use

The test FAILS if:
- ❌ Friction > 0 at Q=0
- ❌ BHP decreases at any point when Q increases
- ❌ Physics violation errors in console
- ❌ Yield stress not applied (dp_before = dp_after when τ_y > 0)
- ❌ Default rheology being used instead of user input

---

## Notes

- **V26 has extensive debugging**: Use the console logs actively
- **Wait for stabilization**: Don't change Q too quickly (10-15 sec per change)
- **Check section flows**: `window.__qAnnAbove_m3s` and `window.__qAnnBelow_m3s` should match Q_DS when no losses
- **Friction regime**: Check if laminar/transition/turbulent is correct for Reynolds number
- **No artificial cutoffs**: V26 uses smooth transitions, no hard cutoffs

---

## Reference: V26 Improvements Over V22

1. **Explicit yield stress in turbulent flow** (lines 644-653):
   ```javascript
   if (tauY > 0) {
     dp_yield = (4 * tauY) / Dh;
     dp += blend * dp_yield;  // Add yield contribution
   }
   ```

2. **Comprehensive friction logging** (lines 687-715)

3. **Smooth laminar-turbulent transitions** (Re: 2000-2300)

4. **Zero flow handling** (line 670):
   ```javascript
   if (Q <= 0) return 0;  // Explicit zero friction at zero flow
   ```

5. **Section flow verification** before friction calc (lines 1617-1622)

---

**Good luck with testing! Report all findings - we need to stress the code to find any remaining issues.**
