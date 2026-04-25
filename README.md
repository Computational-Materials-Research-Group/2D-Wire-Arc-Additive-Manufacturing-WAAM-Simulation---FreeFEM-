# 2D Wire Arc Additive Manufacturing (WAAM) Simulation - FreeFEM++

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/WAAM-Arc%20Deposition-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Goldak-Double%20Ellipsoid-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Multi--Layer-Element%20Activation-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/DOI-10.5281%2Fzenodo.19770792-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A 2D finite element simulation of <b>Wire Arc Additive Manufacturing (WAAM)</b> using FreeFEM++.
  Models a moving electric arc depositing low-carbon steel layer by layer onto a substrate
  using the Goldak double-ellipsoid volumetric heat source with quiet element activation.
</p>
<img width="1008" height="772" alt="waam wall" src="https://github.com/user-attachments/assets/97cad0a4-49cf-4b6b-a279-8ba3682729f5" />

---

## Citation

If you use this code in your research, please cite:

```bibtex
@software{mishra_2026_waam,
  author    = {Mishra, A.},
  title     = {2D Wire Arc Additive Manufacturing (WAAM) Simulation - FreeFEM++},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.19770792},
  url       = {https://doi.org/10.5281/zenodo.19770792}
}
```

Plain text citation:

> Mishra, A. (2026). *2D Wire Arc Additive Manufacturing (WAAM) Simulation - FreeFEM++*. Zenodo. https://doi.org/10.5281/zenodo.19770792

---

## Physics

Wire Arc Additive Manufacturing (WAAM) is a directed energy deposition process that uses an electric arc as the heat source and a metal wire as feedstock to build large near-net-shape components layer by layer. The thermal history of each deposited layer determines microstructure, residual stress, and mechanical properties of the final part.

This simulation models the following coupled phenomena:

- Transient 2D heat conduction with implicit Euler time integration
- Goldak double-ellipsoid volumetric heat source (asymmetric front/rear ellipsoids)
- Multi-layer deposition with quiet element activation method
- Alternating scan direction per layer (bi-directional deposition)
- Real-time melt pool detection and cumulative HAZ tracking
- Layer-by-layer matrix reassembly as new material activates

---

## Geometry

```
y = 18mm  |==========================|  Layer 4 active  (16-18mm)
y = 16mm  |==========================|  Layer 3 active  (14-16mm)
y = 14mm  |==========================|  Layer 2 active  (12-14mm)
y = 12mm  |==========================|  Layer 1 active  (10-12mm)
          |                          |
          |    Steel Substrate       |  Hsub = 10mm
          |                          |
y = 0     |__________________________|
          0          Lx = 80mm
```

- `Nl = 4` deposited layers, each `Hlayer = 2mm` thick
- Layer 1 scans Left to Right, Layer 2 Right to Left, alternating
- Arc y-centre sits at the midplane of each active layer
- All layers above current are quiet (near-zero properties) until activated

---

## Material Parameters

### Low-Carbon Steel Wire (ER70S)

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Density | rhoM | 7800 | kg/m3 |
| Specific heat | cpM | 600 | J/kg/K |
| Thermal conductivity | kkM | 35 | W/m/K |
| Melting temperature | TmM | 1773 | K |
| Convection coefficient | hc | 15 | W/m2/K |
| Initial temperature | T0 | 300 | K |

### Quiet Element Properties (inactive layers)

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Density | rhoQ | rhoM x 1e-3 | kg/m3 |
| Conductivity | kkQ | kkM x 1e-4 | W/m/K |

Quiet elements use near-zero properties to remain thermally inactive until the arc reaches their layer. This avoids singularity in the system matrix while keeping the full domain mesh intact throughout the simulation.

---

## Arc / Heat Source Parameters

### Process Inputs

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Arc voltage | Varc | 20 | V |
| Arc current | Iarc | 180 | A |
| Thermal efficiency | etaArc | 0.85 | - |
| Effective arc power | Qarc | etaArc x V x I = 3060 | W |
| Travel speed | varc | 8 | mm/s |

### Goldak Double-Ellipsoid Parameters

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Front semi-axis | af | 4 | mm |
| Rear semi-axis | ar | 8 | mm |
| Depth semi-axis | cc | 3 | mm |
| Front heat fraction | ff | 0.6 | - |
| Rear heat fraction | fr | 1.4 | - |

---

## Governing Equations

### Transient Heat Conduction

```
rho(x) * cp(x) * dT/dt - div(k(x) * grad(T)) = Q_arc(x, y, t)
```

### Goldak Double-Ellipsoid Source

```
Front ellipsoid (x >= xc):
  Qf(x,y) = [6*sqrt(3)*ff*Qarc / (pi*sqrt(pi)*af*af*cc)]
           * exp(-3*(x-xc)^2/af^2) * exp(-3*(y-yc)^2/cc^2)

Rear ellipsoid (x < xc):
  Qr(x,y) = [6*sqrt(3)*fr*Qarc / (pi*sqrt(pi)*ar*ar*cc)]
           * exp(-3*(x-xc)^2/ar^2) * exp(-3*(y-yc)^2/cc^2)
```

Where `(xc, yc)` is the moving arc centre. `ar > af` correctly models the elongated
heat trail behind the arc that is physically observed in arc welding and WAAM.

### Boundary Conditions

```
-k * dT/dn = hc * (T - T0)    on all edges (Newton convection)
```

### Melt Pool Detection

```
MeltPool(x,t) = 1    if T(x,t) >= TmM
MeltPool(x,t) = 0    if T(x,t) <  TmM

MeltEver(x)   = 1    if T(x,t') >= TmM for any t' <= t  (latched)
MeltDepth(x,t) = T(x,t) - TmM            (positive inside pool)
```

---

## Numerical Method

| Aspect | Choice |
|--------|--------|
| Spatial discretisation | Finite Element Method (FEM) |
| Element type | P1 (linear) for temperature, P0 (constant) for material |
| Time integration | Implicit Euler (unconditionally stable) |
| Matrix strategy | varf + A^-1 * b; matrix rebuilt once per layer |
| Linear solver | Conjugate Gradient (CG) |
| Material assignment | Expression-based P0 (evaluated at element centroids) |
| Layer activation | Quiet element method; properties updated per layer |
| Mesh | Fixed throughout; no remeshing between layers |

The system matrix `A` is rebuilt **once per layer** (not every timestep) since material properties only change at layer activation events. The RHS vector `b` is rebuilt every timestep to track the moving arc position.

---

## Output Fields

Each `.vtu` file contains the following fields:

| Field | Description | Range |
|-------|-------------|-------|
| `Temperature` | Nodal temperature | 300 to 1773 K |
| `MeltPool` | Instantaneous melt indicator | 0 or 1 |
| `MeltEver` | Cumulative melt / HAZ track | 0 or 1 (latched) |
| `MeltDepth` | T minus TmM, signed | negative to 0 |
| `Material` | Region: 0=quiet, 1=substrate, 2/3/4/5=layer number | integer |

---

## Repository Structure

```
waam-freefem/
|
|-- waam.edp                       # Main FreeFEM++ simulation script
|-- waam_results/
|   |-- waam.pvd                   # ParaView collection file
|   |-- result_0005.vtu            # Saved at step 5
|   |-- result_0010.vtu            # Saved at step 10
|   |-- ...                        # Every 5 steps per layer
|-- README.md
```

---

## How to Run

### Requirements

- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org
- Output directory `D:\freefem++\` must exist before running

### Step 1 - Run the simulation

```bash
FreeFem++ waam.edp
```

The script will:
1. Build the full domain mesh (substrate + all 4 quiet layers)
2. Refine mesh in the deposition zone
3. Loop over 4 layers, activating each before its arc pass
4. Run arc scan for each layer (L to R or R to L alternating)
5. Save VTU files every 5 steps to `D:\freefem++\waam_results\`
6. Write `waam.pvd` linking all timesteps

Console output per saved step:
```
--- Activating layer 1 ---
Layer 1/4  Step 5/2000   t=0.025s   xarc=0.04mm   Tmax=1773.0K   Melt=12 nodes
Layer 1/4  Step 10/2000  t=0.05s    xarc=0.08mm   Tmax=1773.0K   Melt=18 nodes
--- Layer 1 complete ---
--- Activating layer 2 ---
Layer 2/4  Step 5/2000   t=10.025s  xarc=79.96mm  Tmax=1773.0K   Melt=15 nodes
```

### Step 2 - Open in ParaView

1. `File > Open` > navigate to `D:\freefem++\waam_results\`
2. Change Files of type to `All Files (*.*)`
3. Select `waam.pvd` > OK
4. Choose PVD Reader when prompted > OK
5. Click `Apply`
6. Set colour field to `Material` to verify layer activation
7. Switch to `Temperature` and click `Rescale to Data Range Over All Timesteps`
8. Press `Play` to watch layer-by-layer deposition

### Step 3 - Visualize melt pool

**Option A - Melt pool animation**
```
Colour field:   MeltPool
Colour map:     Blue-Red
Press Play
```

**Option B - Solidus boundary contour**
```
Filters > Contour
  Contour By:   MeltDepth
  Value:        0.0
  Apply
  Colour:       White
Press Play
```

**Option C - Full deposited HAZ track**
```
Colour field:   MeltEver
Shows all material that was ever molten across all 4 layers
```

**Option D - Layer activation sequence**
```
Colour field:   Material
  0 = quiet (not yet deposited)
  1 = substrate
  2 = layer 1
  3 = layer 2
  4 = layer 3
  5 = layer 4
```

---

## What to Look for in Results

### Layer-by-Layer Activation

Watch the `Material` field: layers appear one at a time as the arc reaches each new height. Each layer starts as quiet (0) and switches to its layer number as the arc activates it.

### Alternating Scan Direction

Layer 1 arc moves left to right. Layer 2 moves right to left. This bidirectional strategy is standard in WAAM to balance thermal gradients and reduce warping. The asymmetry of the Goldak ellipsoid (`ar > af`) means the heat trail always follows the arc correctly regardless of direction.

### Reheating of Previous Layers

Each new layer partially reheats the layer below it. The `MeltEver` field accumulates all remelted zones across all layers, showing the thermal cycling history that determines final grain structure.

### Melt Pool Size Evolution

The melt pool grows slightly in later layers because the substrate and previous layers are already warm. A larger melt pool in later layers is physically expected and indicates correct thermal accumulation behaviour.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Out of bound in operator []` | P0 field assigned via element index loop `[][k]` | Use expression-based assignment `field = expr(x,y)` |
| `End of String` lex error | Non-ASCII character in string | Replace em dashes, arrows with plain ASCII |
| `Compile error: )` | `^` operator in `int2d` expression | Replace `af^2` with `afsq = af*af` precomputed |
| Temperature stays at 300K | Layer not activated before arc reaches it | Check `yBot` and `yTop` arrays are correctly indexed |
| Matrix singular | Quiet elements have exactly zero conductivity | Use `kkQ = kkM * 1e-4` not zero |
| All layers active at once | Expression uses `y > Hsub` instead of per-layer bounds | Use `(y > yLB)*(y <= yLT)` for each layer |
| `D:\freefem++\` not found | Parent directory missing | Create `D:\freefem++\` manually before running |

---

## Extending the Model

| Extension | What to change |
|-----------|----------------|
| More layers | Increase `Nl`; `Htot`, layer arrays, scan pattern all auto-update |
| Different material (Ti-6Al-4V) | `rhoM=4430`, `kkM=7`, `cpM=560`, `TmM=1933` |
| Different material (Al alloy) | `rhoM=2700`, `kkM=167`, `cpM=900`, `TmM=933` |
| Wider bead | Increase `af`, `ar`, `cc` Goldak parameters |
| Deeper penetration | Increase `Iarc` or decrease `varc` |
| Inter-pass cooling dwell | Add a cooling loop between layer activations |
| Radiation BC | Add `sigma*eps*(T^4 - T0^4)` nonlinear term on boundaries |
| Residual stress | Couple second variational problem for displacement field |
| 3D model | Replace mesh with mesh3, use int3d, P13d elements |

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:

- **Share** - copy and redistribute the material in any medium or format
- **Adapt** - remix, transform, and build upon the material

Under the following terms:

- **Attribution** - You must give appropriate credit to akshansh11 and provide a link to this repository
- **NonCommercial** - You may not use the material for commercial purposes

Copyright 2026 akshansh11. All rights reserved for commercial use.
