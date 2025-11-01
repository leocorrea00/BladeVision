# BOP System Implementation Status

## Completed Work ✅

### 1. Direction Button UI (Issue #1)
- ✅ Replaced single toggle with separate UP/DOWN buttons
- ✅ Green highlight shows selected direction
- ✅ Applied to both Choke and Kill lines
- ✅ Much clearer visual indication
- **Files:** `Losses V26.txt` lines 6707-6760 (Choke), 6812-6865 (Kill)

### 2. Formation Fluids Bar Repositioning (Issue #4)
- ✅ Moved from `well.x - 35` to `well.x - 60`
- ✅ No longer clashes with Kill Line
- ✅ Clear 18-pixel separation maintained
- **Files:** `Losses V26.txt` line 5059

### 3. Influx Advection & Circulation (Issues #2, #3)
- ✅ Formation fluids now advect (circulate) with mud flow
- ✅ Downward advection in below-BOP section (lines 1456-1465)
- ✅ Upward advection in below-BOP section (lines 1487-1496)
- ✅ Transfer from below-BOP to riser when flowing UP (lines 1526-1538)
- ✅ Influx properly moves through wellbore and reaches surface
- ✅ Formation fluids bar now shows fluid movement
- **Files:** `Losses V26.txt` stepAdvection() function

### 4. Influx Density Mixing (Issue #3)
- ✅ Mixed density calculation already implemented
- ✅ `effectiveCellDensity_ppg()` properly mixes mud + formation fluids (lines 1889-1928)
- ✅ Used in `annulusAvgPPG()` and `ecdAtDepth()`
- ✅ BHP calculations reflect lighter formation fluids
- ✅ Gas significantly reduces BHP, oil/water moderate effect

### 5. Pressure-Driven Kick Mode (Issue #9)
- ✅ Continuous influx when BHP < PP confirmed working (lines 1259-1263)
- ✅ Influx rate = ProductivityIndex × (PP - BHP)
- ✅ Influx added to correct cell based on kick depth
- ✅ With advection implemented, influx now properly circulates

---

## Remaining Work 🚧

### 6. UP Flow Pressure Balance (Issues #5, #6, #7) - **CRITICAL & COMPLEX**

**Current Problem:**
- User controls flow rates Q_CHOKE, Q_KILL for UP direction
- Physics doesn't work this way - can't force fluid UP through closed system

**Required Solution:**
- Remove flow rate control for UP direction (make read-only/calculated)
- User controls only backpressures: CHOKE_PRESSURE_psi, KILL_PRESSURE_psi, SBP
- System calculates flow distribution based on pressure balance
- Multiple parallel paths: Riser (if BOP open), Choke line, Kill line
- Each path has: Hydrostatic + Friction(Q) + Applied Pressure = BHP
- Solve non-linear system for Q values that satisfy all paths

**Implementation Approach:**

```javascript
// Pseudo-code for pressure-driven UP flow solver
function calculateUPFlowDistribution() {
  const BHP_at_BOP = pressureAtDepth(BOP_DEPTH);

  // Identify open paths
  const paths = [];
  if (!BOP_CLOSED) paths.push({name: 'RISER', ...});
  if (CHOKE_LINE_OPEN && CHOKE_DIRECTION === 'UP') paths.push({name: 'CHOKE', ...});
  if (KILL_LINE_OPEN && KILL_DIRECTION === 'UP') paths.push({name: 'KILL', ...});

  // For each path, solve: BHP = P_hydro + P_friction(Q) + P_applied
  // Use iterative solver with conductance approximation

  // Initial guess: distribute by inverse resistance
  // Iterate: adjust Q until all paths have same source pressure

  return {Q_riser, Q_choke, Q_kill};
}
```

**Key Challenges:**
1. Friction is non-linear with flow rate (power law, turbulent/laminar)
2. Multiple paths in parallel create coupled system
3. Need stable iterative solver that converges quickly
4. Handle cases where requested pressure cannot be achieved
5. UI must show calculated flow rates (read-only) for UP direction

**Files to Modify:**
- `routeAnnulusFlowsWithLosses()` - BOP CLOSED section (lines 2450-2610)
- Flow routing for BOP OPEN state
- UI controls for Choke/Kill flow rates (conditionally disable for UP)
- Add visual indicators showing "pressure-driven" vs "pump-driven" mode

**Recommended Phased Approach:**
1. **Phase A:** Implement simplified linear conductance solver
2. **Phase B:** Add iteration for non-linear friction convergence
3. **Phase C:** Handle special cases (insufficient pressure, one dominant path, etc.)
4. **Phase D:** UI updates and read-only flow displays
5. **Phase E:** Extensive testing with different BOP/MPD/Choke/Kill combinations

### 7. Gas Expansion & Migration (Issue #10)

**Ideal Gas Mode:**
- Implement PV = nRT expansion as pressure decreases
- Gas migrates upward AND expands as it rises
- Volume increase affects wellbore fluid level
- Requires pressure tracking at each cell depth

**Real Gas Mode:**
- Add Z-factor pre-calculated lookup table
- Based on reduced pressure (P/Pc) and temperature (T/Tc)
- Interpolate Z from table to avoid iterations
- More accurate for high-pressure gas

**Implementation:**
```javascript
// Pre-calculated Z-factor table
const Z_TABLE = {
  // [Pr][Tr] = Z
  // Pr = P/Pc (reduced pressure)
  // Tr = T/Tc (reduced temperature)
};

function calculateGasExpansion(cellIndex, isAboveBOP) {
  const gasAmount = isAboveBOP ? gasPhaseFrom_above[cellIndex] : gasPhaseFrom_below[cellIndex];
  if (gasAmount < 0.001) return;

  const depth = calculateCellDepth(cellIndex, isAboveBOP);
  const P = pressureAtDepth(depth);
  const T = temperatureAtDepth(depth); // Need to add temperature profile

  if (GAS_MODEL === 'IDEAL') {
    // V2/V1 = (P1/P2) * (T2/T1)
    const referenceP = porePressure_psi;
    const expansionFactor = referenceP / Math.max(P, 14.7);
    // Expand gas in cell...
  } else if (GAS_MODEL === 'REAL') {
    const Pr = P / GAS_Pc;
    const Tr = T / GAS_Tc;
    const Z = interpolateZFactor(Pr, Tr);
    // V = (Z*n*R*T)/P
  }
}
```

### 8. BHP Transition Smoothing (Issue #11)

**Problem:**
- Sharp BHP drops when flow transitions between laminar/turbulent
- Occurs at Reynolds number boundaries
- Two safeguards already implemented but still seeing jumps

**Root Cause:**
- Friction factor changes discontinuously at regime boundaries
- Need smoother transition zone

**Solution:**
```javascript
function calculateFrictionFactor(Re, roughness) {
  const RE_LAMINAR = 2100;
  const RE_TURBULENT = 4000;
  const RE_TRANSITION_ZONE = 2500;
  const BLEND_WIDTH = 500;

  if (Re < RE_LAMINAR) {
    return 64 / Re; // Laminar
  } else if (Re > RE_TURBULENT) {
    return colebrookFriction(Re, roughness); // Turbulent
  } else {
    // Smooth blend in transition zone
    const f_laminar = 64 / Re;
    const f_turbulent = colebrookFriction(Re, roughness);
    const blendFactor = smoothstep(RE_LAMINAR, RE_TURBULENT, Re);
    return lerp(f_laminar, f_turbulent, blendFactor);
  }
}

function smoothstep(edge0, edge1, x) {
  const t = clamp((x - edge0) / (edge1 - edge0), 0, 1);
  return t * t * (3 - 2 * t); // Hermite interpolation
}
```

---

## Testing Recommendations

### After UP Flow Pressure Balance Implementation:

**Test Case 1: BOP Closed, Choke Line Only UP**
1. Close BOP
2. Open choke line, direction UP
3. Set choke pressure = 500 psi
4. Start pumping DS
5. Verify: Flow OUT calculated (not user-set)
6. Verify: Choke line pressure approaches 500 psi (if possible)
7. Verify: Warning if 500 psi cannot be achieved

**Test Case 2: BOP Closed, Both Choke & Kill UP**
1. Close BOP
2. Open both lines, direction UP
3. Set choke pressure = 300 psi, kill pressure = 500 psi
4. Start pumping DS
5. Verify: Flow splits based on resistance (more through choke = lower resistance)
6. Verify: Both pressures achieved if physically possible

**Test Case 3: BOP Open + MPD + Kill Line UP**
1. Open BOP, turn on MPD (SBP = 300 psi)
2. Open kill line, direction UP, set kill pressure = 600 psi
3. Start pumping DS
4. Verify: Most flow goes through riser (lower resistance)
5. Verify: Some flow through kill line only if BHP high enough
6. Verify: Kill line flow may be zero if 600 psi cannot be achieved

**Test Case 4: Gas Kick with Circulation**
1. Enable formation fluids (GAS)
2. Set pressure-driven mode, PP > BHP
3. Start circulation
4. Verify: Gas enters at kick depth
5. Verify: Gas migrates upward in formation fluids bar
6. Verify: Gas reaches surface and exits
7. Verify: BHP drops as gas reduces hydrostatic
8. Verify: Gas expands as it rises (if gas expansion implemented)

---

## Architecture Notes

### Flow Routing State Machine

```
BOP OPEN:
  - Riser path available
  - Choke/Kill lines optional (if open)
  - MPD affects riser path resistance
  - Normal circulation path

BOP CLOSED:
  - Riser isolated
  - Only Choke/Kill lines cross BOP
  - Must have at least one line open for UP flow
  - If no UP paths: closed well (BHP increases)

DOWN Flow:
  - User controls flow rate (pump-driven)
  - SPP calculated from friction + pressure balance
  - Choke/Kill lines act like booster line

UP Flow:
  - User controls backpressure (pressure-driven)
  - Flow rate calculated from pressure balance
  - Must solve multi-path distribution
  - Choke/Kill lines act like CML (extract from wellbore)
```

### Pressure Balance Equations

For each UP flow path `i`:
```
BHP_source = P_hydro_i + P_friction_i(Q_i) + P_applied_i

Where:
- BHP_source = common source pressure (BHP at BOP or annulus)
- P_hydro_i = hydrostatic in path i (ρ * g * h)
- P_friction_i = friction loss (function of Q_i, geometry, rheology)
- P_applied_i = user-set backpressure at surface
- Q_i = flow rate in path i (unknown, to be solved)

Constraint:
Sum(Q_i) = Q_available_from_source
```

Solve this system iteratively:
1. Guess initial Q distribution
2. Calculate P_friction for each path
3. Adjust Q values to balance pressures
4. Repeat until convergence

---

## Git History

### Commits Made:
1. `a982991` - BOP System Part 8: Open/Close valves & closed well physics
2. `5d5007e` - BOP System Part 9: UI improvements (direction buttons, formation bar)
3. `08b5e4d` - Implement influx advection and fluid tracking (Issues #2, #3, #9)

### Branch:
`claude/fix-kick-influx-logic-011CUeNqLFmFHUw7KBQ4SJ5c`

---

## Next Steps

1. ✅ **Document current status** (this file)
2. 🚧 **Design pressure balance solver** (detailed algorithm)
3. 🚧 **Implement multi-path UP flow logic** (major refactor)
4. 🚧 **Update UI for pressure-driven mode** (read-only flow rates UP)
5. 🚧 **Add gas expansion physics** (Ideal + Real gas)
6. 🚧 **Fix BHP transition smoothing** (smoother friction blending)
7. 🚧 **Comprehensive testing** (all BOP/MPD/Choke/Kill combinations)

**Estimated Remaining Effort:** 3-4 hours of focused development

---

## Questions for User

1. **UP Flow Solver:** Prefer iterative solver (slower, accurate) or simplified analytical (faster, approximate)?
2. **Gas Properties:** What gas? (CH4, H2S, CO2) - affects Pc, Tc for Real Gas mode
3. **Temperature Profile:** Constant or gradient? Need for gas expansion calculations
4. **Convergence Tolerance:** How accurate must pressure balance be? (e.g., ±10 psi acceptable?)
5. **UI Feedback:** Show calculated vs requested pressures? Warning messages format?

---

*Status Document Created: Session continuing UP flow logic implementation*
*Last Updated: Current session*
*Contact: Claude Code session on branch claude/fix-kick-influx-logic-*
