# Dual-TMD Optimization for Irregular RC Buildings using Genetic Algorithm
### Nonlinear Seismic Analysis · OpenSeesPy · FEA · Accepted at StructE NATCON 2026

---

## 🏆 Academic Recognition

> **Accepted for Oral Presentation — StructE NATCON 2026**  
> This research was peer-reviewed and selected for oral presentation at the National 
> Conference on Structural Engineering (StructE NATCON 2026).

---

## 🔍 What This Project Does

A complete computational framework for **seismic performance evaluation and 
Dual-TMD optimization** of plan-irregular reinforced concrete buildings — 
across 5, 10, and 15 storey configurations.

The framework automatically designs the structure, optimizes damper placement 
and parameters simultaneously using a Genetic Algorithm, and validates 
performance through nonlinear Time History Analysis.

---

## 🏗️ The Engineering Problem

Plan-irregular buildings are inherently more vulnerable to earthquakes because 
their asymmetric geometry causes **lateral-torsional coupling** — the building 
twists and sways simultaneously, amplifying damage.

Conventional TMD design:
- Uses manual trial-and-error placement
- Optimizes mechanical parameters only
- Ignores torsional interaction effects

**This framework solves all three problems simultaneously.**

---

## ⚙️ Technical Stack

| Tool | Purpose |
|------|---------|
| Python 3.x | Core programming |
| OpenSeesPy | 3D Nonlinear Finite Element Analysis |
| NumPy | Matrix operations |
| Matplotlib | Drift profiles and displacement plots |
| Custom Genetic Algorithm | Global optimization engine |

---

## 📐 Structural Models

- **3 building scales:** 5, 10, and 15 storeys
- **Geometry:** Plan-irregular RC frames (torsionally sensitive)
- **Design standard:** IS 456:2000 (concrete design) + IS 1893:2016 (seismic)
- **Automated design script** calculates beam and column dimensions 
  from input specifications — no manual sizing required
- Full **3D Finite Element Analysis** in OpenSeesPy

---

## 🌍 Ground Motions

Nonlinear Time History Analysis (THA) performed using:
- **Near-Field records** — high intensity, directivity effects, short duration
- **Far-Field records** — lower intensity, longer duration, frequency content variation

Both cases tested across all 3 building scales with and without optimized TMDs.

---

## 🧬 Genetic Algorithm — Global Optimization

The GA simultaneously optimizes **both parameters and placement** of Dual-TMDs:

**Variables optimized:**
- Mass ratio (μ)
- Tuning frequency ratio
- Damping ratio (ξ)
- Physical topological placement on the structure
```
Initialize population → Nonlinear THA simulation for each candidate
→ Evaluate fitness (minimize RMS drift + roof displacement)
→ Selection → Crossover → Mutation → Next generation
→ Converge to optimal [μ, f, ξ, placement]
```

This treats damper design as a **global optimization problem** — 
not sequential manual tuning.

---

## 📊 Key Findings

- ✅ Optimized Dual-TMDs significantly reduce **RMS roof displacement** 
  and **inter-storey drift** under both ground motion types
- ✅ For irregular structures, optimal TMD placement is at 
  **extreme perimeter roof corners** — reducing lateral-torsional coupling
- ✅ Framework outperforms conventional manual TMD design methods
- ✅ Results validated across 5, 10, and 15 storey models

---

## 📁 Project Structure
```
Dual-TMD-Optimization-SeismicAnalysis/
│
├── models/
│   ├── building_5storey.py         # 5-storey irregular RC model
│   ├── building_10storey.py        # 10-storey irregular RC model
│   └── building_15storey.py        # 15-storey irregular RC model
│
├── design/
│   └── auto_design.py              # Automated beam/column sizing script
│
├── optimization/
│   └── genetic_algorithm.py        # GA — parameters + placement optimizer
│
├── analysis/
│   └── nonlinear_tha.py            # Nonlinear Time History Analysis
│
├── ground_motions/
│   ├── near_field/                 # Near-field ground motion records
│   └── far_field/                  # Far-field ground motion records
│
├── results/
│   └── plots/                      # Drift profiles, displacement plots
│
├── main.py                         # Run full framework
└── README.md
```

---

## 🚀 How to Run
```bash
pip install openseespy numpy matplotlib

# Run full optimization for 10-storey building
python main.py --storeys 10 --ground_motion near_field

# Plot results
python results/plot_results.py
```

---

## 📚 Standards & References

- IS 456:2000 — Plain and Reinforced Concrete Code of Practice
- IS 1893:2016 — Criteria for Earthquake Resistant Design of Structures
- OpenSeesPy Documentation

---

## 👩‍💻 Author

**Stuti Singh**  
B.Tech Civil Engineering, SVNIT Surat (2022–2026) | CGPA: 7.87  
Research Intern — IIT Jodhpur (Civil & Infrastructure Engineering)  
📧 sstuti521@gmail.com · [LinkedIn](]www.linkedin.com/in/stuti-singh-20255a253)

---

## 📄 Citation

> Singh, S. (2026). *Dual-TMD Optimization for Seismic Performance of 
> Plan-Irregular RC Buildings using Genetic Algorithm and Nonlinear 
> Time History Analysis.* StructE NATCON 2026 — Oral Presentation.
