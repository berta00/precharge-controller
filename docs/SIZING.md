# Precharge-controller's sinzing file

## Index
1. [LV Power supply]()
2. [LV Logic]()
3. [LV Relay drive]()
4. [HV Sense]()
5. [HV Power]()

## 3. LV Power Supply

### 3.1 Reverse polarity & Gate protection (lv_supply and shutdown lines)

A P-Channel MOSFET (Q1) is used instead of a standard series diode to prevent reverse polarity damage. This design choice eliminates the typical $0.7\text{ V}$ forward voltage drop, maximizing efficiency and minimizing heat dissipation. 

PMOS (Q1) required specs:
- **Max $V_{DS} \leq -20\text{ V}$**
- **Low $R_{DS(on)}$**
- **Continuous current $I_{D} \geq 10\text{ mA}$** (for lv_supply line)
- **Continuous current $I_{D} \geq 250\text{ mA}$** (for shutdown line)

Choosen Vishay's Si3437DV specs:
- $V_{DS} = -30\text{ V}$
- $I_{D} = -4.8\text{ A}$
- $V_{GS(max)} = \pm 20\text{ V}$

The gate is pulled to ground via $R1 = 10\text{ k}\Omega$, ensuring the PMOS is fully enhanced (turned on) when a positive voltage is correctly applied. 
In protection of the MOSFET from voltage transients, a Zener diode (D1 / D3) is placed between the gate and source. Since the SI3437DV has a max $V_{GS}$ of $20\text{ V}$, a $15\text{ V}$ Zener clamps the voltage safely below the destruction limit.

### 3.2 Transient overvoltage protection (lv_supply and shutdown lines)

To protect the downstream linear regulator, logics chips, and relay coils from high-energy transients, a diode clamping stage is implemented.

TVS Diode (D2 / D13) required specs:
- **Standoff voltage $\geq 15\text{ V}$:** To avoid clamping normal battery fluctuations.
- **Clamping voltage $\leq 35\text{ V}$:** To prevent exceeding the linear regulator's maximum input voltage.

Choosen D2 / D13 specs:
- $V_{c} = 28\text{ V}$
- max $I_{s} = 2\text{ A}$

$R28$ and $R29$ acts as a series current-limiting resistor. During a severe overvoltage spike (e.g., $V_{spike} = 60\text{ V}$), the TVS diode rapidly enters avalanche breakdown and clamps the line at $28\text{ V}$. The surge current absorbed by the TVS is limited by $R28$ and $R29$ according to the formula:
$$
I_{s} = \frac{V_{spike} - V_{c}}{R}
$$
Inverting the formula to find $V_{spike}$:
$$
V_{spike} = I_{s} \cdot R + V_{c}
$$
Considering and $R = 10\space \Omega$:
$$
V_{spike} = 2 \cdot 10\space \Omega + 28V = 48\text{ V}
$$

### 3.3 Linear voltage stabilizer (lv_supply line)

The discrete logic block requires a highly stable voltage, due to the logic nature of the load.

Voltage Regulator (IC1) required specs:
- **High input voltage tolerance**
- **Output voltage $= 12\text{ V}$**

Choosen STMicroelectronics's L7812ABV specs:
- Max input voltage = $35\text{ V}$
- Output voltage = $12\text{ V}$
- Output current up to $1.5\text{ A}$

Following the manufacturer's datasheet recommendations two filtersing capacitor are placed in paralel with input and output lines:
- $C1 = 0.33\space \mu\text{F}$ (input)
- $C2 = 0.1\space \mu\text{F}$ (output)

### 3.4 Bypass filters (shutdown line)

Because the +12V_SHDW line supplies inductive loads (relay and contactor coils), switching events generate sharp current steps and voltage sags. A dual-stage bypass filter is utilized:

- **High-Frequency Bypass (C3):** A $0.1\space \mu\text{F}$ multilayer ceramic capacitor (MLCC) is placed locally to absorb high-frequency EMI.
- **Inductive Transient Bypass (C4):** To supply sudden transient currents when multiple relay coils switch simultaneously without causing severe voltage droops ($\Delta V$), a bulk capacitor is required. 

According to the Omron G9KB-1A12V relay datasheet, the nominal coil resistance is $R_{c} = 51\space \Omega$. To optimize power consumption and thermal performance, the system applies a holding voltage of $V_{\text{hold}} = 6\text{ V}$ (50% of the rated 12V) in relay drive block.

The holding current drawn by a single relay coil is:
$$
I_{\text{hold}} = \frac{V_{\text{hold}}}{R_{\text{coil}}} = \frac{6\text{ V}}{51\text{ }\Omega} \approx 117\text{ mA}
$$

Considering a maximum of 2 relay coils active simultaneously under holding conditions, the total load current is:
$$
I_{\text{load}} = 2 \times 117\text{ mA} = 234\text{ mA}
$$

Considering a maximum $\Delta V = 0.5\text{ V}$ on the line during a switching event lasting $\Delta t = 0.5\text{ ms}$, the required capacitance is calculated as:
$$
C_4 \ge \frac{I_{\text{load}} \cdot \Delta t}{\Delta V} = \frac{234\text{ mA} \cdot 0.5\text{ ms}}{0.5\text{ V}} = 234\text{ }\mu\text{F}
$$


## 2. LV Logic

### 2.1 Enable level shifter

The level is shifed by connecting the ENABLE input line to the gate of an NMOS. The gate at high input, connects the capacitor directly to ground discharging it imediatly.

NMOS (Q4) required specs:
- **Max $V_{DS} \geq 20V$**
- **Max $I_{DS} \geq 120\text{ mA}$**
- **Max $V_{GS(th)} \leq 3\text{ V}$**

Choosen Onsemi's 2N7002K:
- Inner diode protection
- $V_{DS} = 60\text{ V}$
- $I_{DS} = 800\text{ mA}$
- $V_{GS(th)} = 1\text{ V}$
- max $R_{DS(on)} = 4\space \Omega$ (wont interfeere with C6's discharge time)

### 2.2 Enable timer

Schmitt trigger (IC3) required specs are:
- **$12\text{ V}$ power domain:** In order to power it with +12_LOGIC line
- **Hysteresis voltage ($V_h$):** It's a defined separation between the positive ($V_p$) and negative ($V_n$) thresholds, in order to prevent oscillation if the input is around the triggering spot.
- **Unlimited input rise and fall times:** The trigger must be design to tollerate extremely slow voltage ramps generated by the capacitor charing, the minimum spec is around 50ms.
- **Low input current:** To ensure correct operations, the trigger must ensure minimal current draw from the input pin, in order to preserve the $\tau$ constant of the RC network.
- **Low current draw**
- **High noise margin**

Choosen Texas Instrument's CD40106B specs:
- $3\text{ V} \leq V_{DD} \leq 18\text{ V}$
- Infinite input rise time tollerance
- Negligible current draw on input line
- Negligible current draw
- High noise immunity ($\approx 45\% \cdot V_{DD}$)
- $V_h \approx 2.4\text{ V}$ at $12\text{ V}$
- $V_p \approx 7.1\text{ V}$ at $12\text{ V}$

Given these specs, we can size the whole timer.

Capacitor charge:
$$
V_C(t) = V_{DD} \cdot (1 - e^{-\frac{t}{R6 \cdot C6}})
$$

The inverted formula, in order to calculate $RC \space (= \tau)$:
$$
R6 \cdot C6 = -\frac{t}{\ln\left(1 - \frac{V_P}{V_{DD}}\right)}
$$
$$
R6 \cdot C6 = -\frac{50\text{ ms}}{\ln\left(1 - \frac{7.1\text{ V}}{12\text{ V}}\right)} = 0.0558\text{ s}
$$

Given this information, we can choose an RC couple which fits the $\tau$ contraint:
- $C6 = 1\space \mu\text{F}$ (not elettrolitic because they have high tollerances for a timer)
- $R6 = 56\text{ k}\Omega$ (standard value)

R19 is used as discharge resistence, in order to prevent a short circuit and protect the NMOS (Q4). In this case $I_{peak} = \frac{12\text{ V}}{R19}$. Choosing a resistor of $R19 = 100\space \Omega$, we are sure to preserve the NMOS and to not interfere with R6 and ensuring a stable down signal when enable is up. The discharge time with $\tau = 100\space \Omega \cdot 1\space \mu\text{F}$ is around $0.1\text{ ms}$, so i wont affect the timer functioning.

The timer wont be precise to the millisecond, due to component tollerances.

### 2.3 Logic chips

The schmitt trigger inverter has been already discussed in the previous section, because its main features are dictated by the timer.

And gate chip (IC2) requirements are:
- **$12\text{ V}$ power domain:** In order to power it with +12_LOGIC line.
- **Logic thresholds compatible with CD40106B:** The chip will need to interface with the inverter.
- **Low current draw**
- **High noise margin**

Choosen Texas Instrument's (CD4081B) specs:
- $3\text{ V} \leq V_{DD} \leq 18\text{ V}$
- Same logic family (CMOS 4000B series) as the schmitt trigger inverter
- Negligible current draw
- Quad 2-input architecture
- High noise immunity ($\approx 45\% \cdot V_{DD}$)

$R18 = 1\text{k }\Omega$ and $R24 = 10\text{k }\Omega$ are used to protect the IC2's A input pin. Using this configuration (with $R18$ in series with the main line, and $R24$ as pulldown resistore) we ensure protection for:
$$
I_{max} = \frac{V_{spike} - V_{c}}{R18}
$$
Inverting the formula:
$$
V_{max} = I_{max} \cdot R18 + V_{c}
$$
Considering the maximum input currento of the chip $I_{max} = 10\text{ mA}$ and the internal clamping voltage $V_{c} = 12.5\text{ V}$:
$$
V_{max} = (10\text{ mA} \cdot 1\text{ k}\Omega) + 12.5\text{ V} = 22.5\text{ V}
$$

Or gate chip (IC5) requirements are:
- **$12\text{ V}$ power domain:** In order to power it with +12_LOGIC line.
- **Logic thresholds compatible with CD40106B:** The chip will need to interface with the inverter and the and.
- **Low current draw**
- **High noise margin**

Chossen Texas Instrument's (CD4071B) specs:
- $3\text{ V} \leq V_{DD} \leq 18\text{ V}$
- Same logic family (CMOS 4000B series) as the schmitt trigger inverter
- Negligible current draw
- High noise immunity ($\approx 45\% \cdot V_{DD}$)

All the logic chips are choosen with the M96 SMD package ensuring higher fatigue failure and a reduction to parassite inductances. All the chips have a decoupling capacitor of $0.1\space \mu\text{F}$.


## 3. LV Relay drive

### 3.1 Driver

The NMOSs (Q3, Q5, Q6) act as electronic switches to energize and control the high-capacity Omron G9KB-1A12V relay coils. 

NMOS (Q3, Q5, Q6) required specs:
- **max $V_{DS} \ge 20\text{ V}$:** To safely withstand the 12V supply line and inductive transients.
- **max $I_{D} \ge 250\text{ mA}$**
- **Low $R_{DS(on)}$**

Chosen Diodes Incorporated's DMN6140L specs:
- $V_{DS} = 60\text{ V}$
- $I_{D} = 1.6\text{ A}$
- $V_{GS(th)} = 3.0\text{ V} \leq 12\text{ V}$ logic

**Gate Pull-Down Resistors (R11, R12, R13):** to mitigate flactuations when CMD_XXX_IN is down.

**Gate Series Resistors (R14, R15, R16):** Sized to limit the gate current and prevent. Choosing $R_{gate} = 1\text{ k}\Omega$, the peak gate current from a 12V control signal is limited to:
  $$
  I_{gate} = \frac{12\text{ V}}{1\text{ k}\Omega} = 12\text{ mA}
  $$
  This safely protects the upstream logic outputs.

### 3.2 Holding voltage

As Omron G9KB relay's datasheet recommendations for optimize power consumption and coil preservation, a power-saving holding circuit is implemented using a parallel RC network ($R25$ / $R26$ / $R27$ and $C14$ / $C15$ / $C16$) placed in series with each relay coil. This circuit applies full rated voltage ($12\text{ V}$) instantaneously for reliable pull-in, and then exponentially reduces the voltage to a safe holding level ($V_{\text{hold}} = 6\text{ V}$, corresponding to 50% of the rated voltage) after an initial actuation window of $t = 0.1\text{ s}$.

#### Sizing Calculation:
Given the relay coil resistance $R_{\text{coil}} = 51\text{ }\Omega$, the steady-state holding current required to maintain the relay in the energized state with $V_{\text{hold}} = 6\text{ V}$ is:
$$
I_{\text{hold}} = \frac{V_{\text{hold}}}{R_{\text{coil}}} = \frac{6\text{ V}}{51\text{ }\Omega} \approx 117.6\text{ mA}
$$

To drop the remaining $6\text{ V}$ across the series resistor $R25$ / $R26$ / $R27$ at steady state:
$$
R = \frac{V_{\text{CC}} - V_{\text{hold}}}{I_{\text{hold}}} = \frac{12\text{ V} - 6\text{ V}}{117.6\text{ mA}} \approx 51\text{ }\Omega
$$

To determine the required capacitance $C14$ / $C15$ / $C16$ that dictates the time constant for the voltage transition, we utilize the standard exponential charging formula for the RC network.

Base formula:
$$
V_C(t) = V_{\text{max}} \cdot \left(1 - e^{-\frac{t}{R_{eq} \cdot C}}\right)
$$

Inverting the formula to find $C14$ / $C15$ / $C16$:
$$
C = -\frac{t}{R_{\text{eq}} \cdot \ln\left(1 - \frac{V_C(t)}{V_{\text{max}}}\right)}
$$

Considering the equivalent resistance $R_{\text{eq}} = R \parallel R_{\text{coil}} = 25.5\text{ }\Omega$, $t = 0.1\text{ s}$, $V_f = 6\text{ V}$, and the target voltage $V_C(t) = 4.2\text{ V}$ corresponding to a 70% threshold:
$$
C = -\frac{0.1\text{ s}}{25.5\space \Omega \cdot \ln\left(1 - \frac{4.2\text{ V}}{6\text{ V}}\right)} = \frac{0.1}{25.5 \cdot 1.2039} \approx 3300\text{ }\mu\text{F}
$$

### 3.3 Flywheel protection

As recommended by the relay manufacturer, using only a standard freewheeling diode delays the release time of the relay contacts. Adding a Zener diode in series solves this by increasing the discharge voltage, allowing the coil current to decay much faster while protecting the driver from overvoltage.

**Flywheel Diodes (D6, D8, D10) required specs:** Fast recovery or standard silicon diode with a repetitive peak reverse voltage $V_{RRM}$ at least 10 times the coil rated voltage ($\ge 120\text{ V}$), and a forward current $I_F$ exceeding the nominal coil current ($\ge 300\text{ mA}$).

**Zener Diodes (D7, D9, D11) required specs:** The manufacturer recommends a Zener voltage approximately 3 times the rated coil voltage to ensure optimal release time and energy dissipation:
$$
V_{Zener} \approx 3 \cdot V_{\text{coil}} = 3 \cdot 12\text{ V} = 36\text{ V}
$$
