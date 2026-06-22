                    Character/String Data from Memory
                          │ (2 columns per char)
                          ▼
┌──────────────────────────────────────────────────────────┐
│                 Register File (Multi‑Column)             │
│  Each register holds N columns of 0–9, wired as voltage  │
│  clusters. Registers can be treated as numeric or char.  │
└────────┬──────────────────────────────────┬──────────────┘
         │ A‑bus (multi‑column)             │ B‑bus
         ▼                                  ▼
┌──────────────────────────────────────────────────────────┐
│                    ALU Column Chain                      │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐       │
│  │ DPE    │◄──│ DPE    │◄──│ DPE    │◄──│ DPE    │       │
│  │Col 3   │   │Col 2   │   │Col 1   │   │Col 0   │       │
│  │(Thous.)│   │(Hunds.)│   │(Tens)  │   │(Ones)  │       │
│  └────────┘   └────────┘   └────────┘   └────────┘       │
│       ▲            ▲            ▲            ▲           │
│       └────────────┴─ Carry/Borrow Chain ──┘             │
│                                                          │
│  Operation Selection: Op‑Sel decoder (Add,Sub,Mul,Div,   │
│                        AND,OR,NOT,Pass)                  │
└──────────────────────────┬───────────────────────────────┘
                           │ Result‑bus (multi‑column)
                           ▼
              Back to Register File / Memory