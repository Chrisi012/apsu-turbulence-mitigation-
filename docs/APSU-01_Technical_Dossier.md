# APSU-01: Technical Dossier & Safety Case
### Detailed Sizing, Transient Dynamics, Thermal Limits and FMEA

---

## 1. System Mass & SWaP Budget

To comply with aerodynamic balance and flutter constraints on ultralight wings (Shark 600 wingtip/flaplet station), total subsystem weight is budgeted under 450 g:

| Subassembly | Primary Components | Estimated Mass |
|---|---|---|
| Supercapacitor Bank | 6x 50F / 2.7V EDLC (Maxwell/Eaton radial) | 88 g |
| High-ESR Decoupling Bank | 4x 1000 µF 25V Solid Aluminum-Polymer + MLCCs | 34 g |
| Power Electronics | Back-to-Back FETs, gate drivers, D2PAK 1.5Ω brake resistor | 42 g |
| EMI Stage (DO-160G §21) | Nanocrystalline common-mode core + differential chokes | 65 g |
| PCB Assembly | 4-layer 2oz copper board (100 mm x 60 mm) | 46 g |
| Housing & Hardware | Extruded aluminum enclosure + thermal interface pad | 85 g |
| Connectors & Harnessing | Radiall/Amphenol avionics-grade locking connectors | 25 g |
| **Total Subsystem Mass** | | **385 g (< 450 g target)** |

---

## 2. Hybrid Buffer Energy & Transient Sizing

The unit operates between an aircraft 14V Rotax bus and a direct-drive flaplet actuator demanding 25A peak current at 25 Hz.

### Energy Derivation across 3-Second Gust Encounter
* Nominal buffer float voltage: $V_{\text{nom}} = 13.8\text{ V}$
* Lower operational limit during burst: $V_{\text{min}} = 12.62\text{ V}$
* Equivalent EDLC series capacitance (6x 50F in series):
  $$C_{\text{eq}} = \frac{50\text{ F}}{6} = 8.33\text{ F}$$
* Net deliverable energy without generator assistance:
  $$\Delta E = \frac{1}{2} C_{\text{eq}} \left(V_{\text{nom}}^2 - V_{\text{min}}^2\right) = \frac{1}{2} (8.33) \left(13.8^2 - 12.62^2\right) \approx 129.8\text{ J}$$
* Over a 3.0-second burst duration, this represents an average power contribution of:
  $$P_{\text{buffer}} = \frac{129.8\text{ J}}{3.0\text{ s}} \approx 43.3\text{ W}$$
* At 13.2V mean bus level, this injects $\approx 3.28\text{ A}$ continuous equivalent buffer current, maintaining generator draw at a steady 5.18A during high-frequency duty cycles.

### Instantaneous Ohmic Drop ($t < 10\ \mu\text{s}$)
* 4000 µF Solid Polymer Bank ESR: $R_{\text{ESR,poly}} \approx 2.0\text{ m}\Omega$
* Internal shunt and trace impedance: $R_{\text{trace}} \approx 1.0\text{ m}\Omega$
$$\Delta V_{\text{instant}} = I_{\text{step}} \cdot (R_{\text{ESR,poly}} + R_{\text{trace}}) = 25\text{ A} \cdot 0.003\ \Omega = 0.075\text{ V}$$

---

## 3. Dynamic Brake Transient Thermal Limit

When the analog eFuse trips on overcurrent ($I_{\text{load}} > 40\text{ A}$), the actuator leads are shorted across a $1.5\ \Omega$ D2PAK surface-mount resistor to ground via a delayed FET.

* Peak instantaneous voltage (back-EMF / bus decay): $V_{\text{peak}} \approx 14.0\text{ V}$
* Peak instantaneous current:
  $$I_{\text{brake,peak}} = \frac{14.0\text{ V}}{1.5\ \Omega} \approx 9.33\text{ A}$$
* Peak instantaneous power:
  $$P_{\text{brake,peak}} = 14.0\text{ V} \cdot 9.33\text{ A} \approx 130.6\text{ W}$$
* Total energy dissipated across $65\text{ ms}$ centering transient (exponential decay approximation $\tau \approx 20\text{ ms}$):
  $$E_{\text{event}} = \int_0^{0.065} P(t)\, dt \approx \frac{1}{2} P_{\text{peak}} \cdot \tau_{\text{equiv}} \approx 130.6\text{ W} \cdot 0.025\text{ s} \approx 3.27\text{ J}$$
* **Thermal Evaluation:** A standard 10W D2PAK thick-film resistor supports single-pulse energy ratings up to $15\text{ J}$ for pulses under $100\text{ ms}$. The single-event stress remains within $22\%$ of the manufacturer pulse SOA.

---

## 4. Aero-Mechanical Centering Model

The $65\text{ ms}$ centering metric is derived from an analytical torque-balance equation:
$$J \ddot{\theta} + \left(B_{\text{mech}} + \frac{K_e K_t}{R_{\text{brake}} + R_{\text{winding}}}\right) \dot{\theta} + K_{\text{aero}} \theta = 0$$

* Combined flaplet and direct-drive rotor inertia: $J = 1.8 \cdot 10^{-4}\text{ kg}\cdot\text{m}^2$
* Dynamic aerodynamic restoring hinge moment: $K_{\text{aero}} \approx 0.42\text{ N}\cdot\text{m/rad}$ at $180\text{ km/h}$ TAS
* Counter-electromotive damping factor: $K_e = K_t = 0.045\text{ V}\cdot\text{s/rad}$
* Total dynamic loop impedance: $R_{\text{total}} = 1.5\ \Omega + 0.35\ \Omega = 1.85\ \Omega$

The resulting electrodynamic damping ratio ($\zeta \approx 0.68$) drives the surface to $| \theta | < 0.5^\circ$ neutral deflection in $65\text{ ms}$, avoiding flutter and aerodynamic overshoot.

---

## 5. Failure Mode and Effects Analysis (FMEA)

| Component / Subsystem | Failure Mode | Direct Effect | Mitigation Strategy |
|---|---|---|---|
| **eFuse Disconnect FET** | Fails Shorted (Drain-Source) | Actuator cannot be isolated on fault | Secondary upstream 30A aircraft thermal breaker; MCU flags telemetry alarm via CAN-FD |
| **Brake Shunt FET** | Fails Shorted | Line continuously clamped to 1.5Ω | Hardware interlock disables main eFuse if brake gate is high; prevents 14V bus short |
| **Hardware Comparator** | Latch-up / Stuck Low | Analog cutoff does not trigger at 40A | Redundant software interrupt via circular DMA ADC samples ($< 2.14\text{ ms}$ latency) |
| **Microcontroller Clock** | HSE/HSI Oscillator Stall | Supervisor loop stops executing | External TI TPS3851 window watchdog pulls Master Reset and drops eFuse gate line |
| **14V Bus Supply** | Engine Crank Sag ($V_{\text{bus}} < 9\text{ V}$) | Buffer dumps reverse energy into starter | Back-to-back N-channel FET configuration isolates reverse conduction path |
