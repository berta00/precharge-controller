# Precharge-controller's description file

## Table of Contents
- [Precharge-controller's description file](#precharge-controllers-description-file)
  - [Table of Contents](#table-of-contents)
  - [1. Architecture overview](#1-architecture-overview)
  - [2. Power supply \& Current Budget](#2-power-supply--current-budget)
    - [2.1 Protection circuit and regulation](#21-protection-circuit-and-regulation)
    - [2.2 Current budget](#22-current-budget)
    - [2.3 Regulator power dissipation](#23-regulator-power-dissipation)
  - [3. Low voltage logic](#3-low-voltage-logic)
    - [3.1 ENABLE Level Shifter and Timer](#31-enable-level-shifter-and-timer)
    - [3.2 SR latch and completion memory](#32-sr-latch-and-completion-memory)
    - [3.3 Loss of supply protection](#33-loss-of-supply-protection)
  - [4. Low voltage relay drive](#4-low-voltage-relay-drive)
    - [4.1 Driver](#41-driver)
    - [4.2 Coil holding voltage](#42-coil-holding-voltage)
    - [4.3 Freewheeling](#43-freewheeling)
  - [5. High voltage sensing](#5-high-voltage-sensing)
    - [5.1 DC\_LINK mesurement (100%)](#51-dc_link-mesurement-100)
    - [5.2 TS\_POS mesurement (95%)](#52-ts_pos-mesurement-95)
    - [5.3 Galvanic insulation](#53-galvanic-insulation)
    - [5.4 Comparation](#54-comparation)
  - [6. High voltage power](#6-high-voltage-power)


## 1. Architecture overview

This document details the design of the hardware-only precharge controller for a Formula SAE electric vehicle. The system architecture is physically and galvanically divided into two distinct domains: low voltage (LV) and high voltage (HV).

The low voltage domain features: 
- **Power supply** block: Stabilizes the unregulated LV input and protects the board from reverse polarity and transient overvoltages.
- **Logic** block: Uses discrete CMOS logic to evaluate safety conditions, implement an asymmetric 50ms timer, and provide a fail-safe completion memory without programmable devices.
- **Relay drive** block: Controls the three main relay coils using the safety shutdown supply and low-side NMOS drivers.


The high voltage domain features:
- **Sensing** block: Utilizes dual high-impedance voltage dividers, an isolated DC/DC converter, and an analog comparator to safely confirm the 95% DC-link charge state.
- **block**: Contains the external accumulator insulation relays (AIR+, AIR-), the precharge relay, and the precharge power resistor.

<p align="center">
    <img src="../docs/img/root_sheet.png" style="width: 700px; float: center;" >
</p>


## 2. Power supply & Current Budget

This section details the power distribution, protection mechanisms, and thermal verification of the internal power supply block, ensuring operation across the specified 16-26V DC main supply and 12V shutdown supply ranges.

### 2.1 Protection circuit and regulation

Both LV_LOGIC and LV_SHUTDOWN lines are protected against: 
- **Reverse polarity:** using a PMOS transistor configured with a Zener diode, ensuring $V_{GS}$ protection to the PMOS.
- **Transient overvoltage:** using a TVS diode, chosen because of its ability to withstand high voltage spikes.

LV_LOGIC line is stabilized by a L7812ABV linear voltage stabilizer to generate a linear +12V supply. 
LV_SHDW already comes stabilized, but it needs bypass filters in order to protect the line by the EMI of relay coils.

<p align="center">
    <img src="../docs/img/lv_power_supply_1_sheet.png" style="width: 1000px; float: center;" >
    <img src="../docs/img/lv_power_supply_2_sheet.png" style="width: 1000px; float: center;" >
</p>

### 2.2 Current budget

The current consumption is divided between two independent supplies:
- **+12_LOGIC:**  The logic chips draws negligible quiescent current (under 1 mA). The primary loads are the linear regulator and the DEP1-12-S12-M DC/DC converter (with an efficienzy of 80%), which supplies a LM393B comparator and a APC817D1 optocoupler used to galvanically isolate low and high domain. The total maximum continous current draw on the linear regulator is around $4.0\space mA$.

$$
I_{max} = I_{IC1} + I_{IC2} + I_{IC3} + I_{IC5} + I_{PS1}
$$
$$
I_{PS1} = \frac{I_{DCDCout}}{\text{DCDC efficiency}} = \frac{\frac{12\space V - V_{Q7}}{R9} + I_{IC4}}{\text{0.80}}
$$
$$
I_{max} \approx 4.3\text{ mA} + 5\space \mu\text{A} + 5\space \mu\text{A} + 5\space \mu\text{A} +  \frac{\frac{12\text{ V} - 1.2\text{ V}}{4.7\text{ k}\Omega} + 0.8\text{ mA}}{0.80} \approx 8 \text{ mA}
$$

- **+12_SHDW:** The three G9KB-1A12V relays are powered only by the shutdown circuit. Using an RC voltage holder, the consumption per coil is reduced approximately to $117\text{ mA}$ at $6\text{ V}$ (according to datasheet). A maximum of two relays are energized at the same time. Therefore, the maximum, continuous current draw is around $234\text{ mA}$.

$$
I_{max} = I_{U2} + I_{U3} \approx 117\text{ mA} + 117\text{ mA}= 234\text{ mA}
$$

### 2.3 Regulator power dissipation

The L7812ABV regulator steps down the 16-24V input to stable 12V, because the relays are powered by the shutdown line, the low voltage supplies only the logic domain. 
The worst-case continous power disipation is calculated as:

$$ P_{dis} = (V_{in \space max} - V_{out}) \cdot I_{logic \space max} + V_{in \space max} \cdot I_{IC1} $$
$$ P_{dis} \approx (24 \text{ V} - 12 \text{ V}) \cdot 4 \text{ mA} + 24\text{ V} \cdot 4.3\text{ mA} \approx 151 \text{ mW} $$

This result is not concerning for heat dissipation, therefore IC1 does not require additional cooling components

## 3. Low voltage logic

This section details the logic circuit responsible of controlling the relay drive block, based on two primary inputs: an external ENABLE signal and a CHARGE_REQUIRED signal from the high-voltage sensing block. The logic drives three distinct outputs: CMD_AIR_PRE (precharge relay), CMD_AIR_POS (positive AIR), and CMD_AIR_NEG (negative AIR).

<p align="center">
    <img src="../docs/img/final_logic_diagram.jpeg" style="width: 900px; float: center;" >
</p>

### 3.1 ENABLE Level Shifter and Timer

The external ENABLE signal is level-shifted by an N-MOSFET to safely match the internal 12V CMOS logic domain. The shifted line is connected to a 50 ms timer implemented using an RC network and a Schmitt trigger inverter (IC3). At board startup, if the enable signal is low, the capacitor charges towards 12V imediatly. If the enable signal is high, the capacitor is held discharged to ground; the inverter reads a low input and its output goes high. During operation, if the enable signal goes low, the capacitor starts to charge. Once the $V_p = 7.1\space V$ threshold of the Schmitt trigger is reached, the inverter interprets the input as a logic high, which results in a delayed falling edge of the final output.
A simulation of this circuit, done with LTspice, is available [here](/hardware/sim/enabe_timer.asc).

<p align="center">
    <img src="../docs/img/lv_logic_1_sheet.png" style="width: 400px; float: center;" >
</p>

### 3.2 SR latch and completion memory

The core sequencing and state retention are managed by a discrete set-reset (SR) latch architecture built using cross-coupled logic gates. 
- **Sequence Initiation:** When the system is powered on, the shutdown supply is verified, and the ENABLE signal is asserted high, the latch initiates the precharge sequence by closing AIR− and the precharge relay, while keeping positive AIR open.
- **Completion Memory:** Once the high-voltage sensing circuit determines that the DC-link voltage has reached $95\%$ of the traction system potential, the CHARGE_REQUIRED signal goes down. This event triggers the completion memory logic, switching the SR latch state to decouple the precharge resistor branch and command the closure of the positive AIR contactor (CMD_AIR_POS), ensuring the system remains latched in the fully operational state until a shutdown command occurs.

### 3.3 Loss of supply protection

To comply with safety requirements regarding power loss and fault contingencies, a monitoring and reset and gate is integrated into the logic domain:
- **Shutdown and Low-Voltage Supervision:** The circuit continuously tracks the integrity of both the $12\text{ V}$ shutdown safety supply and the unregulated low-voltage lines.
- **Fail-Safe Response:** If the shutdown supply or the low-voltage auxiliary supply drops below operating thresholds, or if the ENABLE signal is removed for more than $50\text{ ms}$, the protection logic overrides the SR latch state. This forces all relay control outputs (CMD_AIR_PRE, CMD_AIR_POS, CMD_AIR_NEG) to actively open within a maximum of $250\text{ ms}$, safely isolating the high-voltage Traction System from the inverter.

<p align="center">
    <img src="../docs/img/lv_logic_3_sheet.png" style="width: 700px; float: center;" >
</p>

<TODO calculate precise opening time>


## 4. Low voltage relay drive

This section details the low-voltage relay driver circuitry responsible for translating control signals from the logic domain into higher current coils supplied with the shutdown line.

### 4.1 Driver

The relay driver stage interfaces the $12\text{ V}$ outputs of the low-voltage logic block with the higher current demands of the relay coils. Each driver command channel uses a low-side switching transistor configuration to complete the coil circuit to ground. The input stage includes current-limiting and pull-down protection resistors to prevent floating states and ensure fail-safe behavior if a control line is accidentally disconnected.

### 4.2 Coil holding voltage

To optimize thermal performance and reduce continuous power consumption from the auxiliary supply during long endurance events, the relay driver incorporates an optimized coil drive architecture matching the Omron G9KB-1A DC12 specifications.

### 4.3 Freewheeling

When an electromechanical relay coil is switched off, the sudden collapse of the magnetic field induces a severe reverse-voltage transient (back-EMF) that can exceed hundreds of volts, risking permanent damage to the board.
- **Clamp Diodes:** A fast-switching flyback diode is placed in a reverse-biased configuration directly across each relay coil terminals. 
- **Transient Dissipation:** Upon turn-off, the induced inductive current safely circulates through the freewheeling diode, dissipating the stored magnetic energy as thermal loss across the diode's forward voltage drop and parasitic resistance.

<p align="center">
    <img src="../docs/img/lv_relay_drive_1_sheet.png" style="width: 850px; float: center;" >
    <img src="../docs/img/lv_relay_drive_2_sheet.png" style="width: 850px; float: center;" >
    <img src="../docs/img/lv_relay_drive_3_sheet.png" style="width: 850px; float: center;" >
</p>


## 5. High voltage sensing

### 5.1 DC_LINK mesurement (100%)

The DC_LINK sensing branch measures the actual voltage on the inverter side, directly monitoring the charge state of the internal DC-Link capacitor. The measurement point is located downstream of the positive relays to read the actual voltage present at the inverter terminals. A high-impedance resistive divider scales the signal to $100\%$ of the nominal measurement range, while Zener clamping diodes and a low-pass RC filter protect the input stage against overvoltage transients and high-frequency noise.

<p align="center">
    <img src="../docs/img/hv_sensing_1_sheet.png" style="width: 650px; float: center;" >
</p>

### 5.2 TS_POS mesurement (95%)

The TS_POS sensing branch measures the high-voltage battery pack (TS) voltage upstream of the positive relays. Even TS_NEG is mesured upstream of the relays, because when the board is operational the AIR NEG relay is always closed. This resistive divider is scaled to output $95\%$ of the measured voltage relative to the DC_LINK scale, setting the hardware precharge completion threshold ($V_{\text{DC\_LINK}} = 0.95 \cdot V_{\text{TS\_POS}}$). The input line is protected by high-voltage current-limiting resistors, Zener clamping diodes, and a low-pass RC filter.

<p align="center">
    <img src="../docs/img/hv_sensing_2_sheet.png" style="width: 650px; float: center;" >
</p>

### 5.3 Galvanic insulation

To ensure complete electrical separation between the high-voltage domain and the low-voltage logic, the circuit implements a dual galvanic isolation barrier without a shared ground:
- **Power isolation:** an isolated DC/DC converter locally generates the supply line for the HV analog sensing and comparison stages starting from the low-voltage supply line.
- **Signal isolation:** the digital output of the comparison stage is transmitted across the barrier to the LV logic (CHARGE_REQUIRED_IN) through an optocoupler, preserving full galvanic isolation.

<p align="center">
    <img src="../docs/img/hv_sensing_3_sheet.png" style="width: 700px; float: center;" >
    <img src="../docs/img/hv_sensing_4_sheet.png" style="width: 650px; float: center;" >
</p>

### 5.4 Comparation

DC_LINK (scaled to 100%) and TS_POS (scaled to 95%) are fed into the inputs of a voltage comparator. As long as the inverter capacitor voltage remains below 95% of the battery pack voltage, the comparator keeps the CHARGE_REQUIRED_IN signal up. As soon as the DC_LINK voltage exceeds the 95% TS_POS threshold, the comparator output switches state, signaling precharge completion to the LV logic.

<p align="center">
    <img src="../docs/img/hv_sensing_5_sheet.png" style="width: 250px; float: center;" >
</p>


## 6. High voltage power

The high-voltage power stage (hv_power) performs the final actuation of the system, interfacing the battery pack with the inverter through the three main relays (negative AIR, precharge relay with series current-limiting resistor, and positive AIR). The relay coils are powered directly by the +12V_SHDW line (Shutdown Circuit) on the high side and switched to ground (GND) on the low side via N-channel MOSFETs driven by the CMD_AIR_NEG_OUT, CMD_AIR_PRE_OUT, and CMD_AIR_POS_OUT logic outputs:
1. **Precharge start:** with the Shutdown Circuit closed and a valid ENABLE_IN signal, the logic closes the negative AIR and the precharge relay, gradually charging the DC-Link capacitor through the precharge resistor.
2. **Completion and run state:** once 95% of the battery voltage is reached (CHARGE_REQUIRED_IN low), the positive AIR closes and the precharge relay simultaneously opens, holding the state latched via the SR latch.
3. **Safe state:** any opening of the Shutdown Circuit (+12V_SHDW) immediately removes coil power and resets the control logic, physically opening all high-voltage contacts.

<p align="center">
    <img src="../docs/img/hv_power_1_sheet.png" style="width: 650px; float: center;" >
    <img src="../docs/img/hv_power_2_sheet.png" style="width: 650px; float: center;" >
    <img src="../docs/img/hv_power_3_sheet.png" style="width: 650px; float: center;" >
</p>

