We now complete the Transone Central Processing Unit (CPU) by designing the **Register File (RAM)** and the **Control Unit (Decoder)**, then integrating them with the existing ALU. Every component operates on the same multi‑wire, four‑state voltage foundation, ensuring that numeric and character data are indistinguishable at the hardware level. The CPU fetches instructions, decodes them, orchestrates data movement, and executes operations—all governed by the laws of analog summation and threshold detection.

---

## 1. CPU Architecture Overview

The Transone CPU consists of three tightly coupled blocks:

- **Register File** – stores multi‑column data words (numbers or strings) as precise charge states.
- **Control Unit** – fetches instruction words, decodes them via analog comparators, and generates the sequencing signals that drive the ALU and register file.
- **ALU (Digit Processing Element chain)** – the column‑based arithmetic/logic engine already detailed.

A Harvard‑style architecture is employed: instruction memory and data memory are separate, both accessible in the same clock cycle. The word width is parametric; for this design we assume **8 columns per word**, enough for a 4‑digit number or two characters (4 columns each). The ALU is accordingly expanded to 8 DPEs, retaining the right‑to‑left carry/borrow chain.

```
                    ┌─────────────┐
                    │ Instruction │
                    │   Memory    │
                    └──┬──────────┘
                       │ instruction word (8 columns)
                       ▼
               ┌───────────────┐
               │ Control Unit  │
               │ (Decoder +    │
               │  Sequencer)   │
               └───┬───────┬───┘
        register   │       │   ALU Op‑Sel
        controls   │       │
                   ▼       ▼
          ┌─────────────────────────────┐
          │       Register File         │
          │  (8‑column multi‑port RAM)  │
          └─┬───────────────────────┬───┘
            │ A‑bus (8×5 wires)     │ B‑bus (8×5 wires)
            ▼                       ▼
   ┌─────────────────────────────────────────┐
   │            ALU (8‑DPE chain)            │
   │  Col7 … Col0 with Carry/Borrow Chain    │
   └────────────────────┬────────────────────┘
                        │ Result‑bus (8×5 wires)
                        ▼
               Back to Register File
               or Data Memory (load/store)
```

All buses are multi‑wire groups. For example, an 8‑column bus is physically a bundle of up to 40 wires (each column up to 5 wires), plus a few extra for carry/borrow. The voltages are strictly confined to {−1 V, 0 V, 1 V, 2 V}.

---

## 2. Register File (Multi‑Column Volatile RAM)

The register file stores **G words** of **C columns** each (here G=8 registers, C=8 columns). Every column cell holds a digit 0–9 using a capacitive storage well and precision comparators, as described in the earlier memory cell blueprint.

### 2.1 Storage Cell Design (One Column)

Each column cell stores a digit value as an **exact quantity of charge** in a small capacitor. The cell has:

- **Capacitor Cstore** – physical capacitor, sized to hold up to 9 V without dielectric breakdown.
- **Access transistor** – pass‑gate enabled when the register is selected for read/write.
- **Three reference comparators** – set at −0.5 V, +0.5 V, +1.5 V to distinguish the four legal states in a single‑wire readout, but here we need to store multi‑wire digits 0–9. Instead of storing a single analog voltage 0–9 V directly (which would violate the rule that wires never carry intermediate voltages), the cell stores the **total charge** that corresponds to the sum of the wire voltages that represent the digit. The write process converts the multi‑wire column input into a precise current that charges the capacitor. The read process reconstructs the multi‑wire output using an encoder.

**Write operation:**  
The incoming column wires (0–9 V total) are connected to a summing node that generates a current proportional to the digit value. This current charges the capacitor through a fixed time window, so the stored voltage \( V_{store} = k \cdot \text{digit} \). For example, digit 9 produces \( V_{store} = 2.25\) V (if scaled). A simple scaling factor keeps \( V_{store} \) within safe linear range of the readout amplifier.

**Read operation:**  
When the cell is read, a high‑impedance buffer Op‑Amp senses \( V_{store} \) without draining the capacitor. An analog‑to‑wires encoder (the same used in the ALU output stage) converts this scalar voltage back into the standard multi‑wire column format: floor(digit/2) wires at 2 V, plus one 1 V wire if the digit is odd. The output drivers are Op‑Amp buffers with diode clippers to enforce the legal state limits.

Thus, the storage element itself does not need to hold a voltage outside the legal range; it holds a scaled version that is always between 0 V and 2.25 V (a range that can be accommodated with careful design, and the readout encoder reconstructs the legal states). All exposed column outputs are strictly 0 V, 1 V, or 2 V.

**Refresh:** The capacitor charge leaks slowly. A background refresh cycle reads every cell and writes it back, similar to DRAM. During refresh, the multi‑wire output is temporarily generated and then fed back to the write circuit.

### 2.2 Register File Array

For G=8 registers of C=8 columns, we arrange an array of 64 storage cells (8 rows × 8 columns). Two read ports (A‑bus and B‑bus) and one write port (Result‑bus) are supported.

- **Address decoding:** Register selection is controlled by **register address buses** from the Control Unit. Each address is a 1‑column digit (0–7, encoded as a single voltage or as a multi‑wire digit). Comparators in the address decoder activate the corresponding row’s access transistors. The address decoding uses threshold comparators, entirely analog and within the Transone framework.
- **Multi‑port implementation:** Each cell has multiple access transistors, one per port. Isolation resistors prevent cross‑talk when multiple ports read the same cell simultaneously.
- **Output buses:** The A‑bus and B‑bus each consist of C column bundles. For a read, the selected row’s cells drive their encoded multi‑wire outputs onto the respective buses. Simultaneous reads from different registers on A and B ports are allowed.

### 2.3 Integration with Character Data

The register file stores character strings exactly as multi‑column numbers. For example, the string `"ab"` (codes 01, 02) occupies four columns: [0,0,0,1,0,0,0,2] (if columns numbered 7..0). When the Control Unit issues a load instruction, the character data arrives from memory as voltage clusters and is written to the register file. No conversion occurs. The ALU can later process these columns as arithmetic operands or as character codes—the register file is agnostic.

---

## 3. Control Unit (Decoder and Sequencer)

The Control Unit fetches instruction words from instruction memory, decodes them to generate the Op‑Sel signals for the ALU and the address/control signals for the register file, and sequences the right‑to‑left column execution.

### 3.1 Instruction Format

Transone instructions are **multi‑column words** themselves, stored in instruction memory in the same voltage‑cluster format. A typical instruction occupies one 8‑column word, divided into fields:

- **Opcode:** 2 columns (Col7, Col6) – specifies the operation (e.g., ADD=01, SUB=02, MUL=03, DIV=04, AND=05, OR=06, NOT=07, MOV=08, LOAD=09, STORE=10, etc.).
- **Destination Register (Rd):** 1 column (Col5) – register number 0–7.
- **Source Register 1 (Rs1):** 1 column (Col4).
- **Source Register 2 / Immediate field (Rs2/Imm):** 4 columns (Col3..Col0) – either a register number (if opcode uses register‑register) or an immediate value (if opcode uses immediate). A mode bit in the opcode can differentiate, or distinct opcodes for register and immediate forms.

Example: `ADD R1, R2, R3` encoded as columns: [0,1] (opcode ADD=01), [0,1] (Rd=R1), [0,2] (Rs1=R2), [0,3] (Rs2=R3), and remaining columns zero.

Immediate values up to 9999 can be encoded directly; for character operations, the immediate field would hold the 2‑digit character code (e.g., 30 for case offset) or a string constant.

### 3.2 Instruction Fetch

A **Program Counter (PC)** is a multi‑column register (e.g., 4 columns to address memory). After reset, the PC outputs its value to the instruction memory address bus. The memory returns an 8‑column instruction word. This word is latched into the **Instruction Register (IR)**, a special register inside the Control Unit built from the same capacitive storage cells.

The PC is incremented after fetch: a dedicated adder (or the ALU itself, if we borrow it) adds 1 to the PC value, and the result is stored back to PC. To keep the CPU self‑contained, we include a small **PC incrementer** that is just a single‑digit adder with carry across the PC columns; it respects the same multi‑wire rules.

### 3.3 Instruction Decoding

The decoder is an analog circuit that extracts the opcode and operand fields from the IR and generates control signals.

- **Field extraction:** The IR columns are physically grouped. The opcode columns (Col7, Col6) are directly routed to a **opcode decoder block**. This block uses threshold comparators to recognize specific code numbers. For instance, the opcode for ADD is the decimal number 01. The decoder employs a parallel set of comparators: one set subtracts the constant 1 from the opcode field; if the result is exactly zero (all column voltages are zero), the “ADD” line is driven to 1 V. Another set checks for SUB (code 02), etc. This is done by analog subtraction and zero‑detection, entirely feasible with the DPE design. The zero‑detection circuit simply checks if all result column wires are at 0 V.

- **Operand fields:** The destination register field (Col5) is routed to a decoder that activates one of the 8 write‑enable lines for the register file. The source register fields (Col4 and possibly Col3) are sent to read‑address decoders for the A and B ports.

- **Immediate field:** If the instruction uses an immediate, the 4 columns from Col3..Col0 are routed directly to the ALU’s B‑bus as an immediate operand, bypassing the register file.

The decoder outputs are all in the form of 1 V control signals (enabled) or 0 V (disabled), which are acceptable as they control switching transistors.

### 3.4 Sequencer (Microcode Execution)

The ALU’s column‑sequential execution is managed by a **sequence counter**. For a multi‑column operation, the sequencer:

1. Asserts the ALU Op‑Sel lines (based on decoded opcode) throughout the entire operation.
2. Starts with column 0 (Ones). It enables the column 0 DPEs on both A and B buses, and forces the Carry/Borrow‑In of column 0 to 0 V.
3. After the DPEs settle (a few nanoseconds), the column 0 result is captured in a temporary latch, and its Carry/Borrow Out is forwarded to column 1.
4. The sequencer then increments a column counter, enabling column 1 DPEs and using the latched carry. This repeats up to column 7 (or until the highest column specified by the data width).
5. When all columns are processed, the final result is written to the register file.

The sequencer is a simple finite state machine. Its state is stored in a multi‑column register (the column counter), which is incremented by a dedicated +1 adder (again, using the same analog summing rules). The clock is a stable oscillator that triggers each step.

For **character‑string operations**, the sequencer may handle only the necessary columns (e.g., columns 0 and 1 for single character), leaving higher columns untouched. The opcode can specify the size (byte=2 columns, word=4 columns, dword=8 columns). The sequencer reads a size field and stops early, saving power.

### 3.5 Example: Case Conversion Instruction

Suppose the CPU executes a custom “TO‑UPPER” instruction that converts a character in register R1 from lowercase to uppercase by adding 30. This can be synthesized from existing operations, but we can also implement it as a single microcoded instruction.

1. Fetch: IR ← `TOUPPER R1` (opcode=11, Rd=R1, Rs1=R1, Imm=30).
2. Decode: opcode=11 activates the “ADD‑IMM” control path.
3. Execute: Sequencer routes R1 to A‑bus, immediate 30 to B‑bus. Column 0 (Ones) DPE receives A=ones digit of character (from R1’s Col0), B=0 (imm ones=0). Computes sum, carry if any. Column 1 (Tens) DPE receives A=tens digit, B=3 (imm tens=3). Computes sum, result stored back to R1’s Col0 and Col1. Higher columns of R1 are zeroed or passed through. This is identical to any 2‑digit addition.

Thus, the hardware sees no difference between adding 30 to a number and converting case.

---

## 4. Complete CPU Cycle

A single Transone instruction cycle:

1. **Fetch:** PC → instruction memory → IR. PC incremented.
2. **Decode:** Opcode field compared, control lines generated. Register addresses decoded.
3. **Operand Read:** Rs1 and Rs2 read from register file onto A‑bus and B‑bus.
4. **Execute (multi‑step sequence):** Sequencer steps columns 0..width-1, DPEs compute, carry/borrow chain ripples, results latched.
5. **Write Back:** Result‑bus written to Rd register.

All these steps happen within one major clock cycle, with minor sub‑cycles for the column stepping. The control unit orchestrates the timing.

---

## 5. Integration: Numbers and Characters Coexist Natively

Because the entire CPU—registers, ALU, control—treats all data as multi‑column decimal digits, there is no distinction between numeric and character data. The character encoding map simply assigns numbers 1–26 and 31–56 to letters. The CPU handles them transparently:

- **Load string:** Memory → register file as column clusters.
- **String compare:** SUBTRACT operation on two registers; borrow flag indicates order.
- **Case conversion:** ADD immediate 30.
- **String copy:** MOV instruction moves columns.

The program counter, address calculations, and loop counters are all numeric values, but they share the same registers and ALU as character data. The seamless integration is a direct result of the Transone principle: computation through physical voltage sums, with decimal columns representing both numbers and character codes.

---

## 6. Conclusion

We have completed the Transone CPU by constructing:

- A **multi‑column capacitive register file** that stores digits 0–9 as scaled charge and reconstructs the legal voltage clusters on read.
- A **control unit** that fetches multi‑column instruction words, decodes them with analog comparators, and sequences the ALU’s right‑to‑left column execution.
- Full integration of the character encoding, so that the same ALU operations that add numbers also add character codes to perform case conversion or string arithmetic.

The entire CPU operates exclusively within the four‑state voltage system, with no binary data representation. The result is a machine where the boundaries between numeric and character processing dissolve—a unified analog‑digital architecture that computes directly on the electrical representation of meaning.