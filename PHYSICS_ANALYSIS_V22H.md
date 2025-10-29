# Physics Analysis - Losses V22H

## Executive Summary

After comprehensive code review of Losses V22H, the physics implementation appears correct in principle, but requires stress testing across the full DS flow range (0-2000 gpm) to verify:

1. **Zero Flow (Q=0)**: ECD at TD should equal MW (no friction)
2. **Monotonic Increase**: BHP should increase continuously with flow rate
3. **User Rheology**: Rheology parameters from user input must be correctly applied

---

## Code Flow Analysis

### 1. Rheology Parameter Flow

```
User Input (R600, R300, R200, etc.)
  ↓
collectRheoPoints() → converts to shear rate & stress
  ↓
doRheologyFit() → fits Bingham, Power Law, and Herschel-Bulkley models
  ↓
fitParams object (stores K, n, τ_y for each model)
  ↓
getRheoForCalc() → retrieves params based on ACTIVE_RHEO_MODEL
  ↓
publishRheoToGlobals() → sets window.TAUY_Pa, window.K_PaSn, window.N_dim
  ↓
__dpPerM_annulus_GNF_fromQ() → uses these globals for friction calculation
```

**✓ Correct**: User rheology is being used

### 2. Pressure Calculation at TD

```javascript
// pressureAtDepth(TD_DEPTH) line 1538-1577
function pressureAtDepth(d){
  const fluidTop = annulusTopDepthGlobal();

  // Hydrostatic (only wetted depth)
  const depth_wet = d - fluidTop;
  const ecd_ppg = ecdAtDepth(d);
  const P_hydro = PSI_COEF * ecd_ppg * depth_wet;

  // Friction (rheology-aware)
  const P_fric = frictionPsi_withCML_rheo(d);

  // Surface back pressure (only when riser full and MPD ON)
  const P_sbp = (MPD_ON && riserFull) ? (CURRENT_SBP || 0) : 0;

  return P_hydro + P_fric + P_sbp;
}
```

**✓ Physics**: P_total = P_hydrostatic + P_friction + P_sbp

### 3. Friction Calculation

```javascript
// frictionPsi_withCML_rheo(d) line 1472-1508
function frictionPsi_withCML_rheo(d){
  // Get section flows (accounts for losses)
  const qAbove = window.__qAnnAbove_m3s;  // m³/s
  const qBelow = window.__qAnnBelow_m3s;  // m³/s

  // Calculate wet lengths for each section
  const L_riser = max(0, min(BOP_DEPTH, depth) - topRiser);
  const L_csg   = (depth > BOP_DEPTH) ? max(0, min(depth, SHOE_DEPTH) - topBelow) : 0;
  const L_oh    = (depth > SHOE_DEPTH) ? max(0, depth - max(topBelow, SHOE_DEPTH)) : 0;

  // ΔP/L for each section (psi/m) using rheology
  const dpdm_riser = __dpPerM_annulus_GNF_fromQ(qAbove, ID_ris, OD_ds, mName);
  const dpdm_csg   = __dpPerM_annulus_GNF_fromQ(qBelow, ID_csg, OD_ds, mName);
  const dpdm_oh    = __dpPerM_annulus_GNF_fromQ(qBelow, ID_oh,  OD_ds, mName);

  return dpdm_riser*L_riser + dpdm_csg*L_csg + dpdm_oh*L_oh;
}
```

**✓ Physics**: Friction = Σ(ΔP/L × Length) for each section

### 4. Zero Flow Handling

```javascript
// __dpPerM_pipe_GNF_fromQ() line 1398-1416
function __dpPerM_pipe_GNF_fromQ(Q_m3s, D_m, tauToShearRateFn){
  const Qtarget = max(0, Q_m3s);
  if (Qtarget <= 0 || D_m <= 0) return 0;  // ✓ Zero flow → zero friction

  // ... numerical inversion to find ΔP that yields Q ...
  return hi; // psi per meter
}
```

**✓ Physics**: When Q = 0, friction = 0

---

## Rheology Models Implemented

### Herschel-Bulkley (default)
```
τ = τ_y + K·γ̇^n
```
- τ_y: Yield stress (Pa) from window.TAUY_Pa
- K: Consistency index (Pa·s^n) from window.K_PaSn
- n: Flow index from window.N_dim

### Bingham Plastic
```
τ = τ_y + μ_p·γ̇
```
- τ_y: Yield point (Pa) from window.TAUY_Pa
- μ_p: Plastic viscosity (Pa·s) from window.MUP_PaS

### Power Law
```
τ = K·γ̇^n
```
- K: Consistency index (Pa·s^n) from window.K_PaSn
- n: Flow index from window.N_dim

---

## Expected Physics Behavior

### Test Condition
- **DS MW = Booster MW** (eliminate density differences)
- **Booster Flow = 0 gpm** (Booster pump OFF)
- **DS Flow Range: 0 → 2000 gpm**
- **P-Only Correction: OFF** (compressibility disabled)

### Expected Results

#### 1. At Q_DS = 0 gpm (No Flow)
```
Friction = 0 psi
BHP_TD = Hydrostatic only
ECD_TD = MW_DS (no friction boost)
```

**Test**: ECD_TD ≈ MW_DS (within 0.01 ppg tolerance)

#### 2. As Q_DS increases (500, 1000, 1500, 2000 gpm)
```
Friction increases (Q > 0)
BHP_TD increases monotonically
ECD_TD increases monotonically
```

**Test**: BHP(Q2) > BHP(Q1) for all Q2 > Q1

#### 3. Friction Relationship
For turbulent flow in annulus:
```
ΔP ∝ ρ·V² ∝ Q²
```

**Test**: Friction should grow approximately quadratically with Q in turbulent regime

---

## Potential Issues to Check

### 1. publishRheoToGlobals() Call Timing
**Location**: Lines 2824, 3305
**Risk**: If not called before first friction calculation, wrong rheology used
**Mitigation**: Called in setup (line 3305) after doRheologyFit()

### 2. Section Flow Calculation
**Location**: recomputeAnnulusSectionFlowsForLosses()
**Risk**: __qAnnAbove_m3s and __qAnnBelow_m3s not initialized
**Mitigation**: Check in frictionPsi_withCML_rheo (line 1476-1479)

### 3. Rheology Model Mismatch
**Risk**: User selects HB but globals have PL values
**Check**: ACTIVE_RHEO_MODEL must match the model used in __dpPerM_annulus_GNF_fromQ

### 4. Yield Stress in Turbulent Flow
**Question**: Should yield stress add pressure boost in turbulent regime?
**Current**: dpPerM_Refined_withRot includes yield via muApp (line 664)
**Physics**: In true turbulent flow, yield stress effect is debated

---

## Comparison with V26 Code (from conversation)

The user's previous conversation on branch `claude/fix-v26-friction-011CUPJMbykaueZjsTkLiMXG` mentioned:

> "The yield stress boost is being calculated correctly (67.8 Pa/m, 166.8 Pa/m, etc.) but NOT being added to the RESULT!"

**Key Difference**:
- **V26**: Had separate yield boost tracking and logging
- **V22H**: Yield stress integrated into friction formula via muApp

**V22H Approach (line 664)**:
```javascript
const muApp = tauY/gdot + K*Math.pow(gdot, n-1);
```

This includes yield stress directly in apparent viscosity, so no separate "boost" tracking needed.

---

## Recommended Tests

### Test 1: Zero Flow Verification
```
Set: Q_DS = 0, Q_BO = 0, MW_DS = MW_BO = 9.5 ppg
Expected: ECD_TD = 9.5 ppg ± 0.01
Expected: Friction_TD = 0 ± 0.1 psi
```

### Test 2: Monotonic Increase
```
Loop: Q_DS from 0 to 2000 gpm in 50 gpm steps
Check: BHP[i+1] >= BHP[i] for all i
Check: Friction[i+1] >= Friction[i] for all i > 0
```

### Test 3: Rheology Parameter Verification
```
Log: window.TAUY_Pa, window.K_PaSn, window.N_dim
Compare: fitParams.tau_y_HB_Pa, fitParams.K_HB_Pa_s_n, fitParams.n_HB
Expected: Values should match (via getRheoForCalc → publishRheoToGlobals)
```

### Test 4: Friction Scaling
```
Test points: 500, 1000, 1500, 2000 gpm
Calculate: Friction ratio F(2Q)/F(Q)
Expected: ~4 (turbulent regime, F ∝ Q²)
Tolerance: 3-5 range acceptable (non-Newtonian effects)
```

---

## Next Steps

1. **Create test HTML page** that systematically tests 0-2000 gpm
2. **Log all key variables** at each flow rate:
   - Q_DS, Q_BO
   - window.__qAnnAbove_m3s, window.__qAnnBelow_m3s
   - TAUY_Pa, K_PaSn, N_dim, ACTIVE_RHEO_MODEL
   - BHP_TD, ECD_TD, Friction_TD, Hydro_TD
3. **Verify monotonicity** and flag any violations
4. **Check zero flow** condition rigorously
5. **Compare with hand calculations** for one test point

---

## Conclusion

The V22H code structure appears physically sound:
- ✓ Rheology from user input is used
- ✓ Zero flow gives zero friction
- ✓ Pressure = hydrostatic + friction (+ SBP if applicable)
- ✓ Friction calculation uses proper rheology models

**However**, rigorous stress testing across full flow range is needed to verify:
- No unexpected friction drops or jumps
- Monotonic BHP increase with flow
- Correct ECD at zero flow
- No artificial cutoffs or corrections violating physics

The test must run with **Booster MW = DS MW** and **Booster Flow = 0** to isolate DS flow effects as requested by user.
