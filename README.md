# nostos

An interactive nanoparticle PBPK explorer. Set a particle's diameter, surface
charge, core density, and coating, then watch a physiologically based
pharmacokinetic model predict where an IV dose ends up across plasma, liver,
spleen, tumor, and brain over 48 hours.

The particle properties act only through biophysics: Stokes–Einstein diffusion,
gravitational margination, pore sieving, opsonization, and macrophage wrapping.
The model structure follows the tumor-bearing nanomedicine PBPK framework of
Dogra et al. (2020).

## Running it

It's one static file with no dependencies and no build step.

```sh
open index.html
```

Any modern browser works. Moving a slider restarts the simulation. Hover over
any panel to read every compartment at that time point. A table view at the
bottom shows the same values without hovering.

## Model

### Compartments

Each organ has an interstitial compartment. Liver and spleen also have a
phagocytic (Kupffer cell / splenic macrophage) pool that particles are captured
into straight from the blood. Amounts are in % injected dose (%ID), starting
from a 100 %ID bolus into plasma.

```
                ┌─── k_in ───▶ interstitium ─── k_out ───┐
   plasma ──────┤                                         ├──▶ plasma
     │          └─── k_mps ──▶ phagocytic pool ──k_deg──▶ degraded
     └── CL_renal ──▶ urine
```

### Equations

With $C_p = A_p / V_p$ and $C_{t,i} = A_{t,i} / V_i$:

$$\frac{dA_{t,i}}{dt} = k_{in,i}\,C_p - k_{out,i}\,C_{t,i} - \delta_i\,k_{deg}\,A_{t,i}$$

$$\frac{dA_{m,i}}{dt} = k_{mps,i}\,A_p - k_{deg}\,A_{m,i}$$

$$\frac{dA_p}{dt} = -\sum_i \left( k_{in,i}\,C_p - k_{out,i}\,C_{t,i} + k_{mps,i}\,A_p \right) - CL_{renal}\,C_p$$

$\delta_i = 1$ for compartments without lymphatic drainage (tumor, brain), where
retained particles degrade in place. Integration is explicit RK4 with
$\Delta t = 0.02$ h.

### How particle properties enter

| Process | Form | Depends on |
|---|---|---|
| Diffusivity | $D_B = k_BT / 6\pi\mu r$ (Stokes–Einstein) | size |
| Margination | $v_{sed} = 2\Delta\rho\,g\,r^2 / 9\mu$, factor $1 + v_{sed}/50\ \text{nm s}^{-1}$ | size, density |
| Pore sieving | $1 - \sigma = (1 - r/r_{pore})^2$, zero when $r \ge r_{pore}$ (Ferry) | size vs. fenestration |
| Opsonization | coating baseline × $(1 + a\,\lvert\zeta\rvert/40)$, $a = 1.8$ if cationic, else 1 | charge, coating |
| Wrapping time | $t_{wrap} = 4\pi r^2 \gamma / p$ (Lunov et al.), shown as a derived value | size |
| Renal filtration | $CL = GFR / (1 + e^{(d - 8)/1.2})$, cutoff near 8 nm | size |

### Transport by organ

| Organ | Fenestration radius | Influx $k_{in}$ | Outflow $k_{out}$ | Phagocytic capture |
|---|---|---|---|---|
| Liver | 100 nm | $L(1-\sigma)$ | lymph flow $L$ | yes, reference density |
| Spleen | 250 nm | $L(1-\sigma)$ | lymph flow $L$ | yes, 0.3× liver |
| Tumor | 350 nm | $PS_T(1-\sigma)$ | none (EPR retention) | no |
| Brain | 0.6 nm | $L(1-\sigma)$ + transcytosis | slow glymphatic, 0.04 mL/h | no |

Lymph flow is $L = Q/500$. The tumor permeability–surface product
$PS_T = 0.0417$ mL/h is back-calculated from Dogra et al. (2020), Table 2:
$P = 1.1\times10^{-7}$ mm/s at 90 nm and $S = 40$ mm²/mm³ for a 2 mL tumor give
0.032 mL/h, which is then divided by that particle's sieving fraction.

Phagocytic capture is
$k_{mps} = K_{MPS} \cdot \text{density} \cdot f_{ops} \cdot (d/80)^{0.8} \cdot \text{margination}$,
with $K_{MPS} = 0.10$ h⁻¹.

Physiology is rat scale (200 g body weight): $V_p = 11.8$ mL, GFR = 78.6 mL/h,
$k_{deg} = 0.012$ h⁻¹.

## Verification

- Mass is conserved to within 10⁻⁶ %ID, with no negative amounts, across all
  144 combinations of the slider extremes.
- RK4 at $\Delta t = 0.02$ h matches $\Delta t = 0.005$ h to 4 significant figures.
- The model reproduces these trends from the literature without being fitted
  to them:
  - Renal clearance switches off above about 8 nm diameter.
  - Tumor delivery peaks near 20 nm at about 5 %ID, inside the 7–22 nm optimum
    reported by Dogra et al. (2020).
  - Liver and spleen together take 50–60 %ID for most particles.
  - At the same size and +30 mV charge, exposed-amine particles leave plasma
    about 5× faster than PEG-shielded ones (half-life 0.8 h vs. 4.2 h). This
    matches the shielded- vs. exposed-amine result in Dogra et al. (2018).

## Limitations

This tool is for teaching and exploration. It isn't calibrated for
quantitative prediction.

- **Opsonization is phenomenological.** The coating and charge multipliers are
  chosen to give realistic relative behavior, not fitted to data. Dogra et al.
  (2020) treat surface chemistry as out of scope.
- **$K_{MPS}$ is an effective rate.** The single-particle wrapping time
  (about 0.1 s for 80 nm) isn't used as a population rate constant. It's shown
  only as a derived quantity.
- **No vascular contribution to organ signal.** Organ curves count only
  interstitial and phagocytic particles, not particles in the organ's blood.
  They aren't directly comparable to imaging region-of-interest data such as
  the SPECT measurements in Dogra et al. (2018).
- **The tumor is homogeneous.** There's no spatial penetration, necrotic core,
  or interstitial pressure.
- **Fenestration radii are representative values.** Real fenestration varies
  widely across species, strains, and tumor types.

## References

- Dogra P, Butner JD, Ruiz Ramírez J, et al. A mathematical model to predict
  nanomedicine pharmacokinetics and tumor delivery. *Comput Struct Biotechnol J*
  2020;18:518–531. [doi:10.1016/j.csbj.2020.02.014](https://doi.org/10.1016/j.csbj.2020.02.014)
- Dogra P, Adolphi NL, Wang Z, et al. Establishing the effects of mesoporous
  silica nanoparticle properties on in vivo disposition using imaging-based
  pharmacokinetics. *Nat Commun* 2018;9:4551. [PMC6208419](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6208419/)
- Parrot M, et al. A minimal PBPK model to predict
  biodistribution of silica nanoparticles in mice. PAGE 2024.
  [Abstract](https://www.page-meeting.org/Abstracts/a-minimal-pbpk-model-to-predict-biodistribution-of-silica-nanoparticles-in-mice/)
