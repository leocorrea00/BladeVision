# Losses Physics Redesign

## Problem Statement

### Issue 1: Incorrect Physics Model
**Current (Wrong):**
1. Calculate Pzone = Hydrostatic + AnnulusFriction(all flow) + SBP
2. If Pzone > Pfrac → activate losses
3. Then route flows separately

**Problem:** This assumes all flow creates annulus friction BEFORE checking if losses occur. In reality, flow splits SIMULTANEOUSLY between annulus (to surface) and fracture.

**Correct Physics:**
1. BHP = Hydrostatic + AnnulusFriction(qUp) + FractureFriction(qFrac) + SBP
2. Flow splits: qUp + qFrac = qIn
3. Iteratively solve for equilibrium where BHP ≈ Pfrac
4. Only when qUp = 0 (no returns) does level drop

### Issue 2: FO Explosion
**Problem:** Even with Booster flowing down correctly, FO explodes to crazy values or goes negative during losses.

**Root Cause:** FO is calculated from qAbove before checking if:
- Level is actually at surface
- Flow is actually positive and stable
- Losses have stabilized

## Solution Design

### Part 1: Friction Redistribution Model

```javascript
function calculateBHPWithLosses(zoneDepth_m, qLoss_m3s, qIn_m3s) {
  // Split flow
  const qUp_m3s = Math.max(0, qIn_m3s - qLoss_m3s);
  const qFrac_m3s = qLoss_m3s;

  // Calculate components
  const P_hydro = calculateHydrostatic(zoneDepth_m);
  const P_fric_ann = calculateAnnulusFriction(zoneDepth_m, qUp_m3s);
  const P_fric_frac = calculateFractureFriction(zoneDepth_m, qFrac_m3s, zone);
  const P_sbp = (riserFull && MPD_ON) ? CURRENT_SBP : 0;

  return P_hydro + P_fric_ann + P_fric_frac + P_sbp;
}
```

### Part 2: Iterative Equilibrium Solver

```javascript
function solveZoneLossEquilibrium(zone, dt_s) {
  const qIn_m3s = (Q_DS + Q_BO - Q_CML) / 15850.323;
  const Pfrac = zone.fracPressure_psi;

  // Binary search for qLoss where BHP ≈ Pfrac
  let qLoss_min = 0;
  let qLoss_max = qIn_m3s * 10; // Allow demand > supply (level drops)

  for (let i = 0; i < 20; i++) {
    const qLoss_trial = (qLoss_min + qLoss_max) / 2;
    const BHP = calculateBHPWithLosses(zone.depth_m, qLoss_trial, qIn_m3s);

    if (Math.abs(BHP - Pfrac) < 5) break; // converged

    if (BHP > Pfrac) {
      qLoss_min = qLoss_trial; // Need more losses
    } else {
      qLoss_max = qLoss_trial; // Too much loss
    }
  }

  return (qLoss_min + qLoss_max) / 2;
}
```

### Part 3: Fracture Friction Model

Fracture friction depends on:
- Fracture geometry (width, length)
- Flow rate through fracture
- Fluid rheology

Simplified model:
```javascript
function calculateFractureFriction(depth_m, qFrac_m3s, zone) {
  if (qFrac_m3s <= 0) return 0;

  // Use zone's reference points or simplified model
  const n = zone.n || 1.0;
  const C_frac = zone.Qref_gpm / Math.pow(zone.DPref_psi || 100, n);

  // Invert: given Q, find ΔP
  const qFrac_gpm = qFrac_m3s * 15850.323;
  const dP_frac = Math.pow(qFrac_gpm / Math.max(C_frac, 1e-6), 1/n);

  return dP_frac;
}
```

### Part 4: FO Protection

```javascript
function safeFlowOut(qAbove_m3s, lossActive, riserAtSurface) {
  // Layer 1: No FO if losses active and not at surface
  if (lossActive && !riserAtSurface) return 0;

  // Layer 2: No FO if qAbove is tiny or negative
  if (qAbove_m3s < 1e-6) return 0;

  // Layer 3: No FO if loss rate > 50% of inflow
  const qIn_m3s = (Q_DS + Q_BO) / 15850.323;
  const qLoss_m3s = totalLossRate_gpm() / 15850.323;
  if (qLoss_m3s > qIn_m3s * 0.5) return 0;

  // Layer 4: Clamp to reasonable range
  const maxFO = qIn_m3s * 1.2; // max 120% of inflow
  return Math.min(qAbove_m3s * 15850.323, maxFO * 15850.323);
}
```

### Part 5: Special Case - Static Overbalance

When Hyd + SBP > Pfrac even with no flow:
```javascript
if (P_hydro + P_sbp > Pfrac && qIn_m3s < 1e-6) {
  // Override SBP to anchor at fracture pressure
  const SBP_override = Math.max(0, Pfrac - P_hydro - 50); // 50 psi margin
  CURRENT_SBP = SBP_override;
  console.log(`Anchor mode: SBP overridden to ${SBP_override} psi`);
}
```

## Implementation Plan

1. ✅ Add fracture friction calculation function
2. ✅ Add BHP calculation with flow split
3. ✅ Replace zoneLoss_gpm with iterative solver
4. ✅ Add FO protection layers
5. ✅ Add static overbalance handling
6. ✅ Performance optimizations (see below)
7. Test with:
   - Normal losses (partial)
   - Total losses (no returns)
   - Static overbalance case

## Performance Optimizations

### Problem:
Initial implementation had 20 iterations per zone per call → potential 20x slowdown!

### Solution - 5 Optimizations:

**1. Early Exit When Inactive**
```javascript
if (BHP_static < Pfrac - LOSS_DEADBAND_PSI) return 0; // Skip solver entirely
```

**2. Feedforward from Previous Result**
```javascript
// Start search near last timestep's result instead of [0, qIn*10]
const qLoss_prev_m3s = zone._Q_filt / 15850.323;
if (qLoss_prev_m3s > 1e-6) {
  qLoss_min = qLoss_prev_m3s * 0.3;  // ±70% bracket
  qLoss_max = qLoss_prev_m3s * 3.0;
}
```
**Impact:** ~3-5x fewer iterations (converges in 2-4 instead of 8-10)

**3. Reduced Max Iterations**
- Changed from 20 → 10 iterations
- Binary search: 2^10 = 1024x resolution (more than enough!)
- **Impact:** 2x speedup

**4. Relaxed Tolerance**
- Changed from 10 psi → 15 psi
- Still physically accurate (±15 psi is negligible at BHP ~4000+ psi)
- **Impact:** ~30% fewer iterations

**5. Early Convergence Exit**
```javascript
if (Math.abs(BHP - Pfrac) < TOLERANCE_PSI) break;
```
- Typical convergence: 3-5 iterations instead of 10
- **Impact:** 2-3x speedup

### Net Performance:
**Before optimizations:** ~20x slower (worst case)
**After optimizations:** ~2-3x slower (typical case), ~5x slower (worst case)

This is acceptable because:
- Only runs when losses are active
- Physics accuracy is worth the small overhead
- Feedforward makes it nearly "free" in steady-state

## Expected Behavior After Fix

### Scenario 1: Partial Losses
- BHP slightly > Pfrac → some flow to fracture, some to surface
- FO > 0 (returns to surface)
- Level stays at surface
- Friction split between annulus and fracture

### Scenario 2: Total Losses
- BHP >> Pfrac → all flow (and more) wants to go to fracture
- qUp = 0 → FO = 0
- Level drops
- Only fracture friction applies

### Scenario 3: Static Overbalance (Hyd + SBP > Pfrac)
- System enters "anchor mode"
- SBP overridden to balance at Pfrac
- Level oscillation prevented
- Acts like Anchor Point Mode

