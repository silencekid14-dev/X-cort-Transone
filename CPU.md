# Transone Central Processing Unit (CPU) Specification

We now complete the Transone Central Processing Unit (CPU) by designing the **Register File (RAM)** and the **Control Unit (Decoder)**, then integrating them with the existing ALU. Every component operates on the same multi‑wire, four‑state voltage foundation, ensuring that numeric and character data are indistinguishable at the hardware level. The CPU fetches instructions, decodes them, orchestrates data movement, and executes operations—all governed by the laws of analog summation and threshold detection.

---

## 1. CPU Architecture Overview

The Transone CPU consists of three tightly coupled blocks:

* **Register File** – stores multi‑column data words (numbers or strings) as precise charge states.
* **Control Unit** – fetches instruction words, decodes them via analog comparators, and generates the sequencing signals that drive the ALU and register file.
* **ALU (Digit Processing Element chain)** – the column‑based arithmetic/logic engine.

A Harvard‑style architecture is employed: instruction memory and data memory are separate, both accessible in the same clock cycle. The word width is parametric; for this design we assume **8 columns per word**, enough for a 4‑digit number or two characters (4 columns each). The ALU is accordingly expanded to 8 DPEs, retaining the right‑to‑left carry/borrow chain.

### Block Diagram

```text
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
     │         Register File       │
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