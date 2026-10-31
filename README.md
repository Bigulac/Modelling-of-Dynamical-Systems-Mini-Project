# Modelling a 3D Printer – Code and Simulation

Code and simulations for the **Mini Project 2026: "Modelling a 3D printer"** of the course *Modelling of Dynamical Systems - SC42155* (TU Delft).

The project models a Core XY 3D printer as a set of coupled dynamical systems: two DC motors driving a belt mechanism, a heated nozzle, a cooling filament, and the discrete logic that handles filament replacement. Everything is simulated in one Jupyter notebook.

> The written report (LaTeX) contains the derivations. This repository contains only the code that produces the numbers and figures.

---

## Contents

| Question | Topic | What the code does |
|---|---|---|
| **Q1.4** | Drawing shapes | Computes the feedforward motor voltages that make the print head draw an **ellipse** and a **wine bottle outline**, then simulates the full 6-state electromechanical model and compares it with the reference path |
| **Q1.6** | Filament cooling (1D PDE) | Simulates heat conduction and cooling along a filament that grows as it is printed on a heated bed |
| **Q2.1** | Filament replacement FSM | Finite state machine for colour change (`C`) and spool finished (`F`) events, with test run and state diagram |
| **Q2.2** | Parked detection automaton | DFA that detects from motor samples that the head has reached the wipe tower at (0, 0), verified by exhaustive checks |
| **Q3.1** | Hybrid system (ellipse) | Full hybrid model with 9 modes (`q0`–`q8`) combining motors, nozzle, FSM and homing automaton; prints an ellipse with filament replacements |
| **Q3.2** | Hybrid system (square) | Same hybrid model, with the printing mode split into 4 sub-modes (`q0.1`–`q0.4`) to print a square |

---

## Repository structure

```
├── Code and Simulation of Mini Project.ipynb   # all code, in question order
└── LICENSE
└── README.md

```

---

## Getting started

### Run in Google Colab
1. Open the notebook in Colab (replace `<username>/<repo>` with your GitHub path):
   `https://colab.research.google.com/github/<username>/<repo>/blob/main/Modelling_of_Dynamical_Systems.ipynb`
2. Choose **Runtime → Run all**.

### Run locally
```bash
git clone https://github.com/<username>/<repo>.git
cd <repo>
pip install numpy scipy sympy matplotlib jupyter
jupyter notebook Modelling_of_Dynamical_Systems.ipynb
```

**Requirements:** Python 3.9+, `numpy`, `scipy`, `sympy`, `matplotlib`.

> **Run the cells top to bottom.** Later sections reuse and redefine names from earlier ones (for example `pos`, `NAMES`, `inputs`, `guards`, `ext_q3`, `T_END`). In particular, Q3.2 overrides several Q3.1 definitions, so running it first or out of order will give wrong results.

---

## Model overview

### Mechanics and motors (Q1.4, Q3)
The head position follows from the two motor angles (Core XY kinematics):

```
x = r_m (θ_l + θ_r),    y = r_m (θ_l − θ_r)
```

Each motor is a DC motor with armature inductance, giving a 6-state model: motor angles, angular velocities and currents (`θ_l, θ_r, ω_l, ω_r, i_l, i_r`). Friction and load coupling are collected in the constants `a`, `b_x`, `b_y`, `c`.

Voltages are computed by **feedforward**. The reference path is differentiated symbolically (SymPy) up to the third derivative and mapped to the exact voltage that produces it. The path is parametrised with a smooth quintic time scaling, so speed and acceleration are zero at the start and end.

### Nozzle (Q3.1)
Three extra states: nozzle capacitor voltage `v_C`, inductor current `i_N`, and nozzle temperature `T_n`. The heater is on (`U_ON = 24 V`) or off depending on the mode.

### Filament cooling (Q1.6)
1D heat equation along the printed filament, solved with an explicit finite-difference scheme:
- axial conduction along the filament,
- convection to air on three faces and contact cooling to the hotbed on the bottom face,
- the nozzle temperature is imposed at the moving printing front,
- insulated boundaries at the path start and end.

The code includes a sanity check against the exact solution without conduction (largest difference ≈ 0.43 K) and a grid refinement table over `ds` and `dt`. Reported results: cooling to 60 °C after roughly 12.5 s and temperature uniform within 1 K after roughly 19.1 s (see the notebook output).

### Discrete logic (Q2.1, Q2.2)
- **Filament FSM:** `PRINTING → STOP → MOVE → WIPE → NOZZLE OFF → LOAD → NOZZLE ON → RESUME → PRINTING`. Events `C` and `F` are only accepted in `PRINTING` and ignored everywhere else. If both arrive together, only one transition is taken.
- **Parked-detection DFA:** reads the symbols `(H,+)`, `(H,0)`, `(V,+)`, `(V,0)` (axis, moving or at rest) and accepts when both walls have been reached. It is checked against a reference implementation on all 87 381 input sequences up to length 8 (0 mismatches), and against a simulated head on a grid (no accepted run ends away from the origin).

### Hybrid system (Q3.1, Q3.2)
| Mode | Name | Mode | Name |
|---|---|---|---|
| `q0` | PRINTING | `q5` | LOAD |
| `q1` | STOP & SAVE | `q6` | NOZZLE ON |
| `q2` | MOVE (homing) | `q7` | RESUME |
| `q3` | WIPE | `q8` | IDLE |
| `q4` | NOZZLE OFF | | |

- Each mode has its own continuous dynamics and input.
- Mode changes are triggered by **guards**, implemented as `solve_ivp` event functions (Radau solver).
- **Reset maps** put the motors at rest and store the interrupted position and path progress `(x_p, y_p)`, so printing can resume exactly where it stopped.
- `MOVE` uses the DFA from Q2.2 for homing: move down until the bottom is hit, then left until the left side is hit, using sampled sensor symbols and a short hold-off time.
- In Q3.2 the square is printed as four sub-modes `q0.1`–`q0.4`, one per side.

Main parameters (editable at the top of Q3.1): `T_min = 180 °C`, `omega_max = 200 rad/s`, printing area 200 × 200 mm, timers `t_save`, `t_wipe`, `t_load`, and the external event list `ext_q3 = [(5.0,'C'), (14.0,'F'), (63.0,'F')]`.

---

## Output figures

The notebook saves these figures (200 dpi PNG) in the working directory:

| File | Shows |
|---|---|
| `fig_paths.png` | Q1.4: reference vs simulated path for ellipse and wine bottle |
| `fig_q3_system_diagram.png` | Transition diagram of the hybrid system (`q0`–`q8`) |
| `fig_q3_modes.png` | Mode sequence over time with accepted events `C`, `F` |
| `fig_q3_path.png` | Print head path: printing, homing, resume |
| `fig_q3_nozzle.png` | Nozzle temperature and heater voltage |
| `fig_q3_motors.png` | Motor speeds, currents and voltages around the replacements |
| `fig_q3_homing.png` | Homing with the automaton state and input symbols |
| `fig_square_modes.png`, `fig_square_path.png` | Q3.2 equivalents for the square |

The state diagrams and plots for Q1.6, Q2.1 and Q2.2 are also generated in their own cells.

---

## Notes and known limitations

- The Q3.1 check that the motor angles stay in `[0, 2π]` (an invariant stated in the report) is **violated** during part of the run. The angles are accumulated rather than wrapped, so the printed message `VIOLATED` is expected. The other checks (speed limit, printing area, nozzle temperature per mode, motors at rest, path on the ellipse) pass.
- Q1.6 uses an explicit Euler scheme, so `dt` must stay small relative to `ds²/α`. The default values `ds = 0.5 mm`, `dt = 5 ms` are stable.
- The model is deliberately simple as per the questions in the assignment(lumped nozzle, 1D filament, ideal belt). See the report for the assumptions.

---

## Author
Marko Bingulac, 6730531 , MSc Systems and Control, TU Delft

Saurav Kishor Avachat, 6654762 , MSc Systems and Control, TU Delft

Course: Modelling of Dynamical Systems, Mini Project 2026
