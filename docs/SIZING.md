# Precharge controller's sizing file

## Index
1. [LV Power supply]()
2. [LV Logic]()
3. [LV Relay drive]()
4. [HV Sense]()
5. [HV Power]()
6. [Connectors]()

## 1. LV Power Supply

### 1.1 Reverse polarity & Gate protection (lv_supply and shutdown lines)

A P-Channel MOSFET (Q1) is used instead of a standard series diode to prevent reverse polarity damage. This design choice eliminates the typical $0.7\text{ V}$ forward voltage drop, maximizing efficiency and minimizing heat dissipation. 

PMOS (Q1) required specs:
- **Max $V_{DS} \leq -20\text{ V}$**
- **Low $R_{DS(on)}$**
- **Continuous current $I_{D} \geq 10\text{ mA}$** (for lv_supply line)
- **Continuous current $I_{D} \geq 250\text{ mA}$** (for shutdown line)

Chosen Vishay's Si3437DV specs:
- $V_{DS} = -30\text{ V}$
- $I_{D} = -4.8\text{ A}$
- $V_{GS(max)} = \pm 20\text{ V}$

The gate is pulled to ground via $R1 = 10\text{ k}\Omega$, ensuring the PMOS is fully enhanced (turned on) when a positive voltage is correctly applied. 
To protect the MOSFET from voltage transients, a Zener diode (D1 / D3) is placed between the gate and source. Since the SI3437DV has a max $V_{GS}$ of $20\text{ V}$, a $15\text{ V}$ Zener clamps the voltage safely below the destruction limit.

### 1.2 Transient overvoltage protection (lv_supply and shutdown lines)

To protect the downstream linear regulator, logics chips, and relay coils from high-energy transients, a diode clamping stage is implemented.

TVS Diode (D2 / D13) required specs:
- **Standoff voltage $\geq 15\text{ V}$:** To avoid clamping normal battery fluctuations.
- **Clamping voltage $\leq 35\text{ V}$:** To prevent exceeding the linear regulator's maximum input voltage.

Choosen D2 / D13 specs:
- $V_{c} = 28\text{ V}$
- max $I_{s} = 2\text{ A}$

$R28$ and $R29$ act as a series current-limiting resistor. During a severe overvoltage spike (e.g., $V_{spike} = 60\text{ V}$), the TVS diode rapidly enters avalanche breakdown and clamps the line at $28\text{ V}$. The surge current absorbed by the TVS is limited by $R28$ and $R29$ according to the formula:
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

### 1.3 Linear voltage stabilizer (lv_supply line)

The discrete logic block requires a highly stable voltage, due to the logic nature of the load.

Voltage Regulator (IC1) required specs:
- **High input voltage tolerance**
- **Output voltage $= 12\text{ V}$**

Choosen STMicroelectronics's L7812ABV specs:
- Max input voltage = $35\text{ V}$
- Output voltage = $12\text{ V}$
- Output current up to $1.5\text{ A}$

Following the manufacturer's datasheet recommendations two filtering capacitor are placed in parallel with input and output lines:
- $C1 = 0.33\space \mu\text{F}$ (input)
- $C2 = 0.1\space \mu\text{F}$ (output)

### 1.4 Bypass filters (shutdown line)

Because the +12V_SHDW line supplies inductive loads (relay and contactor coils), switching events generate sharp current steps and voltage sags. A dual-stage bypass filter is utilized:

- **High-Frequency Bypass (C3):** A $0.1\space \mu\text{F}$ multilayer ceramic capacitor (MLCC) is placed locally to absorb high-frequency EMI.
- **Inductive Transient Bypass (C4):** To supply sudden transient currents when multiple relay coils switch simultaneously without causing severe voltage droops ($\Delta V$), a bulk capacitor is required. 

According to the Omron G9KB-1A12V relay datasheet, the nominal coil resistance is $R_{c} = 51\space \Omega$. To optimize power consumption and thermal performance, the system applies a holding voltage of $V_{hold} = 6\text{ V}$ (50% of the rated 12V) in relay drive block.

The holding current drawn by a single relay coil is:
$$
I_{hold} = \frac{V_{hold}}{R_{coil}} = \frac{6\text{ V}}{51\space \Omega} \approx 117\text{ mA}
$$

Considering a maximum of 2 relay coils active simultaneously under holding conditions, the total load current is:
$$
I_{load} = 2 \times 117\text{ mA} = 234\text{ mA}
$$

Considering a maximum $\Delta V = 0.5\text{ V}$ on the line during a switching event lasting $\Delta t = 0.5\text{ ms}$, the required capacitance is calculated as:
$$
C_4 \ge \frac{I_{load} \cdot \Delta t}{\Delta V} = \frac{234\text{ mA} \cdot 0.5\text{ ms}}{0.5\text{ V}} = 234\space \mu\text{F}
$$


## 2. LV Logic

### 2.1 Enable level shifter

The level is shifted by connecting the ENABLE input line to the gate of an NMOS. The gate at high input, connects the capacitor directly to ground discharging it immediately.

NMOS (Q4) required specs:
- **Max $V_{DS} \geq 20V$**
- **Max $I_{DS} \geq 120\text{ mA}$**
- **Max $V_{GS(th)} \leq 3\text{ V}$**

Choosen Onsemi's 2N7002K specs:
- Inner diode protection
- $V_{DS} = 60\text{ V}$
- $I_{DS} = 800\text{ mA}$
- $V_{GS(th)} = 1\text{ V}$
- max $R_{DS(on)} = 4\space \Omega$ (won't interfere with C6's discharge time)

### 2.2 Enable timer

Schmitt trigger (IC3) required specs are:
- **$12\text{ V}$ power domain:** In order to power it with +12_LOGIC line
- **Hysteresis voltage ($V_h$):** It's a defined separation between the positive ($V_p$) and negative ($V_n$) thresholds, in order to prevent oscillation if the input is around the triggering spot.
- **Unlimited input rise and fall times:** The trigger must be design to tolerate extremely slow voltage ramps generated by the capacitor charging, the minimum spec is around 50ms.
- **Low input current:** To ensure correct operations, the trigger must ensure minimal current draw from the input pin, in order to preserve the $\tau$ constant of the RC network.
- **Low current draw**
- **High noise margin**

Choosen Texas Instrument's CD40106B specs:
- $3\text{ V} \leq V_{DD} \leq 18\text{ V}$
- Infinite input rise time tolerance
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
- $C6 = 1\space \mu\text{F}$ (not electrolytic because they have high tolerances for a timer)
- $R6 = 56\text{ k}\Omega$ (standard value)

R19 is used as discharge resistance, in order to prevent a short circuit and protect the NMOS (Q4). In this case $I_{peak} = \frac{12\text{ V}}{R19}$. Choosing a resistor of $R19 = 100\space \Omega$, we are sure to preserve the NMOS and to not interfere with R6 and ensuring a stable down signal when enable is up. The discharge time with $\tau = 100\space \Omega \cdot 1\space \mu\text{F}$ is around $0.1\text{ ms}$, so it won't affect the timer functioning.

The timer wont be precise to the millisecond, due to component tolerances.

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
Considering the maximum input current of the chip $I_{max} = 10\text{ mA}$ and the internal clamping voltage $V_{c} = 12.5\text{ V}$:
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

All the logic chips are choosen with the M96 SMD package ensuring higher fatigue failure and a reduction to parasitic inductances. All the chips have a decoupling capacitor of $0.1\space \mu\text{F}$.


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

**Gate Pull-Down Resistors (R11, R12, R13):** to mitigate fluctuations when CMD_XXX_IN is down.

**Gate Series Resistors (R14, R15, R16):** Sized to limit the gate current and prevent excessive current spikes. Choosing $R_{gate} = 1\text{ k}\Omega$, the peak gate current from a 12V control signal is limited to:
$$
I_{gate} = \frac{12\text{ V}}{1\text{ k}\Omega} = 12\text{ mA}
$$
This safely protects the upstream logic outputs.

### 3.2 Holding voltage

As Omron G9KB relay's datasheet recommendations for optimizing power consumption and coil preservation, a power-saving holding circuit is implemented using a parallel RC network ($R25$ / $R26$ / $R27$ and $C14$ / $C15$ / $C16$) placed in series with each relay coil. This circuit applies full rated voltage ($12\text{ V}$) instantaneously for reliable pull-in, and then exponentially reduces the voltage to a safe holding level ($V_{hold} = 6\text{ V}$, corresponding to 50% of the rated voltage) after an initial actuation window of $t = 0.1\text{ s}$.

Given the relay coil resistance $R_{coil} = 51\space \Omega$, the steady-state holding current required to maintain the relay in the energized state with $V_{hold} = 6\text{ V}$ is:
$$
I_{hold} = \frac{V_{hold}}{R_{coil}} = \frac{6\text{ V}}{51\space \Omega} \approx 117.6\text{ mA}
$$

To drop the remaining $6\text{ V}$ across the series resistor $R25$ / $R26$ / $R27$ at steady state:
$$
R = \frac{V_{CC} - V_{hold}}{I_{hold}} = \frac{12\text{ V} - 6\text{ V}}{117.6\text{ mA}} \approx 51\space \Omega
$$

To determine the required capacitance $C14$ / $C15$ / $C16$ that dictates the time constant for the voltage transition, we utilize the standard exponential charging formula for the RC network.

Base formula:
$$
V_C(t) = V_{max} \cdot \left(1 - e^{-\frac{t}{R_{eq} \cdot C}}\right)
$$

Inverting the formula to find $C14$ / $C15$ / $C16$:
$$
C = -\frac{t}{R_{eq} \cdot \ln\left(1 - \frac{V_C(t)}{V_{max}}\right)}
$$

Considering the equivalent resistance $R_{eq} = R \parallel R_{coil} = 25.5\space \Omega$, $t = 0.1\text{ s}$, $V_f = 6\text{ V}$, and the target voltage $V_C(t) = 4.2\text{ V}$ corresponding to a 70% threshold:
$$
C = -\frac{0.1\text{ s}}{25.5\space \Omega \cdot \ln\left(1 - \frac{4.2\text{ V}}{6\text{ V}}\right)} = \frac{0.1}{25.5 \cdot 1.2039} \approx 3300\space \mu\text{F}
$$

### 3.3 Flywheel protection

As recommended by the relay manufacturer, using only a standard freewheeling diode delays the release time of the relay contacts. Adding a Zener diode in series solves this by increasing the discharge voltage, allowing the coil current to decay much faster while protecting the driver from overvoltage.

**Flywheel Diodes (D6, D8, D10) required specs:** Fast recovery or standard silicon diode with a repetitive peak reverse voltage $V_{RRM}$ at least 10 times the coil rated voltage ($\ge 120\text{ V}$), and a forward current $I_F$ exceeding the nominal coil current ($\ge 300\text{ mA}$).

**Zener Diodes (D7, D9, D11) required specs:** The manufacturer recommends a Zener voltage approximately 3 times the rated coil voltage to ensure optimal release time and energy dissipation:
$$
V_{Zener} \approx 3 \cdot V_{coil} = 3 \cdot 12\text{ V} = 36\text{ V}
$$


## 4. HV Sense

### 4.1 DC_LINK voltage partitioner (100%)

The DC_LINK voltage partitioner scales the high-voltage of the inverter down to a safe, scaled sensing voltage compatible with the comparator inputs.

**Required specifications:** 
- Maximum inverter voltage: $V_{TS max} = 600\text{ VDC}$.
- Target maximum sense voltage: $V_{sense max} \approx 12.0\text{ V}$ (matching the isolated logic supply rail).
- High voltage resistor string to limit power dissipation and comply with component voltage ratings.

To distribute the voltage stress safely across multiple surface-mount resistors (preventing arc-over and exceeding maximum working voltage per resistor), three resistors in series are used:
$$
R3 = R4 = R5 = 330\text{ k}\Omega \quad (\text{Total series } R_{series} = 990\text{ k}\Omega)
$$
Using a paralel resistor $R7 = 20\text{ k}\Omega$, the voltage division ratio is:
$$
V_{out} = V_{in} \cdot \left(\frac{R7}{R_{series} + R7}\right) = 600\text{ V} \cdot \left(\frac{20\text{ k}\Omega}{990\text{ k}\Omega + 20\text{ k}\Omega}\right) \approx 11.88\text{ V}
$$
This safely maps $600\text{ V}$ to just under $12\text{ V}$.

### 4.2 TS_POS voltage partitioner (95%)

The TS_POS voltage partitioner scales the battery-side positive line so that it can be directly compared against the scaled DC_LINK voltage. Precharge completion is evaluated by monitoring when the inverter-side voltage rises to reach $95\%$ of the battery-side voltage.

To properly establish the comparison threshold, the attenuation of this divider is adjusted. Using identical high-side resistors:
$$
R20 = R21 = R22 = 330\text{ k}\Omega \quad (\text{Total } R_{series} = 990\text{ k}\Omega)
$$
The bottom resistor $R23$ is calculated to scale the battery voltage reference:
$$
R23 = 21\text{ k}\Omega
$$
This ensures accurate level matching for the dual-comparator evaluation stage.

### 4.3 RC low pass filter (DC_LINK and TS_POS lines)

To eliminate high-frequency switching noise and electromagnetic interference (EMI) originating from the inverter, an RC low-pass filter is implemented on both sensing lines.

The Thevenin resistance seen by the filter capacitor is:
$$
R_{th} = \frac{R_{series} \cdot R7}{R_{series} + R7} \approx \frac{R_{series} \cdot R23}{R_{series} + R23} \approx 20\text{ k}\Omega
$$
Targeting a cutoff frequency of $f_c \approx 10\text{ Hz}$:
$$
C = \frac{1}{2 \pi \cdot R_{th} \cdot f_c} = \frac{1}{2 \pi \cdot 20\text{ k}\Omega \cdot 10\text{ Hz}} \approx 0.80\space \mu\text{F}
$$

This result leads to the use of a standard $1\space \mu\text{F}$ capacitor.

### 4.4 Transient overvoltage protection (DC_LINK and TS_POS lines)

To protect the sensitive comparator inputs from voltage spikes, clamping diodes are placed on the filtered sense lines.

$12\text{ V}$ Zener diodes are selected to clamp any voltage exceeding the comparator's internal isolated supply line ($V_{CC} = 12\text{ V}$), safely shunting excessive transient energy to ground and protecting the chip.

### 4.5 Comparation

Comparator (IC4) required specs:
- **Supply voltage range $\ge 12\text{ V}$:** To operate reliably from the isolated logic rail.
- **Low input offset voltage and high precision:** To accurately resolve small differential voltage thresholds during precharge completion.
- **Open-collector / open-drain output:** To interface seamlessly with pull-up networks and galvanic isolation stages.

Chosen Texas Instruments LM393BIDR specs:
- Dual differential comparator configuration.
- Wide supply voltage range up to $36\text{ V}$.
- Low input offset voltage ($\pm 2.0\text{ mV}$ typ.).
- Low input bias current and open-collector outputs suitable for robust digital level translation.

### 4.6 Galvanic isolation

#### 4.6.A Optoisolator for CHARGE_REQUIRED signal

Optoisolator (Q7) required specs:
- **Galvanic isolation rating $\ge 1\text{ kV}$:** To ensure complete electrical safety between the high-voltage Traction System and low-voltage control domain.
- **Current transfer ratio (CTR):** High enough to guarantee reliable transistor saturation on the receiving side under low LED drive currents.

Chosen American Bright's APC-81701-SL specs:
- High isolation voltage between input and output ($AC 5,000\text{ Vrms}$).
- High Collector-Emitter voltage ($V_{CEO} \ge 35\text{ V}$).
- Fast switching response tailored for digital status signaling.

Current-limiting resistor R9 is sized to establish the required forward LED current. With an isolated logic supply of $12\text{ V}$ and a standard LED forward voltage $V_F \approx 1.2\text{ V}$:
$$
R9 = \frac{V_{CC} - V_F}{I_F} \approx 4.7\text{ k}\Omega
$$

#### 4.6.B DCDC converter for low voltage supply

DC-DC Converter (PS1) required specs:
- **Input voltage range:** Compatible with the main low-voltage logic line ($12\text{ V}$).
- **Output voltage:** Regulated $12\text{ V}$ output to power the isolated sensing and comparison circuitry.
- **Isolation rating:** Reinforced galvanic isolation to separate high-voltage reference grounds from low-voltage electronics.

Chosen Cui's DEP1-12-512-M specs:
- Designed specifically for isolated low-power auxiliary lines in electric vehicle architectures.

To filter out switching noise generated by the isolated DC-DC converter (PS1), local decoupling capacitors ($C10$, $C11$, $C12$) are placed across the input and output power lines. 

$C10$ and $C12$ are placed to decouple the DCDC converter (PS1) from both lines. $C11$ is placed to decouple from the isolated line the comparator.

$C10 = C11 = C12 = 0.1\space\mu\text{F}$

The optoisolator (Q7) doesn't need a decoupling capacitor due to its LED nature of the load.


## 5. HV Power

### 5.1 Precharge resistor

Precharge resistor (R17) required specs:
- **Maximum Voltage $\ge 600\text{ V}_{DC}$:** To withstand the maximum Traction System voltage.
- **Resistance value sizing:** Must limit the maximum precharge current to strictly less than $1\text{ A}$, including resistor manufacturing tolerance.
- **Thermal and Energy ratings:** Must absorb the energy stored in the DC-link capacitor ($C_{DC} = 500\space \mu\text{F}$ nominal, up to $600\space \mu\text{F}$ with tolerance).

To ensure the precharge current never exceeds $1\text{ A}$ at the maximum voltage ($V_{max} = 600\text{ V}$) and considering a resistor tolerance of $\pm 5\%$:
$$
R_{min} = \frac{V_{max}}{I_{max}} = \frac{600\text{ V}}{1\text{ A}} = 600\space \Omega
$$
Choosing a nominal resistance $R_{17} = 1000\space \Omega$:
$$
I_{peak} = \frac{V_{max}}{R_{17} \cdot (1 - \text{tol})} = \frac{600\text{ V}}{1000\space \Omega \cdot 0.95} \approx 0.632\text{ A} < 1\text{ A}
$$

The worst-case time constant $\tau_{max}$ considering maximum resistance ($R_{max} = 1050\space \Omega$) and maximum DC-link capacitance with tolerance ($C_{max} = 600\space \mu\text{F}$):
$$
\tau_{max} = R_{max} \cdot C_{max} = 1050\space \Omega \cdot 600\space \mu\text{F} = 0.63\text{ s}
$$

The time required to reach $95\%$ of the Traction System voltage:
$$
t_{95} = \tau_{max} \cdot \ln(20) = 0.63\text{ s} \cdot 2.9957 \approx 1.89\text{ s}
$$
This is well within the required $10\text{ s}$ limit specified by FSAE rules.

Energy dissipated by $R17$ during a precharge cycle (equal to the energy stored in the DC-link capacitor):
$$
E = \frac{1}{2} \cdot C_{max} \cdot V_{max}^2 = \frac{1}{2} \cdot 600\space \mu\text{F} \cdot (600\text{ V})^2 = 108\text{ J}
$$

Peak pulse power at the start of precharge:
$$
P_{peak} = \frac{V_{max}^2}{R_{min}} = \frac{(600\text{ V})^2}{950\space \Omega} \approx 378.9\text{ W}
$$

Choosen TE-Automotive's HSA501K0J specs:
- $R = 1\text{ k}\Omega$
- $5\%$ tolerance
- $P_{max} = 50\text{ W}$
- $V_{max} = 1.25\text{ kV}$
- Chassis mount

The Precharge resistor needs to be fixed outside the board on the chassis.

### 5.2 Relays

Relays (U2, U3, U4) required specs:
- **Contact Voltage Rating $\ge 600\text{ VDC}$:** To safely switch and isolate the high-voltage Traction System bus.
- **Continuous Current Rating $\ge 5\text{ A}$:** To handle the maximum continuous current through the main AIR contactors after precharge.
- **Coil Voltage and Drive Compatibility:** Must operate reliably from the $12\text{ V}$ shutdown supply rail and integrate with the shutdown and holding voltage architecture.

Chosen Omron G9KB-1A DC12 specs:
- Designed specifically for high-voltage DC switching in electric vehicles.
- Rated for switching high DC voltages and currents up to $600\text{ VDC}$ / $50\text{ A}$.
- Low contact resistance ensuring minimal voltage drop and thermal dissipation at continuous currents.
- Coil resistance $R_{coil} = 51\space \Omega$, matching the power-saving holding circuit discussed in lv_relay_drive block.


## 6. Connectors

### 6.1 Low Voltage Connector (J1)

Low Voltage Connector ($J1$) required specs:
- **Automotive Environment Compliance:** Must withstand high vibration, mechanical shock, moisture, and thermal cycles typical of a Formula SAE racing vehicle.
- **Current and Voltage Rating:** Fully compatible with the unregulated low-voltage auxiliary supply ($16-24\text{ VDC}$) and the $12\text{ VDC}$ shutdown safety circuit.
- **Keying and Positive Latching:** Must feature a robust mechanical locking mechanism to prevent accidental disconnection under severe vehicle vibrations.

Chosen TE Connectivity / Deutsch's DTF13-4P-G003 specs:
- 4-pin 90-degree flangeless receptacle (DT series) designed for harsh environment automotive applications.
- Gold-plated copper alloy pins ensuring superior corrosion resistance and highly reliable electrical contact.
- Robust PA46 GF30 housing rated for wide operating temperature ranges and mechanical stress.

### 6.2 High Voltage Terminal Blocks (J2, J3, J4)

High Voltage Terminal Blocks ($J2, J3, J4$) required specs:
- **Voltage Rating $\ge 600\text{ VDC}$:** To safely handle Traction System potentials and prevent flashover or breakdown.
- **Current Rating:** Capable of carrying continuous currents (up to $5\text{ A}$ through main AIRs) and withstanding transient precharge currents.

Chosen Weidmüller's PCF 7.50/03/180 3.5SN OR BX (9500630000) specs:
- 3-pole PCB terminal block with a $7.50\text{ mm}$ pitch.
- Flat-blade connection supporting $6.3\text{ mm}$ and $2.8\text{ mm}$ spade connectors with a $180^\circ$ conductor outlet direction.
- Maximum ratings: $1000\text{ V}$ and $24\text{ A}$.

The central pole is left disconnected in order to ensure maximum separation between potentials.
