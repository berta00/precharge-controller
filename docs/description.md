# Precharge-controller's description file

## Index
1. [Architecture overview]()
2. [Power supply & Current Budget]()
3. [Low voltage logic]()
4. [Relay drivers]()
5. [High voltage sensing & Galvanic isolation]()
6. [Precharge power circuit sizing]()
7. [External interfacing of the board]()

## 1. Architecture overview

This document details the design of the hardware-only precharge controller for a Formula SAE electric vehicle. The system architecture is physically and galvanically divided into two distinct domains: low voltage (LV) and high voltage (HV).

The low voltage domain features: 
- **Power supply** block: Stabilizes the unregulated LV input and protects the board from reverse polarity and transient overvoltages.
- **Logic** block: Uses discrete CMOS logic to evaluate safety conditions, implement an asymmetric 50ms timer, and provide a fail-safe completion memory without programmable devices.
- **Relay drive** block: Controls the three main relay coils using the safety shutdown supply and low-side NMOS drivers.


The high voltage domain features:
- **Sensing** block: Utilizes dual high-impedance voltage dividers, an isolated DC/DC converter, and an analog comparator to safely confirm the 95% DC-link charge state.
- **block**: Contains the external accumulator insulation relays (AIR+, AIR-), the precharge relay, and the precharge power resistor.


## 2. Power supply & Current Budget

This section details the power distribution, protection mechanisms, and thermal verification of the internal power supply block, ensuring operation across the specified 16-26V DC main supply and 12V shutdown supply ranges.

### 2.1 Protection circuit and regulation

Both LV_LOGIC and LV_SHUTDOWN lines are protected against: 
- Reverse polarity: using a PMOS transistor configured with a Zener diode, ensuring $V_{GS}$ protection to the PMOS.
- Transient overvoltage: using a TVS diode, chosen because of its ability to withstand high voltage spikes.

LV_LOGIC line is stabilized by a L7812ABV linear voltage stabilizer to generate a linear +12V supply. 
LV_SHDW already comes stabilized, but it needs bypass filters in order to protect the line by the EMI of relay coils.

### 2.2 Current budget

The current consumption is divided between two independent supplies:
- **+12_LOGIC**:  The logic chips draws negligible quiescent current (under 1 mA). The primary load is the DEP1-12-S12-M DC/DC converter, which supplies an LM393B comparator and a APC817D1 optocoupler used to galvanically isolate low and high domain. The total maximum continous current draw is around $4.0\space mA$

$$
I_{max} = I_{IC1} + I_{IC2} + I_{IC3} + I_{IC5} + I_{PS1}
$$
$$
I_{PS1} = \frac{I_{DCDCout}}{\text{DCDC efficiency}} = \frac{\frac{12\space V - V_{Q7}}{R9} + I_{IC4}}{\text{DCDC efficiency}}
$$
$$
I_{max} \approx 4.3\space mA + 5\space \mu A + 5\space \mu A + 5\space \mu A +  \frac{\frac{12\space V - 1.24\space V}{4.7\space k\Omega} + 0.8\space mA}{0.80} \approx 8 \space mA
$$

- **+12_SHDW**: The G9KB-1A12V relays are powered only by the shutdown circuit. Using the RC voltage holder, the consumption per coil is reduced approximately to $117\space mA$ at $6\space V$ (according to datasheet). A maximum of two relays are energized at the same time. Therefore, the maximum, continuous current draw is around $234\space mA$.

$$
I_{max} = I_{U2} + I_{U3} \approx 117\space mA + 117\space mA= 234\space mA
$$

### 2.3 Regulator power dissipation

The L7812ABV regulator steps down the 16-24V input to stable 12V, because the relays are powered by the shutdown line, the low voltage supplies only the logic domain. 
The worst-case continous power disipation is calculated as:

$$ P_{dis} = (V_{in \space max} - V_{out}) \cdot I_{logic \space max} + V_{in \space max} \cdot I_{IC1} $$
$$ P_{dis} \approx (24 \space V - 12 \space V) \cdot 4 \space mA + 24\space V \cdot 4.3\space mA \approx 151 \space mW $$

This result is not concerning for heat dissipation, therefore IC1 does not require additional cooling components

## 3. Low voltage logic

This section details the logic circuit responsible for controlling the relay drive block. Based on two primary inputs: an external ENABLE signal and a CHARGE_REQUIRED signal from the high-voltage sensing block. The logic drives three distinct outputs: CMD_AIR_PRE (precharge relay), CMD_AIR_POS (positive AIR), and CMD_AIR_NEG (negative AIR).

### 3.1 ENABLE Level Shifter and Timer

The external ENABLE signal is level-shifted by an N-MOSFET to safely match the internal 12V CMOS logic domain. The shifted line is connected to a 50 ms timer implemented using an RC network and a Schmitt trigger inverter (IC3). At board startup, if the enable signal is low, the capacitor charges towards 12V. If the enable signal is high, the capacitor is held discharged to ground; the inverter reads a low input and its output goes high. During operation, if the enable signal goes low, the capacitor starts to charge. Once the $V_p = 7.1\space V$ threshold of the Schmitt trigger is reached, the inverter interprets the input as a logic high, which results in a delayed falling edge of the final output.
Below a simulation of this circuit done with LTspice, available [here](/hardware/sim/enabe_timer.asc).



### 3.2 SR latch and completion memory

### 3.3 Loss of supply protection
