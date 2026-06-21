# Transone Architecture Complete Specification

The Transone architecture is an alternative hardware design that processes information using four distinct electrical voltage states instead of the standard binary two-state system. The name **Transone** stands for **Transistor-Addition-to-One**, representing a device that functions like a transistor but natively combines multiple input voltage signals into a unified output wire.

## 1. Architectural Definition and Hardware Foundation

The foundation of Transone relies on a four-state voltage scale. Instead of using complex multi-transistor binary logic gates to compute mathematical rules, the system routes raw electrical voltages through analog components to allow physical laws to execute math.

### The Four Allowed Physical States

Every wire inside a Transone core must strictly carry one of these four electrical signals. Voltages outside these specific windows are illegal:

* **State 0:** 0.0V (Represents the value 0 / System Off)
* **State -1:** -1.0V (Represents the value -1 / Reverse Current)
* **State 1:** 1.0V (Represents the value 1 / Low Positive Current)
* **State 2:** 2.0V (Represents the value 2 / High Positive Current)

### High-Level Representation of Base-10 Numbers (0 to 9)

Because the hardware is limited to the four base states above, multi-digit numbers from 0 to 9 are represented by grouping individual Transone state wires into structural columns, exactly like the human decimal system. Each column wire can hold a maximum value of State 2.

To represent larger numbers like 6, 8, or 10, the architecture stacks state wires in parallel within that specific column:

* **Number 0:** One wire at State 0 (0V)
* **Number 1:** One wire at State 1 (1V)
* **Number 2:** One wire at State 2 (2V)
* **Number 3:** One wire at State 2 + One wire at State 1 (2V + 1V = 3V total value)
* **Number 4:** Two wires at State 2 (2V + 2V = 4V total value)
* **Number 6:** Three wires at State 2 (2V + 2V + 2V = 6V total value)
* **Number 8:** Four wires at State 2 (2V + 2V + 2V + 2V = 8V total value)
* **Number 9:** Four wires at State 2 + One wire at State 1 (2V + 2V + 2V + 2V + 1V = 9V total value)
* **Number 10:** Shifts entirely to the next physical column (The Tens Column wire receives a State 1, while the Ones Column wires drop to State 0).

---

## 2. Structural Arrangement and Execution Order

To perform operations correctly, Transone hardware blocks must be arranged in a specific physical sequence:

1. Inputs enter through matched isolation resistors to prevent backward current leakage.
2. The signals merge at a central node where physical currents naturally sum together.
3. The merged signal is stabilized and cleaned by an Operational Amplifier (Op-Amp).
4. The output is evaluated by a Voltage Comparator or Clipper circuit to handle boundaries.
5. Multi-column calculations must always be executed from right to left (Ones column first, then Tens, then Hundreds) so that any Carry or Remainder voltages can feed forward into the next stage of processing.

---

## 3. Basic Operation 1: Addition

Addition merges positive voltage inputs through an Op-Amp summing configuration. If the combined input exceeds the maximum capacity of the current column, a comparator circuit strips away the excess voltage and triggers a State 1 carry signal to the next column.

### Examples
* **Example 1: 1 + 1**
    * Transone Method: Input A supplies 1V. Input B supplies 1V. They combine at the node.
    * Solution: The Op-Amp reads the combined current and outputs a clean 2V. The system registers State 2, which equals 2.
* **Example 2: 2 + 1**
    * Transone Method: Input A supplies 2V. Input B supplies 1V. They combine at the node.
    * Solution: The Op-Amp registers the combined 3V force. Because this exceeds a single wire state capacity, the system balances it across two parallel output wires in the Ones column: one wire outputs 2V and the second outputs 1V. The system registers a total value of 3.
* **Example 3: 4 + 4**
    * Transone Method: Input A uses two parallel wires at 2V each (4V total). Input B uses two parallel wires at 2V each (4V total). All four inputs merge at the processing node.
    * Solution: The combined voltage equals 8V. The Op-Amp array splits this cleanly across four parallel output wires in the Ones column, each running at 2V. The system registers a total value of 8.
* **Example 4: 6 + 3**
    * Transone Method: Input A uses three wires at 2V (6V total). Input B uses one wire at 2V and one wire at 1V (3V total). All lines merge.
    * Solution: The combined input equals 9V. The Op-Amp array routes this into four output wires at 2V and one output wire at 1V within the Ones column. The system registers a total value of 9.
* **Example 5: 8 + 6**
    * Transone Method: Input A uses four wires at 2V (8V total). Input B uses three wires at 2V (6V total). All lines merge.
    * Solution: The total incoming force is 14V. The column comparator sees that this exceeds the decimal boundary of 9V. The comparator strips 10V from the pool and routes a State 1 (1V) signal to the Tens column wire. The remaining 4V stays in the Ones column, distributed across two wires at 2V each. The final readout is a 1 in the Tens place and a 4 in the Ones place, equaling 14.

---

## 4. Basic Operation 2: Subtraction

Subtraction is executed by injecting a negative electrical pressure (State -1 / -1V) into the summing node to counteract positive voltages. When dealing with larger subtractions, Transone units are arranged in a sequential pipeline chain where each stage subtracts a maximum step of -1V until the full subtraction value is achieved.

### Examples
* **Example 1: 2 - 1**
    * Transone Method: Input A supplies 2V. The subtraction lines inject a single -1V signal into the node.
    * Solution: The voltages meet and cancel each other out: 2V + (-1V) = 1V. The Op-Amp outputs a clean 1V (State 1), which equals 1.
* **Example 2: 1 - 1**
    * Transone Method: Input A supplies 1V. The subtraction lines inject a single -1V signal into the node.
    * Solution: The positive and negative currents balance perfectly: 1V + (-1V) = 0V. The Op-Amp outputs 0V (State 0), which equals 0.
* **Example 3: 1 - 2 (The Double Drop Chain)**
    * Transone Method: To subtract 2 without using an unauthorized -2V state, the signal passes through two sequential stages.
    * Solution: Stage 1 takes the initial 1V and adds -1V, outputting 0V. This 0V output wire acts as the direct input for Stage 2. Stage 2 takes that 0V and adds the second -1V step. The final output wire reads a clean -1V (State -1), which equals the correct negative value -1.
* **Example 4: 6 - 2**
    * Transone Method: Input A consists of three wires running at 2V each (6V total). The subtraction architecture routes the signal through a two-stage pipeline where each stage injects a -1V signal into the wire cluster.
    * Solution: Stage 1 drops the total pool from 6V to 5V. Stage 2 drops the remaining pool from 5V to 4V. The final output is distributed across two wires running at 2V each. The system registers a total value of 4.
* **Example 5: 10 - 2**
    * Transone Method: The input starts with a State 1 (1V) on the Tens column wire and 0V on the Ones column. To perform the calculation, the 1V Tens signal is broken down into its equivalent Ones column values: five wires running at 2V each (10V total). This wire cluster is passed through a two-stage subtraction pipeline.
    * Solution: Stage 1 injects -1V, reducing the voltage pool to 9V. Stage 2 injects another -1V, reducing the final pool to 8V. The output wires stabilize into four parallel lines running at 2V each in the Ones column. The system registers a total value of 8.

---

## 5. Basic Operation 3: Multiplication

Multiplication scales an input voltage array using an analog multiplier Op-Amp circuit. To maintain the structural integrity of the core, a diode clipper circuit acts as a hard boundary wall on the output lines, preventing voltages from drifting into unauthorized states and routing excess values to overflow channels.

### Examples
* **Example 1: 2 * 1**
    * Transone Method: Input A supplies 2V. Input B supplies 1V.
    * Solution: The multiplier circuit scales the 2V input by a factor of 1. The output wire stabilizes at 2V (State 2), which equals 2.
* **Example 2: 2 * -1**
    * Transone Method: Input A supplies 2V. Input B supplies -1V.
    * Solution: The multiplier circuit processes the inputs and physically flips the electrical polarity. Because a raw -2V output is illegal under the 4-state rule, the output clipper constraints the primary line to -1V (State -1) and activates an auxiliary underflow line to denote the remaining missing value.
* **Example 3: 4 * 2**
    * Transone Method: Input A uses two parallel wires at 2V each (4V total). Input B uses a control signal scaled to a factor of 2.
    * Solution: The analog multiplier scales the incoming 4V electrical force by 2, resulting in a total internal value of 8V. The circuit maps this cleanly across four parallel output lines running at 2V each. The system registers a total value of 8.
* **Example 4: 3 * 3**
    * Transone Method: Input A uses one 2V wire and one 1V wire (3V total). Input B supplies a scaling factor of 3.
    * Solution: The circuit multiplies the total incoming force of 3V by 3, generating an internal sum of 9V. The output stages balance the current across four parallel 2V wires and one 1V wire. The system registers a total value of 9.
* **Example 5: 6 * 2**
    * Transone Method: Input A uses three parallel wires at 2V each (6V total). Input B applies a scaling factor of 2.
    * Solution: The multiplier processes the inputs to generate a total internal force of 12V. The column comparator flags that this output exceeds the single-column decimal limit of 9V. The system subtracts 10V from the pool, sending a State 1 (1V) carry signal to the Tens column wire. The remaining 2V stays in the Ones column on a single wire running at 2V. The combined multi-column readout indicates 12.

---

## 6. Basic Operation 4: Division

Division scales an input voltage down using an analog divider circuit configuration. Fractional values that fall between the allowed states are cleared from the primary output line using a remainder routing system, which grounds the fractional line to a clean state and outputs the leftover value on a separate remainder wire.

### Examples
* **Example 1: 2 / 1**
    * Transone Method: Input A (Numerator) supplies 2V. Input B (Denominator) supplies 1V.
    * Solution: The divider circuit scales the 2V input down by a factor of 1. The output wire reads a clean 2V (State 2), which equals 2.
* **Example 2: 2 / 2**
    * Transone Method: Input A supplies 2V. Input B supplies 2V.
    * Solution: The divider circuit scales the 2V input down by a factor of 2. The output wire drops to a clean 1V (State 1), which equals 1.
* **Example 3: 8 / 2**
    * Transone Method: Input A consists of four parallel wires at 2V each (8V total). Input B supplies a denominator factor of 2.
    * Solution: The divider circuit processes the total 8V input and cuts the electrical force in half. The resulting 4V output is distributed across two parallel wires running at 2V each. The system registers a total value of 4.
* **Example 4: 1 / 2 (The Fractional Remainder)**
    * Transone Method: Input A supplies 1V. Input B supplies 2V.
    * Solution: The mathematical result is 0.5, which is an illegal intermediate voltage. The internal sensing circuit prevents the main line from entering the dead zone by pulling the primary output wire down to a clean 0V (State 0). Simultaneously, the uncomputed 1V input signal is diverted directly out of a secondary remainder wire. The hardware reads the final state as 0 with a remainder of 1.
* **Example 5: 6 / 0 (The Zero Firewall)**
    * Transone Method: Input A supplies three wires at 2V each (6V total). Input B supplies 0V.
    * Solution: Attempting to divide by zero causes an analog circuit to pull unsafe levels of current. Before the signals can merge at the divider circuit, an inline supervisor transistor detects the 0V input on line B and snaps shut, physically breaking the connection. The divider is safely isolated, the main outputs are grounded to 0V, and a dedicated error wire is injected with a 1V signal to notify the system of an invalid operation.

---

## 7. Technical Considerations

### Memory Cells for Multi-Wire Quaternary States
To store the multi-wire states of the Transone architecture without reverting to binary, the memory system must be designed around **Capacitive Volatile Storage** or **Floating-Gate Non-Volatile Storage**. Instead of a single capacitor that is either fully charged (1) or completely empty (0), a Transone memory cell uses a specialized storage element that traps precise quantities of electrical charge.

**The Transone Cell Blueprint**
Each physical storage cell consists of a tiny capacitive well and three built-in reference comparators tuned exactly to the internal state thresholds: -1.0V, 1.0V, and 2.0V.

* **Writing to Memory:** When the CPU pushes a State 2 (2.0V) signal down a wire, an access transistor opens, and the storage well is filled to maximum capacity. If a State -1 signal is sent, the circuit pulls electrons out of the well, creating a negative charge state.
* **Reading from Memory:** When the cell is read, the stored electrical pressure is routed through an ultra-high impedance buffer Op-Amp. This amplifier senses the exact charge level without draining it, cleanly regenerating the precise voltage (0V, -1V, 1V, or 2V) back onto the multi-wire bus lines.

### Handling Logic Operations (AND, OR, NOT)
Because Transone processes arithmetic through analog voltage totals, traditional binary logic gates cannot be used directly. The architecture replaces standard Boolean logic with **Threshold Logic** and **Polarity Inversion**.

* **The NOT Gate (The Inverter):** In binary, NOT turns a 1 to a 0. In Transone, the NOT operation is handled by an **Inverting Amplifier** with a gain of exactly -1. Inputting a State 1 (1.0V) into the inverter causes the Op-Amp to output a State -1 (-1.0V). Inputting a State 2 (2.0V) outputs an out-of-bounds -2.0V, which is immediately caught by the system's Clipper circuit and conditioned into a State -1 along with an overflow flag.
* **The AND / OR Gates (Threshold Comparators):** To execute conditional decisions like AND and OR, Transone routes the input wires into a standard summing node, followed immediately by a voltage comparator acting as a gatekeeper.
    * **The OR Operation:** The comparator is tuned to a low threshold of 0.5V. If *either* Input A OR Input B carries voltage, the combined node voltage passes the 0.5V threshold, and the comparator outputs a clean State 1 (1.0V).
    * **The AND Operation:** The comparator threshold is raised to 1.5V. If only one input is active (1.0V), it fails to cross the threshold, resulting in an output of 0V. Only when *both* inputs are active simultaneously does the combined voltage (1.0V + 1.0V = 2.0V) cross the 1.5V line, triggering a clean State 1 output.

### Parallel Subtraction Pipeline Optimization
The sequential subtraction pipeline (Option A) can be completely parallelized while strictly respecting the rule that no single wire may exceed a State -1 (-1.0V) electrical pressure. This is achieved through a technique called **Spatial Distribution**.

Instead of waiting for a single voltage signal to step down through successive stages over multiple clock cycles, the CPU uses a **Parallel Subtraction Matrix**. If the system needs to subtract 3 from a data wire:

1. The control unit opens three independent, parallel reference lines simultaneously, each carrying exactly -1.0V.
2. These three lines do not combine into a single wire (which would create an illegal -3.0V signal). Instead, they are routed to three separate, parallel inputs on a multi-channel summing Op-Amp.
3. The math is executed in a single clock cycle because the Op-Amp processes all three incoming currents at the exact same instant, dropping the target data pool down by 3 steps immediately.

This layout eliminates sequential delay, turning subtraction into a high-speed, single-cycle operation just like addition.

### Thermal and Power Density Implications
Running millions of operational amplifiers continuously inside a dense processor presents a significant engineering challenge: **Static Power Dissipation**. In standard binary CMOS chips, transistors only draw significant power when they are actively switching between 0 and 1. If a computer is sitting idle, it uses very little electricity. Op-Amps, however, are linear analog circuits. To remain perfectly accurate and stable, their internal transistors must stay constantly biased with active current flowing through them, even when the computer is performing no calculations.

* **Thermal Runaway:** Constant current creates continuous heat. If an array of millions of Transone Op-Amps is packed into a tight silicon space, the ambient temperature will rise rapidly, degrading the precision of the resistor networks and leading to calculation errors.
* **Voltage Drift:** As heat rises, the electrical resistance of the wires changes, causing a clean 1.0V signal to drift down to 0.8V or up to 1.2V, threatening the stability of the narrow safety zones.

**Architectural Solutions:**
1. **Power Gating:** The chip architecture must be broken into independent zones. If a specific ALU block is not performing multiplication or division during a clock cycle, its master power supply lines are physically disconnected by a power-gating transistor, completely cutting off the static current.
2. **Chopper Stabilization:** The Op-Amps must use high-frequency clocking mechanisms that constantly cycle the amplifiers on and off, averaging out thermal noise and drift over time to maintain absolute mathematical accuracy at high speeds.

---

## 8. Transone ALU: Single-Digit Processing Element (DPE)

The arithmetic-logic unit is built from an array of identical Digit Processing Elements (DPEs), one per decimal column. A control sequencer steps through columns from right (Ones) to left (Most Significant Digit), feeding each DPE with its operand digits and a carry-in from the previous lower column.

Each DPE accepts:
* **A-bus:** up to five wires carrying the encoded value of digit A (0–9).
* **B-bus:** up to five wires carrying digit B (0–9).
* **Carry/Borrow In:** a single wire carrying either 0V (no carry) or 1V (State 1, indicating a carry/borrow of 10 in the current column’s scale).
* **Operation Code (Op-Sel):** four control lines that configure internal signal routing.
* **Remainder In:** a single wire from a previous division remainder, if any.

It produces:
* **Result-bus:** up to five wires encoding the final digit result (0–9) after the operation.
* **Carry/Borrow Out:** 0V or 1V to be passed to the next higher column.
* **Remainder Out / Error flag:** auxiliary output for fractional remainders or division-by-zero detection.

### DPE Functional Stages
The DPE is divided into five functional stages, cascaded in a single clock cycle:

1. **Input Conditioning & Isolation:** Incoming wires pass through matched isolation resistors (e.g., 10kΩ) to prevent current back-flow and convert voltage to current for summing.
2. **Operation-Specific Analog Compute Core:** Routes currents based on the Op-Code:
    * **Addition:** Multi-input summing amplifier.
    * **Subtraction:** Parallel subtractor matrix using reference generators to handle B values.
    * **Multiplication:** Analog multiplier (e.g., Gilbert cell) scaling A by B.
    * **Division:** Analog divider (multiplier in feedback loop) with remainder logic.
    * **Logic:** Threshold comparators for NOT, AND, OR.
    * **Pass-Through:** Direct routing.
3. **Column Boundary Enforcement & Carry Generation:** Carry-look-ahead comparator chain ensures the result is within 0–9. Adds/subtracts 10V if overflow/underflow occurs and sets carry/borrow out.
4. **Output Encoding & Clipping:** Converts single-ended voltage to multi-wire representation using reference ladders and diode clippers to ensure no signal exceeds limits.
5. **Status & Remainder Routing:** Handles remainder output, division-by-zero protection, and flags.

### Multi-Column ALU Operation & Control Sequencer
The control sequencer is a simple state machine that:
1. Splits source operands into digit columns.
2. The Ones column DPE begins (Carry-In hardwired to 0V).
3. The carry/borrow out from column N connects to the Carry/Borrow-In of column N+1.
4. This right-to-left propagation continues until the Most Significant Digit DPE finishes.

---

## 9. Summary
The Transone ALU is a direct physical embodiment of the Transone philosophy: computation through the natural summation of electrical pressures. It features:
* Usage of four legal voltage states (-1V, 0V, 1V, 2V).
* Decimal digit encoding (0–9) using multi-wire groups.
* Analog-domain arithmetic (Addition, Subtraction, Multiplication, Division) and logic (NOT, AND, OR).
* Column-sequential execution for multi-digit operations.
* Advanced thermal management via power gating and chopper stabilization.

This architecture is designed to be practical, scalable, and fully faithful to the Transone technical specification.
