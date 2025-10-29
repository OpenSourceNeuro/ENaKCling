<p align="left"><img width="270" height="170" src="https://github.com/OpenSourceNeuro/Spikeling-V2/blob/main/Images/SpikyLogo.png">

<h1 align="center"> ENaKCling </h1></p>
<h3 align="center">  A hands‑on neuron model where you can dial the Nernst: real‑time, knob‑controlled Na⁺/K⁺/Cl⁻ (±Ca²⁺) dynamics, chloride homeostasis (KCC2/NKCC1), and synaptic conductances on a microcontroller.*</h3></p>
<p align="center"><h6 align="right">developed by M.J.Y. Zimmermann, L. Zanetti</h6></p>

<br></br>


## What is ENaKCling?
ENaKCling (pronounced **E‑Na‑K‑Cl‑ing**) is an open‑source teaching and exploration platform for cellular neurophysiology. It runs a **conductance‑based single‑compartment neuron** whose **ion concentrations and reversal potentials are user‑adjustable in real time** via front‑panel controls. The model includes:

- Hodgkin–Huxley–style Na⁺/K⁺ channels (with leak), optional Ca²⁺ current
- Dynamic **intracellular [Na⁺]ᵢ, [Cl⁻]ᵢ** and extracellular **[K⁺]ₒ** (with pump/glia terms)
- **Chloride homeostasis** via **KCC2** (K⁺–Cl⁻ cotransport) and **NKCC1** (Na⁺–K⁺–2Cl⁻ cotransport)
- Excitatory (AMPA‑like) and inhibitory (GABA
a) synapses with burst/Poisson drivers
- Temperature‑aware reversal potentials (Nernst), optional GHK voltage equation
- Fully documented parameters and equations (below)

The project targets Arduino/ESP32‑class MCUs, with live DAC/LED/serial outputs for use in labs and demos.


---

## Model summary
We use a **single isopotential compartment** with membrane capacitance \(C_m\) and surface area \(A\). Currents are positive outward. The membrane equation is

```math
C_m\,\frac{dV}{dt} = -\Big(I_\mathrm{Na} + I_\mathrm{K} + I_\mathrm{Cl} + I_\mathrm{Ca} + I_\mathrm{L} + I_\mathrm{pump} - I_\mathrm{syn} - I_\mathrm{app}\Big)\,.
```

Each ionic current is conductance‑based (unless stated otherwise):

```math
I_x = g_x\,a_x^p b_x^q\,(V - E_x),\qquad x\in\{\mathrm{Na},\mathrm{K},\mathrm{Cl},\mathrm{Ca}\}
```
with first‑order gating (Hodgkin–Huxley):


```math
\frac{da}{dt}=\alpha_a(V)(1-a)-\beta_a(V)a,\qquad \frac{db}{dt}=\alpha_b(V)(1-b)-\beta_b(V)b.
```

(You can also compile with a **Rush–Larsen** update for stability of fast gates.)

**Reversal potentials** use **Nernst** by default (temperature \(T\) settable):
```math
E_{\mathrm{ion}} = \frac{RT}{zF}\,\ln\!\left(\frac{[\mathrm{ion}]_o}{[\mathrm{ion}]_i}\right)\qquad (z=+1\text{ for Na⁺/K⁺, } z=-1\text{ for Cl⁻, } z=+2\text{ for Ca²⁺}).
```
Optionally, enable **GHK** for the multi‑ion resting potential.

---

## Dynamic ion concentrations
ENaKCling updates concentration state variables once per time step using flux–current conversion. Let \(\Omega_i\) be the intracellular volume, \(\Omega_o\) a reference extracellular volume (bath), \(F\) Faraday's constant, and \(\gamma = A/(F\,\Omega)\) a geometry factor.

### Sodium (intracellular)
```math
\frac{d[\mathrm{Na}]_i}{dt} = -\gamma_i\,I_{\mathrm{Na}} - 3\,\gamma_i\,I_{\mathrm{pump}} + D_{\mathrm{Na}}\,( [\mathrm{Na}]_o - [\mathrm{Na}]_i ),
```
where the Na⁺/K⁺‑ATPase removes 3 Na⁺ per cycle and \(D_{\mathrm{Na}}\) lumps diffusion/buffering.

### Potassium (extracellular)
```math
\frac{d[\mathrm{K}]_o}{dt} = \gamma_o\,I_{\mathrm{K}} - 2\,\gamma_o\,I_{\mathrm{pump}} - U_\mathrm{glia}([\mathrm{K}]_o) - D_{\mathrm{K}}\,([\mathrm{K}]_o - [\mathrm{K}]_o),
```
with glial uptake \(U_\mathrm{glia}\) (sigmoid or Michaelis–Menten) and bath diffusion toward \([\mathrm{K}]_o^{\*}.\)

### Chloride (intracellular) with KCC2/NKCC1
Two terms model electroneutral cotransporters that set \(E_{\mathrm{Cl}}\):

**Flux forms** (phenomenological, match transporter stoichiometry):
```math
J_{\mathrm{KCC2}} = k_{\mathrm{KCC2}}\big([\mathrm{K}]_i[\mathrm{Cl}]_i - [\mathrm{K}]_o[\mathrm{Cl}]_o\big),\qquad
J_{\mathrm{NKCC1}} = k_{\mathrm{NKCC1}}\big([\mathrm{Na}]_o[\mathrm{K}]_o[\mathrm{Cl}]_o^2 - [\mathrm{Na}]_i[\mathrm{K}]_i[\mathrm{Cl}]_i^2\big).
```
They give net **Cl⁻ influx** positive (units: mol·m⁻²·s⁻¹). Update equation:
```math
\frac{d[\mathrm{Cl}]_i}{dt} = +\gamma_i\,(I_{\mathrm{Cl}} + I_{\mathrm{GABA_A}}) - \frac{A}{\Omega_i}\big(J_{\mathrm{KCC2}} + J_{\mathrm{NKCC1}}\big) - D_{\mathrm{Cl}}\,([\mathrm{Cl}]_i - [\mathrm{Cl}]_i).
```

> **Intuition:** KCC2 drives \(E_{\mathrm{Cl}}\) toward \(E_{\mathrm{K}}\) (Cl⁻ extrusion), NKCC1 loads Cl⁻ (depolarizes \(E_{\mathrm{Cl}}\)). Set \(k_{\mathrm{KCC2}}\) high to get strong inhibition (\(E_{\mathrm{Cl}}\approx E_{\mathrm{K}}\)); set \(k_{\mathrm{NKCC1}}\) high to mimic immature neurons or pathology.

### Calcium (optional)
A compact pool model with pump extrusion and linear buffer factor \(\kappa\):
```math
\frac{d[\mathrm{Ca}]_i}{dt} = -\eta\,I_{\mathrm{Ca}} - \frac{([\mathrm{Ca}]_i - [\mathrm{Ca}]_\mathrm{rest})}{\tau_{\mathrm{Ca}}} - J_{\mathrm{PMCA}}([\mathrm{Ca}]_i),
```
with \(J_{\mathrm{PMCA}} = J_\mathrm{max}\,\frac{[\mathrm{Ca}]_i}{[\mathrm{Ca}]_i + K_{\mathrm{PMCA}}}\) and \(\eta\) the current→flux conversion (includes \(\kappa\)).

### Na⁺/K⁺‑ATPase current
Independent **electrogenic** pump current used above:
```math
I_{\mathrm{pump}} = I_{\max}\,\frac{[\mathrm{Na}]_i^3}{[\mathrm{Na}]_i^3 + K_{\mathrm{Na}}^3}\,\frac{[\mathrm{K}]_o^2}{[\mathrm{K}]_o^2 + K_{\mathrm{K}}^2}.\
```

---

## Synapses (conductance‑based)
- **AMPA** (excitatory): \(I_\mathrm{AMPA} = g_\mathrm{A}\,s(t)(V-E_\mathrm{AMPA})\)
- **GABA\_A** (inhibitory, Cl⁻): \(I_\mathrm{GABA\_A} = g_\mathrm{G}\,s(t)(V-E_{\mathrm{Cl}})\)
- **Kinetics:** either alpha‑function \(s(t)=\tfrac{t}{\tau}\,e^{1-t/\tau}\) or double‑exponential \(ds/dt = \alpha(1-s)T(t) - s/\tau_d\).
- **Presynaptic drive:** Poisson/burst generators with rate/packet controls.

---

## Front‑panel controls

| Control | Symbol | Typical range | Effect |
|---|---|---|---|
| **Temperature** | \(T\) | 292–310 K | Affects all Nernst/GHK reversals |
| **[Na⁺]ᵢ** | \([\mathrm{Na}]_i\) | 5–20 mM | Shifts $E_{\mathrm{Na}}$, pump load |
| **[Na⁺]ₒ** | \([\mathrm{Na}]_o\) | 120–160 mM | Shifts $E_{\mathrm{Na}}$ |
| **[K⁺]ᵢ** | \([\mathrm{K}]_i\) | 120–160 mM | Shifts $E_{\mathrm{K}}$, KCC2 term |
| **[K⁺]ₒ** | \([\mathrm{K}]_o\) | 2–12 mM | Extracellular K⁺ dynamics/glia |
| **[Cl⁻]ᵢ** | \([\mathrm{Cl}]_i\) | 3–25 mM |Sets inhibition polarity via $E_{\mathrm{Cl}}$ |
| **[Cl⁻]ₒ** | \([\mathrm{Cl}]_o\) | 110–140 mM | Bath chloride |
| **g\_Na, g\_K, g\_L** | \(g_x\) | 0–1 (normed) | Channel densities |
| **g\_Ca** | \(g_{\mathrm{Ca}}\) | 0–1 (normed) | Enables Ca²⁺ electrogenesis |
| **Pump strength** | \(I_{\max}\) | 0–2× | Energy‑dependent stabilization |
| **KCC2 strength** | \(k_{\mathrm{KCC2}}\) | 0–3× | Drives $E_{\mathrm{Cl}}\to E_{\mathrm{K}}$ |
| **NKCC1 strength** | \(k_{\mathrm{NKCC1}}\) | 0–3× | Loads Cl⁻ (depolarizes $E_{\mathrm{Cl}}$) |
| **Syn AMPA/GABA gain** | \(g_\mathrm{A}, g_\mathrm{G}\) | 0–5× | Exc/Inh balance |
| **Syn time constants** | \(\tau, \tau_d\) | 1–50 ms | PSP shape |
| **Stimulus** | \(I_\mathrm{app}\) | ±2 nA equiv. | Current clamp input |

> All controls are bounded/sanitized in firmware to avoid non‑physiological states.

---

## Default parameters
> Units: mV, ms, mM, nS, nA, K in Kelvin.

| Parameter | Value | Notes |
|---|---:|---|
| $C_m$| 200 pF | Small soma |
| Area $A$ | 10,000 µm² | For current↔flux conversion |
| $[\mathrm{Na}]_i, [\mathrm{Na}]_o$ | 12, 145 mM | Intracellular/extracellular |
| $[\mathrm{K}]_i, [\mathrm{K}]_o^{*}$ | 140, 3.5 mM | Bath $[\mathrm{K}]_o^{*}$ |
| $[\mathrm{Cl}]_i, [\mathrm{Cl}]_o$ | 7, 130 mM | Mature neuron baseline |
| $[\mathrm{Ca}]_i, [\mathrm{Ca}]_o$ | 0.1 µM, 2 mM | Effective pool |
| $g_\mathrm{Na}, g_\mathrm{K}, g_\mathrm{L}$ | 120, 36, 0.3 nS | HH‑like scale (rescaled) |
| $E_\mathrm{L}$ | −54.4 mV | Leak reversal |
| $g_\mathrm{Ca}$ | 0–2 nS | Optional |
| Pump $I_{\max}$ | 0.2 nA | Tunable |
| $k_{\mathrm{KCC2}}, k_{\mathrm{NKCC1}}$ | 1.0, 0.2 (arb.) | Scale factors |
| $\tau_{\mathrm{Ca}}$ | 80 ms | Ca²⁺ removal |
| $J_\mathrm{max}, K_{\mathrm{PMCA}}$ | 0.1 µM/ms, 0.3 µM | PMCA pump |
| Temp $T$ | 306 K | 33 °C demo labs |

---

## Timing & precision targets
- **Numerical step**: \(\Delta t = 0.05\text{–}0.1\,\mathrm{ms}\) (20–10 kHz); HH gates via Rush–Larsen; ions/synapses every step.
- **Panel scan**: ≥1 kHz; **DAC/LED** update: ≥5 kHz for Vm, **display/serial**: 30–60 Hz.
- **Jitter**: <5% of \(\Delta t\); use a fixed‑rate timer ISR feeding a ring buffer.
- **Stability**: clamp state to physical ranges (e.g., \([\mathrm{Cl}]_i>0\)); soft‑reset on NaN.

---

## Optional compact models (for comparison)
ENaKCling can optionally compile simpler dynamical systems to contrast with the full ion‑aware model:

- **Hindmarsh–Rose (3D)** for qualitative bursting/spiking (dimensionless)
- **Izhikevich (2D)** for rich spiking patterns with low CPU

These are disabled by default but useful for teaching trade‑offs between **biophysical interpretability** and **computational efficiency**.

---

## Implementation notes
- **Integration**: default explicit Euler (fast), optional **RK2** for stiff regimes.
- **Geometry**: set \(A\) and \(\Omega\) to preserve realistic current↔concentration scaling.
- **Units**: all concentrations in mM; Faraday conversion uses SI internally.
- **Safety**: parameter changes are rate‑limited to avoid numerical shocks.

---

## Literature
If you use ENaKCling in a course or paper, please cite this repository and the foundational works below. A non‑exhaustive list (open‑access links where possible):

1. **Hodgkin & Huxley (1952)**. *A quantitative description of membrane current…* J Physiol 117:500–544. [[PMC]](https://pmc.ncbi.nlm.nih.gov/articles/PMC1392413/)  
2. **Nernst / GHK**. Nernst for single‑ion reversal; GHK for multi‑ion membranes. [[Nernst overview]](https://www.ncbi.nlm.nih.gov/books/NBK538338/) · [[GHK explainer]](https://en.wikipedia.org/wiki/Goldman_equation)  
3. **Cressman et al. (2009)**. *Influence of sodium and potassium dynamics on excitability; pump, glia, diffusion.* J Comput Neurosci. [[PDF]](https://complex.gmu.edu/neural/papers/ullah_jcomp_neuro1_2009.pdf)  
4. **Düsterwald et al. (2018)**. *Biophysical models reveal the relative importance of transporter proteins and impermeant anions in chloride homeostasis.* eLife. [[PMC]](https://pmc.ncbi.nlm.nih.gov/articles/PMC6200395/)  
5. **Blaesse et al. (2009)**; **Doyon et al. (2016)**. Reviews on CCCs (KCC2/NKCC1) and neuronal Cl⁻ regulation. [[Neuron 2009 PDF]](https://www.cell.com/neuron/pdf/S0896-6273%2809%2900200-1.pdf) · [[Neuron 2016]](https://www.cell.com/neuron/fulltext/S0896-6273(16)00147-1)  
6. **Chizhov et al. (2019)**. *Mathematical model of Na–K–Cl homeostasis with KCC2/NKCC1 terms.* PLoS Comput Biol. [[PMC]](https://pmc.ncbi.nlm.nih.gov/articles/PMC6420042/)  
7. **Yamada–Koch–Adams (1989/1998)**. Compact **Ca²⁺** pool dynamics with pumps/buffers (Methods in Neuronal Modeling). [[chapter PDF]](https://authors.library.caltech.edu/records/9dnr2-hez54/files/258.pdf?download=1)  
8. **Destexhe–Mainen–Sejnowski (1994/1998)**. Kinetic synapse models; alpha/double‑exponential forms. [[chapter PDF]](https://papers.cnl.salk.edu/PDFs/Kinetic%20Models%20of%20Synaptic%20Transmission%201998-3229.pdf)

> **Note on transporter flux forms:** The phenomenological KCC2/NKCC1 flux equations used here respect stoichiometry (1K:1Cl and 1Na:1K:2Cl) and are commonly employed in ionic‑homeostasis models (e.g., Düsterwald 2018; Chizhov 2019). See source comments for unit conversions.

---
