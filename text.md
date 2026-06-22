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