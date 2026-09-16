# Nestlé Wafer Inspection (Ponda Line 3, Phase 1)

Clone of this repo should look like:

```
.
├── README.md
├── Nestlé Line 3 – KitKat 2F Mini _Solution.pdf
└── PHASE1/
    ├── CODEBASE.md
    ├── VISION_CODE_ANALYSIS.md
    ├── DELTA_COMPARISON.md          ← IPC + HMI keep/change/add
    ├── Wafer_Inspection_Analysis.pdf
    ├── images/
    └── Missing Wafer Inspection Code/   ← TwinCAT project + .library files
```

## What to read first

1. `PHASE1/DELTA_COMPARISON.md` — what must change on **Vision IPC** and **Vision HMI** vs York/Beckhoff.
2. `PHASE1/VISION_CODE_ANALYSIS.md` — how `ProcessSKU` works, line fitment, sag/fiducial notes.
3. `PHASE1/CODEBASE.md` — extracted Structured Text (readable dump; libraries are still compiled).

Open TwinCAT from:

`PHASE1/Missing Wafer Inspection Code/Missing Wafer Inspection Code/Cust_Nestle_Vanguard/Vanguard_Vision_Inspection/Vanguard_Vision_Inspection.sln`
