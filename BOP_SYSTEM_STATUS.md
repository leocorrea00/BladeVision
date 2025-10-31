# BOP System Implementation Status

## ✅ COMPLETED (Parts 1 & 2)

### Foundation & Friction Calculations
- **BOP State Variables**: BOP_CLOSED flag, choke/kill line parameters
- **Choke Line**: Flow rate, direction (UP/DOWN), pressure, ID, fluid grid
- **Kill Line**: Flow rate, direction (UP/DOWN), pressure, ID, fluid grid
- **Grid Initialization**: chokeGrid[N_ABV], killGrid[N_ABV] with fluid tracking
- **Friction Functions**:
  - `chokeFrictionPsiLength()` - rheology-aware friction for choke line
  - `killFrictionPsiLength()` - rheology-aware friction for kill line
  - `bopLinePressureContribution()` - calculates BHP impact from choke/kill lines

### Pressure System Integration
- **pressureAtDepth()** updated to include BOP line contribution
- When BOP CLOSED + lines flow UP: Friction + Line Pressure adds to BHP
- When BOP CLOSED + lines flow DOWN: Line Pressure acts as SPP
- SBP disabled when BOP closed

### Well Geometry UI
- **Editable Line IDs**: Booster ID, Choke ID, Kill ID (all in inches)
- Real-time conversion to meters
- Located in Gel & Geometry panel

### BOP Well Control UI
- **BOP State Toggle**: Visual button (green=OPEN, red=CLOSED)
- **Choke Line Controls**:
  - Flow Rate input (gpm)
  - Pressure input (psi)
  - Direction button (UP/DOWN)
- **Kill Line Controls**:
  - Flow Rate input (gpm)
  - Pressure input (psi)
  - Direction button (UP/DOWN)
- **Help Text**: Explains BOP operation modes
- Positioned below Losses UI

### Visualization (Part 4)
- **BOP Indicator**: Red rectangle when closed, green outline when open at 2000m depth
- **Choke Line Graphics**: Right side of well, color-coded by direction (blue=UP, orange=DOWN)
- **Kill Line Graphics**: Adjacent to choke line, same color scheme
- **Flow Arrows**: Three arrows per line showing flow direction
- **Labels**: Flow rate and direction displayed on rotated text

---

## 🚧 PENDING (Part 5 - Integration Testing)

### Flow Routing Logic for BOP Closed
**File**: `routeAnnulusFlowsWithLosses()` function

#### When BOP OPEN (Current Behavior - Works)
```
DS + BO + TF - CML → Losses → Annulus Up → Surface
```

#### When BOP CLOSED (Needs Implementation)

**Below BOP (Isolated from Riser)**:
- DS flows down drillstring, returns up annulus below BOP
- If losses active: DS return → losses (follows Up/Down/Exit logic)
- **Kicks enter here** and flow upward (need to respect Up/Down/Exit logic with losses)
- Choke/Kill DOWN: Pump fluid into below-BOP annulus
- Choke/Kill UP: Extract fluid from below-BOP annulus (goes to surface)
- No connection to riser (BOP blocks it)

**Above BOP (Riser - Isolated from Below)**:
- Booster can still pump into riser (from surface)
- TopFill can fill riser from surface
- CML can remove from riser
- **No DS return** (BOP blocks it)
- Only exit is via Choke/Kill UP (if configured)

#### Pressure Balance Logic (Both Lines UP)
When both Choke and Kill lines flow UP simultaneously:

1. **Calculate Available Pressure**:
   ```
   P_available = BHP - (Friction_line + Applied_Pressure)
   ```

2. **Priority Logic**:
   - Line with LOWER resistance gets more flow
   - If Choke has 500 psi applied + 200 psi friction = 700 psi total
   - If Kill has 400 psi applied + 150 psi friction = 550 psi total
   - Kill line will take more flow (lower resistance path)

3. **Flow Split**:
   ```javascript
   const R_choke = choke_friction + CHOKE_PRESSURE_psi;
   const R_kill = kill_friction + KILL_PRESSURE_psi;
   const total_R = R_choke + R_kill;

   const Q_choke_fraction = R_kill / total_R;  // Inverse relationship
   const Q_kill_fraction = R_choke / total_R;

   Q_CHOKE_actual = Q_available_belowBOP * Q_choke_fraction;
   Q_KILL_actual = Q_available_belowBOP * Q_kill_fraction;
   ```

### Choke/Kill Line Advection
**File**: `stepAdvection()` function

Need to add fluid tracking for choke and kill lines similar to booster:

```javascript
// CHOKE LINE ADVECTION
if (CHOKE_DIRECTION === "DOWN" && Q_CHOKE > 0) {
  // Fluid enters at TOP (surface), exits at BOTTOM (BOP)
  // Pushes into below-BOP annulus
  const qChoke_m3s = Q_CHOKE / 15850.323;
  const A_choke = Math.PI * CHOKE_LINE_ID_m * CHOKE_LINE_ID_m / 4;
  const v_choke = qChoke_m3s / A_choke;
  const chokeStep_m = TIME_ACCEL * BASE_M_PER_FRAME * (v_choke / vref);

  accChoke += chokeStep_m;
  while (accChoke >= CELL_M) {
    chokeGrid.pop();  // Remove from bottom
    chokeGrid.unshift(MW_CHOKE);  // Add at top
    accChoke -= CELL_M;
  }
}

if (CHOKE_DIRECTION === "UP" && Q_CHOKE > 0) {
  // Fluid enters at BOTTOM (BOP), exits at TOP (surface)
  // Takes fluid from below-BOP annulus
  // Similar logic with shift direction reversed
}

// Repeat for KILL LINE
```

### Formation Fluid Circulation Fixes
**Current Issue**: Formation fluids enter wellbore but may not be properly mixed/tracked

**Required Fixes**:
1. Ensure kicks are added to annulus grids at correct cell
2. Kicks follow advection with upward flow
3. When kicks reach BOP:
   - If BOP OPEN: Continue up through riser (mix into annAboveGrid)
   - If BOP CLOSED: Can only exit via Choke/Kill UP
4. Visualization should show kick fluids in annulus
5. Density mixing already implemented in `effectiveCellDensity_ppg()`

### Visualization Updates

**BOP Indicator on Well Schematic**:
```javascript
// At BOP depth (2000m)
const y_bop = mapDepthToY(BOP_DEPTH);

if (BOP_CLOSED) {
  // Draw closed BOP (red rectangle/symbol)
  fill(255, 0, 0);
  noStroke();
  rect(well.x - 10, y_bop - 5, well.w + 20, 10);
  fill(255);
  textSize(10);
  text("BOP CLOSED", well.x + well.w/2, y_bop);
} else {
  // Draw open BOP (green outline)
  stroke(0, 255, 0);
  noFill();
  rect(well.x - 10, y_bop - 5, well.w + 20, 10);
}
```

**Choke/Kill Line Graphics**:
- Draw vertical lines parallel to riser
- Color-code based on direction (UP=blue, DOWN=orange)
- Show flow arrows
- Label with flow rate

---

## 📋 Implementation Checklist for Part 3

### High Priority (Critical for BOP Operation)
- [ ] Modify `routeAnnulusFlowsWithLosses()` to handle BOP closed state
- [ ] Implement pressure balance when both choke/kill lines flow UP
- [ ] Add choke/kill line advection in `stepAdvection()`
- [ ] Test BOP closed with DS circulation only
- [ ] Test BOP closed with choke line UP
- [ ] Test BOP closed with kill line UP
- [ ] Test BOP closed with both lines UP (pressure balance)

### Medium Priority (Kick Integration)
- [ ] Ensure kicks follow Up/Down/Exit logic with BOP closed
- [ ] Kicks should route to choke/kill lines when BOP closed
- [ ] Test kick circulation through choke/kill lines
- [ ] Verify kick density mixing with choke/kill line fluids

### Low Priority (Polish)
- [x] Add BOP visualization on well schematic
- [x] Add choke/kill line visualization
- [x] Add flow direction arrows
- [ ] Add diagnostic logging for BOP operations
- [ ] Add SPP calculations for choke/kill lines when flowing DOWN

---

## 🎯 Next Steps

1. **Test Current Implementation**:
   - Open simulator, verify BOP UI appears
   - Toggle BOP state (should see button color change)
   - Input choke/kill parameters (should update variables)
   - Check if pressureAtDepth includes BOP contribution

2. **Implement Flow Routing** (Most Complex):
   - Create separate flow paths for BOP closed
   - Handle riser isolation from below-BOP annulus
   - Implement pressure balance algorithm
   - Test with various configurations

3. **Add Advection**:
   - Implement choke line fluid tracking
   - Implement kill line fluid tracking
   - Ensure proper mixing at BOP interface

4. **Final Testing**:
   - Test all BOP operation modes
   - Verify pressure calculations
   - Check flow balance
   - Ensure kicks work correctly with BOP closed

---

## 💡 Design Philosophy

**Modularity**: BOP system integrates seamlessly with existing systems
**Safety**: BOP closed properly isolates riser from below-BOP pressure
**Realism**: Choke/kill line friction and pressure balance matches real operations
**Flexibility**: Users can configure all parameters for different scenarios

---

## 📝 Notes for Developers

- BOP system is backward compatible (BOP_CLOSED defaults to false)
- All friction calculations use existing rheology-aware functions
- Pressure contribution is additive (doesn't break existing pressure calcs)
- UI is positioned to avoid overlap with existing panels
- Choke/Kill line IDs editable in Geometry UI (consistent with other geometry inputs)

**Current Branch**: `claude/fix-kick-influx-logic-011CUeNqLFmFHUw7KBQ4SJ5c`
**Commits**:
- `81bde08`: Part 1 - Foundation and friction
- `60abdd2`: Part 2 - Complete BOP Well Control UI
- `029b835`: Part 3A - Flow routing with pressure balance
- `cf2f5e7`: Part 3B - Choke/kill line advection
- `cb96854`: Part 4 - Complete visualization

Parts 1-4 Complete! Ready for integration testing! 🚀
