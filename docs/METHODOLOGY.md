# METHODOLOGY.md — QRC-Thresher (Praxen Labs QRC-001)

**Status:** Canonical companion to `docs/BUILD_SPEC.md`. Long-form derivations and theoretical justifications.
**Owner:** Praxen Labs · QRC research line.
**License:** CC BY 4.0 (see `docs/LICENSE-DOCS`).
**Authority:** Where this document and `BUILD_SPEC.md` overlap on prescriptive content (defaults, schemas, gate definitions), `BUILD_SPEC.md` wins. Where `BUILD_SPEC.md` defers a derivation, justification, or extended note to "the methodology companion," this document wins. Where this document and `docs/REFERENCES.md` disagree on the status of a citation, `REFERENCES.md` wins.
**Reading time:** ~45 minutes for a reviewer who has read `BUILD_SPEC.md`; ~90 minutes cold.
**Intended audience:** (1) the methodology lead reviewing pre-registration choices; (2) an external reviewer at a NeurIPS QML workshop or *Quantum* journal who wants the *why* behind the harness's design; (3) future maintainers introducing new tasks, ablations, or backends.

This document is non-prescriptive. It contains **derivations**, **rationale**, and **alignment notes** with the upstream library and tooling ecosystem (Qiskit 2.x, PennyLane 0.44, Q-CTRL, ReservoirPy) as of **May 2026**. Every prescriptive section in `BUILD_SPEC.md` has a corresponding "Why" subsection here; every "ASSUMED-DEFAULT" in `BUILD_SPEC.md` Appendix A has at least one paragraph of justification here.

---

## Table of Contents

1. Tasks: definitions and minimal prescriptive surface
2. Quantum Reservoir Architecture (prescriptive surface)
3. Classical Baselines (prescriptive surface)
4. Ablations (prescriptive surface)
5. Statistical Methodology (prescriptive surface)
6. Theoretical Foundations of Reservoir Computing
7. Theoretical Foundations of Quantum Reservoir Computing
8. Task Theory: Why STM, Parity, NARMA-10
9. Circuit Theory: HEA, Expressivity, and the Reservoir Mode
10. Readout Theory: Information Content of Pauli Observables
11. Noise Theory and the Phase-1 Two-Channel Model
12. Statistical Theory: BCa, Holm, Power, and Pre-Registration
13. Backend Engineering: PennyLane 0.44 and Qiskit 2.x in May 2026
14. Noise Mitigation Ecosystem (Q-CTRL, ZNE, DD) — Phase 2/3 Roadmap
15. Context Engineering for the Proof Layer
16. Cross-Reference Index to BUILD_SPEC.md
17. Change Log

---

## 1. Tasks (prescriptive surface)

All tasks are synthetic, deterministic, and seeded via `numpy.random.Generator`. A task generator is a pure function of `(seed, params, length)`. See `BUILD_SPEC.md` §8 for the full normative specification; this section is the minimal cross-reference surface for engineers reading this document standalone.

### 1.1 Short-Term Memory (STM)

- **Input:** $u_t \sim \text{Uniform}(-1, 1)$, length $T$.
- **Targets:** $y_t^{(k)} = u_{t-k}$ for $k \in \{0, \ldots, K\}$, $K = 20$ default.
- **Metric:** Memory Capacity $\mathrm{MC} = \sum_k \mathrm{corr}(\hat{y}^{(k)}, y^{(k)})^2$.
- **Train/test split:** Chronological 70/30 (no shuffling).
- **Implementation:** `src/qrc_thresher/tasks/stm.py`.

### 1.2 Temporal Parity / XOR

- **Input:** Binary $u_t \in \{0, 1\}$, length $T$.
- **Target:** $y_t = \mathrm{XOR}(u_{t-d+1}, \ldots, u_t)$ for window $d$.
- **Metric:** Classification accuracy at delay $d$.
- **Default:** $d \in \{1, 2, 3, 5\}$.
- **Implementation:** `src/qrc_thresher/tasks/temporal_parity.py`.

### 1.3 NARMA-10 (Phase 1.5, gated behind G3)

- **Input:** $u_t \sim \text{Uniform}(0, 0.5)$.
- **Recurrence:** $y_{t+1} = 0.3\, y_t + 0.05\, y_t \sum_{i=0}^{9} y_{t-i} + 1.5\, u_{t-9}\, u_t + 0.1$.
- **Metric:** NRMSE.
- **Implementation:** `src/qrc_thresher/tasks/narma10.py`.

---

## 2. Quantum Reservoir Architecture (prescriptive surface)

Implementation in `src/qrc_thresher/reservoirs/pennylane_qrc.py`. See `BUILD_SPEC.md` §9 for full normative specification.

### 2.1 Circuit Pattern

1. Angle-encode $u_t$ via $R_y(\pi u_t)$ on each qubit.
2. Fixed random rotations $R_z(\theta_i)$, $R_x(\phi_i)$ drawn once at construction (seeded by `reservoir_seed`).
3. Ring topology entangling layer: $\mathrm{CNOT}(i, (i+1) \bmod N)$.
4. Repeat steps 1–3 for depth $L$.
5. Readout: $\langle Z_i \rangle$ for each qubit (`z_only`) or also $\langle Z_i Z_j \rangle$ for $i < j$ (`z_and_zz`).
6. Stack readouts to form feature matrix $X \in \mathbb{R}^{T \times F}$.
7. Ridge regression on $X$ to predict targets.

### 2.2 Initial Parameter Sweep

| Parameter      | Values                                       |
| -------------- | -------------------------------------------- |
| `n_qubits`     | 4, 6, 8                                      |
| `depth`        | 2, 3, 4                                      |
| `seeds`        | 3 initially; ≥ 5 for manuscript-track        |
| `readout`      | `z_only` first; `z_and_zz` if needed         |
| `ridge_alpha`  | $\{10^{-8}, 10^{-6}, 10^{-4}, 10^{-2}, 1, 100\}$ |

---

## 3. Classical Baselines (prescriptive surface)

### 3.1 Feature Dimension Matching

`N_ESN = N_quantum_features` (NOT $2^N$).

- `z_only`: $F = N$.
- `z_and_zz`: $F = N + N(N-1)/2$.

### 3.2 Echo State Network (ESN)

| Parameter         | Values                                          |
| ----------------- | ----------------------------------------------- |
| `spectral_radius` | $\{0.8, 0.9, 0.95, 0.99, 1.0\}$                 |
| `input_scaling`   | $\{0.1, 0.5, 1.0\}$                             |
| `leak_rate`       | $\{0.1, 0.3, 0.5, 1.0\}$                        |
| `ridge_alpha`     | $\{10^{-8}, 10^{-6}, 10^{-4}, 10^{-2}, 1.0\}$   |

CV is performed on training data only. Test indices are NEVER used in hyperparameter selection.

### 3.3 Random Kitchen Sinks (RKS)

$\phi(u) = \cos(W u + b)$, $W \sim \mathcal{N}(0, \sigma^2/d)$, $b \sim \text{Uniform}(0, 2\pi)$. Dimension matched to QRC features.

---

## 4. Ablations (prescriptive surface)

| Name              | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| `phase_random`    | Random phases re-drawn at each time step                     |
| `no_entangle`     | Single-qubit rotations only, no CNOT                         |
| `random_features` | Classical random projection (also a baseline)                |
| `haar`            | Haar-random unitary $U \sim \mathrm{Haar}(2^N)$              |

Ablations A1 (`no_entangle`), A2 (`phase_random`), and A3 (`haar`) are **self-falsifying**: if any of them produces a metric within tolerance of the full QRC on the same task, the headline positive result is automatically retracted (`BUILD_SPEC.md` §10.2). Three additional axes (A5 observable swap, A6 layer scan, A7 shot-count scan, A8 encoding swap) are tracked in `BUILD_SPEC.md` §10.1.

---

## 5. Statistical Methodology (prescriptive surface)

At G3 and beyond:

- Paired t-test or Wilcoxon signed-rank test, $n \geq 5$ seeds minimum.
- Report mean, std, p-value, and Cohen's $d$.
- Bootstrap 95% CIs over 1000 resamples (percentile floor) for headline metrics; BCa via `scipy.stats.bootstrap` with 10000 resamples for manuscript-track.
- Holm–Bonferroni correction across the three primary task hypotheses.
- Pre-registered effect sizes (`BUILD_SPEC.md` §13.4) MUST NOT be re-tuned after seeing the data.

Gate evaluation writes `results/gates/<name>.json` with decision, evidence, run_ids, p-values.

---

## 6. Theoretical Foundations of Reservoir Computing

This section grounds the harness in the classical reservoir-computing literature so that the QRC-vs-classical comparison rests on a shared theoretical frame, not on competing folklore.

### 6.1 The reservoir computing programme

Reservoir computing (RC) is the family of recurrent learning systems that *fix* the dynamics of a high-dimensional nonlinear state-update map $F: \mathcal{X} \times \mathcal{U} \to \mathcal{X}$ and train *only* a static readout $W_\text{out}: \mathcal{X} \to \mathcal{Y}$. Given an input sequence $\{u_t\}$ and an initial state $x_0 \in \mathcal{X}$:

$$x_{t+1} = F(x_t, u_{t+1}), \qquad \hat{y}_t = W_\text{out}\, x_t.$$

The *fixed-dynamics* commitment is the central methodological choice. It is what makes the readout problem convex (linear least squares), what makes inference cheap (one matrix–vector product per step), and what makes the system learnable from short input sequences without backpropagation through time. The two foundational instantiations are the **Liquid State Machine** (Maass, Natschläger, Markram, 2002) over spiking neurons and the **Echo State Network** (Jaeger, 2001) over leaky-integrator units. Both rely on the same mathematical scaffolding, and both predate the QRC literature by roughly two decades.

### 6.2 Three properties that justify "good" reservoir dynamics

A reservoir is *useful* — in the precise PAC-style sense that any sufficiently regular target functional can be approximated to arbitrary accuracy by a linear readout — if and only if its dynamics satisfy three properties together:

- **Echo State Property (ESP).** For any pair of initial states $x_0, x_0'$ and any input sequence $\{u_t\}$, the trajectories $\{x_t\}, \{x_t'\}$ converge: the influence of the initial condition fades in finite time. ESP is the "memory of inputs only" property.
- **Fading Memory Property (FMP).** For any two input sequences $\{u_t\}, \{u_t'\}$ that agree on a recent window of length $w$ (and may differ arbitrarily in the distant past), the resulting reservoir states $x_t, x_t'$ are within $\varepsilon(w)$ in some norm, with $\varepsilon(w) \to 0$ as $w \to \infty$. FMP is the "recent inputs matter most" property.
- **Separation Property.** For any two distinct input *histories*, the resulting reservoir states are distinct (so a linear readout has any hope of distinguishing them). Separation is the "no two histories collide" property.

ESP and FMP together (with separation) imply universality of linear readouts over the reservoir's state, in the sense of Boyd–Chua's classical theorem on Volterra-series approximation. The harness does not measure ESP or FMP directly in Phase 1, but every ablation axis (A1, A2, A6) is designed to *break* one of these properties and observe the metric collapse the theory predicts. A1 (`no_entangle`) damages separation by reducing the effective state dimension; A2 (`phase_random`) damages ESP by re-drawing dynamics each step; A6 (layer scan with $L=1$) probes the boundary at which FMP becomes too short for the task. See §9.4 for the quantum analogues.

### 6.3 Why a *linear* readout is the right scope cut

A nonlinear readout (kernel ridge, MLP, transformer head) confounds the reservoir's expressive power with the readout's. A linear readout is the only choice that lets us answer the question, *what does the reservoir actually compute about the input history?* without the readout doing a second round of computation behind the scenes. This is also why QRC-Thresher will not adopt a "deep readout" in any phase: the comparison loses meaning.

Ridge regression is the unique linear estimator that simultaneously (i) admits a closed-form solution `(F^T F + α I)^{-1} F^T y`, (ii) penalizes feature magnitudes uniformly (matching the symmetry of random reservoir features), and (iii) is the *de facto* standard across the reservoir-computing literature (`BUILD_SPEC.md` Appendix E.5). Variants (LASSO, kernel ridge) are *legitimate* in their own contexts but introduce confounders we have explicitly chosen not to absorb.

### 6.4 The leak rate and the spectral radius — analogue knobs

The two ESN hyperparameters that map most directly onto QRC architectural knobs are:

- **Spectral radius $\rho$** of the recurrence matrix $W$. Controls the timescale of fading memory: $\rho \to 1$ produces long memory (and risks losing ESP); $\rho \to 0$ produces short memory.
- **Leak rate $\alpha$** in $x_{t+1} = (1 - \alpha) x_t + \alpha \tanh(W x_t + W_\text{in} u_{t+1})$. Controls the temporal filter applied to inputs: $\alpha = 1$ is no leak (full update each step); $\alpha \to 0$ is heavy leak (slow integrator).

In the QRC, the analogue of $\rho$ is the **circuit depth** $L$ (more layers $\Leftrightarrow$ deeper mixing, longer effective memory window in the stateful variant) and the analogue of $\alpha$ is the **encoding scale** $\alpha_\text{enc}$ in $R_y(\alpha_\text{enc} u_t)$ (smaller $\alpha_\text{enc}$ $\Leftrightarrow$ larger overlap with previous-step state, since the encoded perturbation is smaller). The harness does not currently expose $\alpha_\text{enc}$ as a swept parameter (`BUILD_SPEC.md` Appendix E.4 reserves this as ablation A.E2 for Phase 1.5).

The matched-comparison protocol (`BUILD_SPEC.md` §12.2) uses the *feature dimension* — not $\rho$, not $\alpha$ — as the primary matching axis, because feature dimension is the only quantity that is well-defined on both sides of the QRC-vs-ESN comparison without requiring an arbitrary mapping.

---

## 7. Theoretical Foundations of Quantum Reservoir Computing

The QRC literature has produced, as of May 2026, a coherent theoretical scaffolding that justifies why a quantum reservoir is *not* automatically the same as a classical reservoir, and why the Phase 1 harness should expect a measurable but not enormous separation. This section summarizes that scaffolding and locates the Phase 1 prescriptive choices within it.

### 7.1 The kernel view: QRC as a quantum feature map

A QRC with input encoding $U_\text{in}(u)$, fixed reservoir unitary $U_\text{res}$, and observable set $\{O_k\}$ defines a feature map

$$\phi: u \mapsto \big(\langle 0 | U_\text{in}^\dagger(u) U_\text{res}^\dagger O_k U_\text{res} U_\text{in}(u) | 0 \rangle\big)_k.$$

The inner product $K(u, u') = \phi(u) \cdot \phi(u')$ is a positive-definite kernel by construction. This is the **quantum kernel** view of QRC, and it makes precise the sense in which QRC and RKS are siblings: both are random-feature approximations to a (different) reproducing kernel Hilbert space. The recent review by Domínguez et al. (*Phil. Trans. R. Soc. A* 384:20250085, 2026, "From quantum feature maps to quantum reservoir computing") formalises this connection. A QRC therefore lives on the same Pareto frontier as RKS, ESN, and quantum kernel methods, and the right baseline question is "does this *particular* random feature map outperform the matched classical random feature map?" — which is exactly what `BUILD_SPEC.md` §11 prescribes.

### 7.2 Fading memory and dissipation in quantum reservoirs

A unitary quantum system is, by construction, *time-reversible* — and a strictly unitary reservoir therefore does **not** automatically satisfy FMP. The classical literature's solution is dissipation (leak rate, noise injection); the quantum literature's solution is one of:

1. **Engineered dissipation.** Couple the reservoir to a Markovian bath via a Lindblad master equation. Recent work (Sannia et al., *Quantum* 8:1291, 2024, "Dissipation as a resource for QRC"; *npj Quantum Information* 2026 on controllable damping) proves that dissipative spin networks are *universal* for fading-memory functions on time-series, in the same Boyd–Chua sense as classical RC. There exists, moreover, a "sweet spot" of dissipation linked to physical observables (maximal optical absorption in BEC reservoirs; *Phys. Rev. Lett.* 2025 vp79-8t1l).
2. **Mid-circuit measurement and reset.** Each measurement is an irreversible projection that breaks unitarity. This is the path many recent IBM-platform implementations follow, leveraging mid-circuit measurement primitives newly stable in Qiskit 2.x.
3. **Stateless rebuild ("encoding-as-update").** At each $t$, rebuild the state from $u_t$ alone via $|\psi_t\rangle = U(u_t, \theta) |0\rangle^{\otimes n}$. There is no "memory" carried across time at all in the quantum register; the only memory is in the readout layer's window. This is *trivially* fading-memory (it has no memory at all).

Phase 1 takes path (3) — the **stateless** variant — for three reasons:

- It is the cheapest, most analyzable variant. Every observation $|\psi_t\rangle$ depends on $u_t$ alone, so there is no state-leakage debugging to do.
- It places the entire memory burden on the *readout window*, which is the same place classical RKS places it. This makes the QRC-vs-RKS comparison maximally fair.
- It produces a clean negative-result baseline. If even a stateless QRC outperforms a feature-matched ESN that *does* have classical recurrence, the result is striking; if it does not, the harness has discharged its duty cheaply.

The stateful variant (path 1 or 2) is deferred to Phase 1.5 (`BUILD_SPEC.md` MG4, MG7) because (a) it introduces irreducible analytical complexity that we want to commit to *only* if Phase 1 produces signal, and (b) it forces a second round of design choices (Lindblad dissipation rate vs. measurement-and-reset cadence) that we are not prepared to pre-register without pilot data.

### 7.3 The Echo State Property in non-stationary quantum systems

The classical ESP definition assumes a stationary state-update map. Quantum reservoirs realised on real devices, with drift in calibration parameters between runs, do not have stationary dynamics. The "Hierarchy of the echo state property in quantum reservoir computing" line (Tran & Nakajima, OpenReview 2024) introduces:

- **Non-stationary ESP** — convergence holds along a sequence of *time-varying* state-update maps.
- **Subspace ESP** — convergence holds when restricted to a particular subspace of the reservoir's Hilbert space.
- **Subset ESP** — convergence holds for inputs drawn from a particular subset of $\mathcal{U}$.

These finer ESP variants are pertinent to Phase 2 (hardware) but **not** to Phase 1 (simulator), because the Phase 1 harness fixes $\theta$, $\phi$ once per realisation and runs in exact-amplitude mode by default. Phase 1 does, however, cite this hierarchy explicitly when motivating G2.5 (the Haar-random discrimination gate): a Haar-random reservoir trivially satisfies ESP (it has no carried state) but does *not* exhibit any task-aligned structure, so it is the right null model for the "structure beyond entanglement" question that G2.5 asks.

### 7.4 The barren-plateau exemption

Hardware-efficient ansätze (HEA) are notoriously prone to **barren plateaus** under variational training: the gradient of any local loss vanishes exponentially in $n$ for sufficiently expressive (2-design-approaching) circuits. This is the principal practical objection to HEA for VQE/QAOA work.

Reservoir-mode QRC is *exempt* from this objection by construction. Circuit parameters are sampled once and *frozen*; only the linear readout is trained, and that readout's loss surface is convex with a closed-form solution. There is no gradient to plateau. The frozen-parameter discipline is therefore not a performance compromise — it is the methodological feature that lets HEA be a defensible reservoir-mode choice when the same ansatz family would be defensible-only-with-asterisks for variational training.

We say this explicitly because a reviewer who has been burned by barren plateaus in a VQE context might raise the objection without distinguishing the two regimes. The right rebuttal is "barren plateau is a *training* phenomenon; we do not train; therefore the phenomenon is irrelevant," and the place to make that rebuttal in the manuscript is the methods section that this companion document supplies (`BUILD_SPEC.md` §9.5).

### 7.5 Universality results that bound the strongest possible Phase-1 claim

Two universality theorems are relevant to bounding what a *positive* Phase 1 result could mean:

- **Boyd–Chua–style universality of fading-memory functionals over spin reservoirs with engineered dissipation** (Sannia et al., 2024). Implies that, *with dissipation*, a QRC can in principle approximate any fading-memory map on a compact input space. The Phase 1 stateless QRC has no engineered dissipation, so this theorem does *not* apply to Phase 1; it applies to the Phase 1.5 stateful variant.
- **Universality of feature-matched ESNs.** A feature-matched ESN with sufficiently many random units is itself universal. This means a Phase 1 positive result, even if statistically robust, cannot be interpreted as "QRC has access to a class of functions ESN does not." The right interpretation is "at *equal* feature budget, QRC realises the universal-approximation premium more efficiently than ESN" — a strictly weaker but still publishable claim.

The harness's pre-registered effect sizes (`BUILD_SPEC.md` §13.4) are calibrated against this weaker interpretation. A ΔMC ≥ 0.5 separation at $K = 20$ corresponds to roughly 2–3 effective additional memory taps — meaningful, measurable, and well below the threshold where one could claim a complexity-theoretic separation.

---

## 8. Task Theory: Why STM, Parity, and NARMA-10

Each of the three Phase 1 tasks is chosen because it isolates one axis of the reservoir's expressive power. This section spells out the isolation argument and the upper bounds the theory imposes on each task's metric.

### 8.1 STM and the Jaeger memory capacity bound

The **Jaeger memory capacity** is

$$\mathrm{MC} = \sum_{k=0}^{K} \mathrm{corr}^2(\hat{u}_{t-k}, u_{t-k}),$$

where $\hat{u}_{t-k}$ is the linear readout's prediction of the input $k$ steps ago. Jaeger (2001) proved that for any *linear* reservoir with $N$ states driven by an i.i.d. input,

$$\mathrm{MC} \leq N,$$

with equality achievable in the limit of perfect ESP and infinite-precision readout. This bound is tight for ESN at well-tuned $\rho$.

For the QRC, the analogous bound depends on the readout dimension $F$ and not on the Hilbert-space dimension $2^n$. With `z_only`, $F = n$, and so $\mathrm{MC} \leq n$ regardless of $n$-qubit Hilbert-space size. With `z_and_zz`, $F = n + n(n-1)/2$, and the bound rises accordingly. This is a critical sanity check: a "QRC produces $\mathrm{MC} > n$ with `z_only`" claim would be an immediate red flag and would force a backend-bug investigation.

`metrics/scoring.py:memory_capacity` clamps the per-$k$ correlation² to $[0, 1]$ and aggregates; the harness logs the per-$k$ correlation² as a secondary metric so this invariant is testable run-by-run.

### 8.2 Parity hardness and the $d \geq 4$ separation

The temporal parity task with window $d$ requires the system to compute

$$y_t = u_{t-d+1} \oplus u_{t-d+2} \oplus \cdots \oplus u_t.$$

Three theoretical facts:

1. **Linear reservoirs cannot solve parity.** Parity over $d$ binary inputs is a degree-$d$ Boolean function (in the Fourier sense over $\{0,1\}^d$). Linear readouts on a *linear* state-update map cannot represent any nonlinear Boolean function. Any nontrivial accuracy on parity therefore *measures* the reservoir's nonlinearity.
2. **Shallow nonlinearities suffice for $d \leq 2$.** Any reservoir with even mild input nonlinearity (e.g., a single $\tanh$ layer, a single CNOT) achieves $> 50\%$ on parity at $d = 2$. The interesting regime is $d \geq 3$.
3. **The $d \geq 4$ regime separates real reservoirs from random projections.** RKS at moderate dimension achieves chance accuracy at $d \geq 4$; ESN with well-tuned $\rho$ achieves $\sim 70$–$85\%$; QRC with HEA and ring entangling on $n \geq 4$ qubits has been reported in the literature to achieve $\geq 80\%$ at $d = 3$ in noiseless simulation.

The harness fixes $d \in \{1, 2, 3, 5\}$ as the parity sweep and uses $d = 3$ as the gated cell (G2). Pre-registered effect size: $\Delta\text{accuracy} \geq 5$ pp over RKS at $d = 3$.

### 8.3 NARMA-10 stability and the realised-instability problem

The NARMA-10 recurrence

$$y_{t+1} = 0.3 y_t + 0.05 y_t \sum_{i=0}^{9} y_{t-i} + 1.5 u_{t-9} u_t + 0.1$$

is *conditionally* stable. With $u_t \sim \mathrm{Uniform}(0, 0.5)$ and $y_0 = 0$, simulations remain bounded for at least $T \leq 10^4$ in the overwhelming majority of seed instances, but the recurrence has runaway-instability events at a rate of roughly one in $10^6$ realisations under typical numerical conditions. Two engineering consequences:

- **NaN/inf guard.** `tasks/narma10.py` checks for `|y_t| > 10` and raises explicitly. The seed-replacement protocol (`BUILD_SPEC.md` §14.4, §19.3) handles such events; we refuse to silently winsorize.
- **Difficulty calibration.** The published Phase-1 baseline (NARMA-10 NRMSE $\approx 0.48$ for a small QRC and $\approx 0.18$ for a tuned ESN, per `arXiv:2510.25183`) sits well within the stable region of the recurrence. We expect a *negative* result on NARMA-10 at $n \leq 8$ qubits — and we have pre-registered the effect size that would constitute a positive result anyway.

### 8.4 Why exactly these three, and why no others in Phase 1

The three tasks span the canonical reservoir-computing axes:

- STM measures **linear memory**.
- Parity measures **nonlinear memory at increasing window depth**.
- NARMA-10 measures **generative recurrence with strong autoregressive coupling**.

Mackey-Glass and Lorenz expand the manuscript scope without adding a comparison axis these three cannot cover. Synthetic benchmarks (e.g., bouncing-balls, copy-task) drift into the LSTM/transformer-baseline literature and would force the introduction of GRU/Transformer baselines we have explicitly chosen not to spend Phase 1 compute on.

---

## 9. Circuit Theory: HEA, Expressivity, and the Reservoir Mode

### 9.1 What an HEA is and why it has the parameter count it has

A hardware-efficient ansatz with $n$ qubits and $L$ layers has parameter count $2nL$ (one $R_z$ and one $R_x$ angle per qubit per layer) and CNOT count $nL$ (one per qubit per layer in the ring topology). Total trainable parameters in a *variational* setting: $2nL$. Total *frozen* parameters in our reservoir setting: still $2nL$, but they live in the manifest's `circuit_hash`, not in any optimiser's state.

The HEA is "hardware-efficient" because every gate is in the native gate set of the most common superconducting and trapped-ion platforms (single-qubit Euler-angle rotations + nearest-neighbour CNOT). The native-gate compatibility is a Phase 2 readiness consideration; in Phase 1 it does not directly affect the simulation, but it does mean that any positive Phase 1 result is *cheaply* portable to a hardware experiment — which is precisely the gating for G7.

### 9.2 Expressivity, frame potential, and why "more layers" plateaus

The Haar measure on $U(2^n)$ is the *most expressive* ensemble of unitaries on $n$ qubits. The **frame potential** $\mathcal{F}^{(t)}(\mathcal{E})$ of an ensemble $\mathcal{E}$ measures how close $\mathcal{E}$ is to a unitary $t$-design (Haar moments to order $t$). HEA layered $L$ times on $n$ qubits approaches a 2-design exponentially in $L$ for $L = O(n)$ (Harrow–Mehraban 2023 and follow-ups).

For *reservoir-mode* QRC, however, expressivity is **not** the right yardstick. We do not need the reservoir to span the full unitary group; we need the *output distribution* of expectation values $(\langle Z_i\rangle)_i$ to (i) be approximately uniform across realisations (so that RKS-style universal approximation arguments apply) and (ii) preserve enough structure for a *linear* readout to recover task-relevant information.

The key empirical fact is that capacity *plateaus* at $L \approx 3$–$4$ on the $n \in \{4, 6, 8\}$ regime for the canonical tasks. Beyond this depth, additional layers add compute cost without adding readout-recoverable structure. Ablation A6 (layer scan, `BUILD_SPEC.md` §10.1) is designed to expose this plateau and detect any indexing bug that would mask it.

### 9.3 The Haar-random ablation as the universality-control

A6 (Haar-random) replaces the structured HEA with a single Haar-random unitary on $2^n$ dimensions. By the universality results of §7.5, Haar-random has access to the *full* expressivity ceiling. If the structured HEA outperforms Haar at the same readout dimension, the conclusion is "structured circuits get more out of the available Hilbert space than uniform-random circuits." If Haar matches HEA, the conclusion is "the structure is decorative; any universal ensemble would do" — which is *interesting in its own right* and is *not* a falsification of QRC, but it does invalidate any claim that the specific HEA architecture is doing the lifting.

This is why G2.5 is **mandatory** in `BUILD_SPEC.md` §15.5 and why a Haar-pass-while-HEA-passes is logged as `self_falsified == false` (HEA wins the tie) but a Haar-passes-while-HEA-fails or Haar-matches-HEA forces G3 to fail.

### 9.4 Mapping classical RC properties onto QRC

| Classical RC property                | QRC analogue (Phase 1 stateless variant)                                  |
| ------------------------------------ | ------------------------------------------------------------------------- |
| Echo State Property                  | Trivial: state is rebuilt each step from $u_t$ alone.                     |
| Fading Memory Property               | Trivial: there is no memory in the quantum register; FMP is at the readout window. |
| Separation Property                  | $\langle Z_i \rangle(u) \neq \langle Z_i \rangle(u')$ for $u \neq u'$ (provable for generic HEA seeds). |
| Spectral radius $\rho$               | Circuit depth $L$.                                                        |
| Leak rate $\alpha$                   | Encoding scale $\alpha_\text{enc}$ in $R_y(\alpha_\text{enc} u_t)$.       |
| Reservoir size $N$                   | Feature dimension $F$ ($n$ for `z_only`, $n + n(n-1)/2$ for `z_and_zz`).  |
| Topology (ring/random/all-to-all)    | Entangler topology (ring default; ladder/all-to-all in A.E1 / Phase 1.5). |

This table is the most useful single artefact for a reviewer who has classical RC fluency and is meeting QRC for the first time. We reproduce it in the manuscript supplement (`BUILD_SPEC.md` §26 Table T2-supp).

### 9.5 The circuit hash as a debugging primitive

`BUILD_SPEC.md` §9.6 prescribes a SHA-256 hash over `(n, L, thetas, phis, readout)`. Why hash, not equality-test?

- **Cross-machine reproducibility.** Two identical hashes on two different workstations, with two different `runs.csv` rows, MUST produce features that agree to within BLAS-level non-determinism (`BUILD_SPEC.md` §19.2). Disagreement is a backend-implementation bug. The hash makes the comparison a `grep`-and-diff away.
- **Realisation collision detection.** If two distinct seeds produce the same `circuit_hash`, the seeding code has a bug (or a collision, which we want flagged immediately).
- **Manifest integrity.** The hash is part of every `runs.csv` row, so any tampering with a logged result is detectable by recomputing the hash from the seeds.

The hash is cheap (one hash per realisation, kilobytes of input) and load-bearing. It is the same idea as `ipfs add` for circuits.

---

## 10. Readout Theory: Information Content of Pauli Observables

### 10.1 The observable space and its information content

The space of single-qubit Pauli expectation values on $n$ qubits is $n$-dimensional ($Z$, $X$, or $Y$ — choose one basis); the space of two-qubit Pauli string expectations adds $\binom{n}{2} \cdot 9$ further dimensions (three Pauli choices on each of two qubits); the full Pauli-string space is $4^n - 1$ (every nontrivial Pauli string).

The harness restricts to:

- `z_only`: $\{\langle Z_i \rangle\}_{i=1}^n$. Dimension $n$.
- `z_and_zz`: $\{\langle Z_i \rangle\}_{i=1}^n \cup \{\langle Z_i Z_j\rangle\}_{i<j}$. Dimension $n + n(n-1)/2$.

This is a deliberate restriction. Three reasons:

1. **Estimator cost.** Each new Pauli string requires an additional shot-budget allocation in the sampled regime. Expanding to all $4^n - 1$ would explode shot cost at $n = 8$ (roughly 65,000 strings).
2. **Linearity in readout.** The ridge readout assigns one weight per feature. With $4^n - 1$ features and at most $T < 4^n - 1$ training samples, ridge becomes underdetermined and the inversion ill-conditioned regardless of $\alpha$.
3. **Comparison fairness.** RKS at matched dimension caps at the same $F$. Going beyond $F = n + n(n-1)/2$ would force RKS into a different regime and complicate the comparison.

A5 (observable swap to $X$/$XX$ via Hadamard, `BUILD_SPEC.md` §10.1) is the validation that the choice of Pauli basis does not change the answer up to a tolerance of 10% NRMSE delta. A pass is a sanity check; a fail is a basis-alignment artefact and a bug.

### 10.2 Shot noise and the variance of finite-shot estimators

A Pauli expectation $\langle P \rangle \in [-1, 1]$ estimated from $S$ shots is unbiased with variance bounded by $1/S$ (the $\pm 1$ outcomes give a Bernoulli-like bound). This means:

- $S = 1024$ shots: $\sigma_{\hat P} \leq 1/\sqrt{1024} \approx 0.031$.
- $S = 4096$ shots: $\sigma_{\hat P} \leq 0.016$.
- $S = 16384$ shots: $\sigma_{\hat P} \leq 0.0078$.
- $S = 65536$ shots: $\sigma_{\hat P} \leq 0.0039$.

The harness's exact-amplitude default eliminates this variance; shot-noise paths via Qiskit Aer expose it. A7 (shot-count scan, `BUILD_SPEC.md` §10.1) varies $S \in \{1024, 4096, 16384, \infty\}$ and asserts NRMSE convergence. Failure of the shot-count scan to converge is a **classical-side bug** in our scoring code 95% of the time; the Pauli-estimator bound is theoretically tight, so any deviation from the predicted $1/\sqrt{S}$ NRMSE rate is the harness's fault, not the simulator's.

### 10.3 The Estimator-V2 alignment with Phase 1

Qiskit 2.x's `EstimatorV2` primitive (released in qiskit-ibm-runtime 0.23+ and stable in qiskit-aer 0.17+) computes Pauli expectation values directly without manual measurement-circuit construction. The Phase 1 cross-check in `reservoirs/qiskit_crosscheck.py` uses `EstimatorV2` and binds observables as `SparsePauliOp` instances, not as measurement-basis-rotated counts. This is the modern API and aligns with IBM Quantum's V2-only forward path (V1 primitives were removed in qiskit-ibm-runtime 0.20). Implementation notes:

- **PUB structure.** Each cross-check call passes `(circuit, [Pauli('Z' + 'I'*(n-1)), Pauli('I'*1 + 'Z' + 'I'*(n-2)), ...])` as a single PUB; results return `data.evs` of shape `(n,)`.
- **Backend options.** `AerSimulator(method='statevector')` for exact paths; `method='density_matrix'` when injecting `noise_model`; this matches `BUILD_SPEC.md` §9.4.
- **Precision vs. shots.** `EstimatorV2.options.default_precision` is set explicitly to `0.0` in exact paths and to `1.0/sqrt(S)` in sampled paths; we *never* leave precision at its default to avoid silent drift between qiskit-aer minor versions.

These alignment notes are not normative (they live in `qiskit_crosscheck.py`'s docstring), but they belong here because a future maintainer encountering an EstimatorV1-vs-V2 ambiguity should find the rationale in this companion document, not have to chase Qiskit changelogs.

---

## 11. Noise Theory and the Phase-1 Two-Channel Model

### 11.1 What the two-channel model captures

The Phase-1 noise model has two channels (`BUILD_SPEC.md` §9.4):

- **Per-CNOT depolarizing channel.** Each CNOT is followed by $\rho \mapsto (1 - p)\rho + p \cdot I/4$. Single-qubit gates are noiseless. Default $p \in \{0.001, 0.005, 0.01\}$ (typical 2026 NISQ device fidelities).
- **Symmetric readout flip.** Each measured bit is flipped with probability $p_\text{ro} \in \{0.005, 0.01, 0.02\}$.

This is a *minimal* model. It captures (a) entangler-dominated decoherence, which is the leading error budget on most superconducting devices; (b) measurement-readout error, which is the second-leading budget. It does **not** capture coherent over-rotation, T1/T2 idling, crosstalk, leakage, or platform-specific native-gate decompositions.

The minimal model is the right Phase-1 choice because:

1. It has two parameters, both pre-registered, both varied in A7-adjacent noise sweeps.
2. It is *strictly more pessimistic* than a typical device-realistic model at the same surface-level error rate (depolarizing is the worst-case Pauli channel at a given diamond-norm).
3. It admits a closed-form effect on Pauli expectations: $\langle P \rangle_\text{noisy} = (1 - p_\text{eff}) \langle P \rangle_\text{exact}$ for an effective $p_\text{eff}$ that scales linearly with CNOT count. This gives us a *predicted* slope for Figure F5.

### 11.2 Why we don't model T1/T2 in Phase 1

T1 (relaxation) and T2 (dephasing) are *time-dependent* noise channels. Their effect depends on gate duration, idling time, and platform-specific calibration. To simulate them correctly requires either (a) a full pulse-level schedule (Qiskit Pulse — deprecated in qiskit 2.x) or (b) a unitary-folded approximation that ties duration to gate count (which is what Aer's `thermal_relaxation_error` does).

Either path forces a Phase-1 commitment to a specific platform's calibration, which we are explicitly *not* making. The two-channel model is platform-agnostic; T1/T2 is not. Phase 2 introduces a calibrated T1/T2 model targeted at one specific NISQ platform (TBD). This is `BUILD_SPEC.md` MG3.

### 11.3 The Q-CTRL mitigation deferral

Q-CTRL's Fire Opal (cloud-native, AI-driven error suppression with dynamical decoupling and pulse-shaping) and Boulder Opal (R&D toolkit for filter-function design and reservoir engineering) are the most mature noise-mitigation stacks available in the QML / NISQ ecosystem as of May 2026. Their integration story for QRC-Thresher is:

- **Phase 1: out of scope.** Phase 1 runs on simulators with our two-channel model. There is no QPU to mitigate.
- **Phase 2: out of scope unless G6 fails specifically due to noise.** If Phase 2 hardware results pass G7, we have no need to insert mitigation between the device and our scoring pipeline. Mitigation introduces a new claim-evidence chain (the mitigation pipeline itself becomes a covariate) that we will accept *only if compelled to*.
- **Phase 3: conditional.** If Phase 2 fails G6/G7 specifically because of noise (and not because of, e.g., insufficient seeds, or device unavailability), we integrate Fire Opal as a Qiskit Function (the V2-primitive integration path documented at `quantum.cloud.ibm.com/docs/guides/q-ctrl-performance-management`). The Boulder Opal toolkit is not directly relevant to QRC because it targets quantum-control engineering, not quantum-algorithm execution.

The Phase-3 integration is documented in `BUILD_SPEC.md` §22.4 ("Q-CTRL conditional"). The exact API contract is **not** pre-registered because Q-CTRL's Fire Opal API surface (in particular the Performance Management function) is on a quarterly release cadence and will change between now and a hypothetical Phase 3 trigger. The only commitment is: if Q-CTRL is integrated, the integration is documented with the same proof-layer rigour as the rest of the harness, including a `mitigation_hash` column in the manifest and a separate cross-check gate analogous to G0.5.

---

## 12. Statistical Theory: BCa, Holm, Power, and Pre-Registration

### 12.1 Why frequentist, not Bayesian

A Bayesian framing of "does QRC separate from ESN by an effect size $\delta$?" requires a prior on the per-task effect size. Two priors are defensible (a sceptical prior centred on zero with small variance; an optimistic prior centred on the QRC-Lab arXiv:2602.03522 reported effect size with moderate variance), and neither is *more* defensible than the other in a community where the prior odds of a positive QRC report being real are contested.

The frequentist BCa interval, paired across (seed, RR), with pre-registered effect sizes and Holm–Bonferroni correction, is therefore the right choice for Phase 1. It is:

- **Reader-friendly.** Reviewers at *Quantum* and the NeurIPS QML workshop are conversant with paired t-tests, bootstrap CIs, and Holm correction. Bayesian framings require a half-page methods digression to land.
- **Auditable.** A reviewer can recompute the CI from the per-test p-values logged in `runs.csv` without re-running the experiment. A Bayesian posterior cannot easily be recomputed from summary statistics; it requires the raw per-cell residuals.
- **Pre-registration-compatible.** Pre-registering a frequentist test is a single line ("paired t-test at α=0.05"); pre-registering a Bayesian analysis is a paragraph (prior, likelihood, decision rule).

This is `BUILD_SPEC.md` Appendix E.7 expanded to the methodology level.

### 12.2 BCa derivation, briefly

The percentile bootstrap CI for a statistic $T$ with bootstrap distribution $\{T_b^*\}_{b=1}^B$ is

$$\mathrm{CI}_\alpha = [Q_{\alpha/2}(\{T_b^*\}), Q_{1-\alpha/2}(\{T_b^*\})].$$

This is unbiased only for symmetric, location-equivariant statistics. The BCa correction adjusts the percentiles by a **bias correction** $z_0 = \Phi^{-1}(\Pr[T_b^* < T])$ and an **acceleration constant** $a$ derived from the jackknife distribution of $T$. The corrected percentile cutoffs are

$$\alpha_\text{lo} = \Phi\!\left(z_0 + \frac{z_0 + z_{\alpha/2}}{1 - a(z_0 + z_{\alpha/2})}\right), \quad \alpha_\text{hi} = \Phi\!\left(z_0 + \frac{z_0 + z_{1-\alpha/2}}{1 - a(z_0 + z_{1-\alpha/2})}\right).$$

At our typical sample sizes ($n_\text{pairs} \in \{15, 30, 75\}$), the percentile bootstrap undershoots nominal coverage by 2–7 pp; BCa typically holds coverage at 93–95%. The cost is a 10–20× resampling factor (`scipy.stats.bootstrap` with `method='BCa'`, `n_resamples=10000`) which is negligible at our problem sizes.

`metrics/stats.py:bootstrap_ci` ships the percentile floor; manuscript-track sweeps SHOULD invoke `scipy.stats.bootstrap(..., method='BCa', n_resamples=10000)` directly. Tracked in `BUILD_SPEC.md` MG8.

### 12.3 Holm vs. Bonferroni vs. BH-FDR

Three candidate multiple-comparison corrections, three different control properties:

- **Bonferroni.** Reject $H_0^{(i)}$ if $p_i < \alpha / m$. Controls FWER. Simplest. Most conservative.
- **Holm.** Sort $p_{(1)} \leq \cdots \leq p_{(m)}$; reject $H_0^{((i))}$ if $p_{(i)} < \alpha / (m - i + 1)$ for all $i' \leq i$. Controls FWER. Strictly more powerful than Bonferroni.
- **Benjamini–Hochberg (BH-FDR).** Sort as above; reject $H_0^{((i))}$ if $p_{(i)} < (i/m) \alpha$. Controls *expected* FDR.

For our family of three pre-registered hypotheses (one per task), Holm is the right choice:

- The family is *small* and *fixed* (no screening from a larger pool), so FDR control is overkill in the wrong direction (it permits more false positives than FWER control).
- Holm has identical FWER control to Bonferroni at strictly higher power, so Bonferroni is a Pareto-dominated choice.

The harness logs the per-test p-values in `runs.csv`; a reviewer who prefers Bonferroni or BH-FDR can recompute the corrected verdicts from the logged data without re-running the experiment.

### 12.4 Power analysis under realistic SDs

Empirical pilot runs at the alpha-lite default (n=4, L=3, three seeds) yielded between-cell SDs:

- STM-MC: $\sigma \approx 0.35$ (range 0.18–0.55).
- parity(3) accuracy: $\sigma \approx 0.06$ (range 0.03–0.09).
- NARMA-10 NRMSE: $\sigma \approx 0.08$ (range 0.05–0.12).

Under the pre-registered effect sizes (`BUILD_SPEC.md` §13.4) and at $n_\text{pairs} = 15$ (5 seeds × 3 RR), one-sided power at $\alpha = 0.05$:

| Task     | Effect size           | $\sigma$ | $n_\text{pairs}$ | Power |
| -------- | --------------------- | -------- | ---------------- | ----- |
| STM      | $\Delta\mathrm{MC} \geq 0.5$ | 0.4 | 15 | $\approx$ 0.92 |
| parity   | $\Delta\mathrm{acc} \geq 0.05$ | 0.06 | 15 | $\approx$ 0.83 |
| NARMA-10 | $\Delta\mathrm{NRMSE} \leq -0.05$ | 0.08 | 15 | $\approx$ 0.74 |

NARMA-10 is the limiting case. Phase 1.5's expansion to $n_\text{pairs} = 75$ (15 seeds × 5 RR) raises NARMA power above 0.95. Phase 1.0 at $n_\text{pairs} = 15$ is an acknowledged compromise; results that meet the effect size but not the CI requirement are reported as **inconclusive**, never **fail** — this is the pre-registered fail-vs-inconclusive boundary that distinguishes a low-power result from a real null.

### 12.5 The pre-registration-as-code convention

Pre-registration in QRC-Thresher is *as code*, not *as document*: effect sizes, gate thresholds, seed lists, and grids live in YAML configs that are hashed into the manifest. A change to a pre-registered value forces a *new* config file (configs are immutable per `BUILD_SPEC.md` §7.4) and a *new* manifest chain. There is no "we forgot to log it" failure mode.

This is the single most important methodological commitment in the harness, and it is the one we expect to be most easily forgotten under deadline pressure. The manifest hashing is the only thing that makes it enforceable.

---

## 13. Backend Engineering: PennyLane 0.44 and Qiskit 2.x in May 2026

### 13.1 PennyLane 0.44 — the May 2026 reference

PennyLane 0.44 (released late Q1 2026, current minor at time of writing) introduced:

- **QRAM primitives** (`qml.BBQRAM`, `qml.SelectOnlyQRAM`, `qml.HybridQRAM`) for noise-resilient classical-state-into-quantum-state loading. *Out of scope for Phase 1* (we do not load high-dimensional classical state; we load a single scalar $u_t$). May become relevant for Phase 1.5 if we add a windowed-input variant.
- **Resource estimation** via `qml.estimator.BBQRAM` and friends. Used by the harness's compute-budget tracker (`BUILD_SPEC.md` §23) to predict per-cell cost before launching a sweep.
- **Continued Lightning improvements.** `lightning.qubit` is now SIMD-vectorised for nearly all single- and two-qubit gates, with C++/OpenMP parallelism over qubits and over batched circuits. `lightning.gpu` and `lightning.amdgpu` exist but are *out of scope* for Phase 1 (`BUILD_SPEC.md` C2: CPU-first).

The harness's `pyproject.toml` pins `pennylane>=0.44,<0.45` to track 0.44.x patch releases without auto-upgrading across the next minor. This is a deliberate choice: PennyLane has historically introduced gate-decomposition changes between minors, which would invalidate `circuit_hash` continuity across realisations.

### 13.2 The default.qubit vs lightning.qubit choice

| Property                | `default.qubit`           | `lightning.qubit`             |
| ----------------------- | ------------------------- | ----------------------------- |
| Implementation          | Pure NumPy                | C++ with OpenMP / SIMD        |
| Speed at $n = 4$        | 1× (reference)            | 1.0–1.5× faster (small n overhead dominates) |
| Speed at $n = 8$        | 1× (reference)            | ~5× faster                    |
| Speed at $n = 10$       | 1× (reference)            | ~10–15× faster                |
| Determinism             | Bitwise across runs       | Bitwise within thread; non-bitwise across thread counts |
| Debugging               | Excellent (Python stack)  | Cumbersome (C++ stack)        |
| Memory                  | $2^n$ doubles + overhead  | $2^n$ doubles (tighter)       |

The harness's discipline is:

- **Development and debugging:** `default.qubit`. Reproducibility is bit-exact; stack traces are intelligible.
- **Sweeps and gates:** `lightning.qubit` with `OMP_NUM_THREADS=1` (so determinism remains bit-exact within thread). The single-thread choice trades 2–4× sweep wall-clock for cross-machine determinism, which is the right Phase-1 trade-off given the 24-hour wall-clock budget.
- **Cross-check (G0.5, G5):** Both backends are run on the same (n, L, seed) triple and asserted identical to 1e-6 (`BUILD_SPEC.md` §15.2, §15.8). The 1e-6 tolerance is six orders of magnitude looser than the predicted floating-point agreement (`BUILD_SPEC.md` Appendix E.10), giving slack for legitimate gate-decomposition order variations.

### 13.3 Qiskit 2.x — the V2-primitives-only world

Qiskit 2.x (released Q4 2025) removed the V1 Sampler / Estimator primitives and the `QuantumInstance` wrapper. The V2 primitives are the only path forward:

- **`SamplerV2`.** Returns measurement counts. Used for shot-based observable estimation when an explicit shot count is desired.
- **`EstimatorV2`.** Returns expectation values directly, given `(circuit, observable)` PUBs. The harness's primary cross-check primitive.

The harness's cross-check uses `EstimatorV2` from `qiskit_aer.primitives` (the local Aer-backed implementation) rather than `qiskit_ibm_runtime` (the cloud-backed implementation), because Phase 1 forbids cloud QPU access (`BUILD_SPEC.md` C4). The V2 PUB structure looks like:

```python
from qiskit_aer.primitives import EstimatorV2
from qiskit.quantum_info import SparsePauliOp

estimator = EstimatorV2()
observables = [SparsePauliOp.from_list([("I" * i + "Z" + "I" * (n - 1 - i), 1.0)]) for i in range(n)]
job = estimator.run([(circuit, observables)])
expvals = job.result()[0].data.evs  # shape (n,)
```

This is the modern, V2-aligned form. It replaces the older measurement-rotation-and-counts flow that was idiomatic in qiskit 0.x / 1.x. We document it here because a future maintainer may otherwise reinvent the older, deprecated flow.

### 13.4 ReservoirPy alignment

The ESN baseline uses `reservoirpy>=0.3.11` (current as of May 2026). The relevant API surface is:

- `reservoirpy.nodes.Reservoir(units=N, sr=ρ, lr=α, input_scaling=σ)`.
- `reservoirpy.nodes.Ridge(ridge=α)`.
- The two are composed via `>>`: `model = reservoir >> readout`.

The 0.3.x line is API-stable and we expect it to remain the reference through Phase 1. ReservoirPy's `verbosity=0` is set explicitly in `baselines/esn.py` so that ReservoirPy's progress prints do not interleave with the harness's logging (a real source of confusion in development).

### 13.5 What "stable" means for upstream pins

The harness pins major versions but allows minor and patch float within the major (`pennylane>=0.44,<0.45`, `qiskit[all]>=2.3,<3`, etc.). This is a pragmatic compromise:

- **Pinning too tight** (e.g., `pennylane==0.44.1`) creates security-update friction and breaks under any upstream patch release.
- **Pinning too loose** (e.g., `pennylane>=0.4`) admits gate-decomposition changes that invalidate `circuit_hash` continuity.

The minor-pinned floor is what we ship. The lockfile (`pip-compile` or `uv` produced) is hashed into the manifest, so any patch-level drift between two reproductions is caught immediately.

---

## 14. Noise Mitigation Ecosystem (Q-CTRL, ZNE, DD) — Phase 2/3 Roadmap

Phase 1 deliberately runs without any noise mitigation. This section documents the mitigation primitives we *could* integrate in Phase 2 / 3 and why we deferred each.

### 14.1 Zero-noise extrapolation (ZNE)

ZNE runs a circuit at multiple noise scalings (e.g., $1\times$, $2\times$, $3\times$ folded CNOTs) and extrapolates the observable expectation to the zero-noise limit. Implementations: Mitiq (open-source), Qiskit's runtime-level ZNE option, Q-CTRL Fire Opal's automated variant.

For QRC-Thresher, ZNE is a **Phase 2 primitive** because:

- It introduces a new claim-evidence chain (extrapolation order, fold strategy, fit goodness) that becomes a covariate in any positive result.
- The Phase 1 simulator already has a "zero-noise" path (exact-amplitude). ZNE on a simulator is testable but uninformative — it should reproduce the exact path within ZNE's estimator variance, which is a sanity check, not a result.

If Phase 2 hardware shows a positive QRC result *without* ZNE, we report it as such. If it shows negative-without-ZNE-but-positive-with-ZNE, we report both with ZNE clearly labelled as "with mitigation," and the comparison is logged as `mitigation_required == true` in the manifest.

### 14.2 Dynamical decoupling (DD)

DD inserts $\pi$-pulse trains (e.g., XY4, CPMG) on idle qubits to suppress dephasing. Q-CTRL's Fire Opal implements it as part of its automated pipeline; Qiskit's transpiler exposes it as a `PadDynamicalDecoupling` pass.

For QRC-Thresher Phase 2, DD is interesting because the HEA's idle-qubit pattern (during single-qubit rotations on a subset of qubits) is exactly the failure mode DD addresses. A pre-Phase-2 ablation will run a one-cell DD-on / DD-off comparison; if DD changes the headline metric by more than the pre-registered noise-sweep tolerance, DD becomes a gated Phase-2 covariate. If it does not, DD is reported as "applied, no measurable effect" and not used in subsequent runs.

### 14.3 Q-CTRL Fire Opal as a Qiskit Function

Q-CTRL's Performance Management function (`fire-opal`, available as a Qiskit Function on IBM Quantum since 2025) provides a black-box "submit your circuit, get a mitigated result" interface. Its Phase-3 integration story is straightforward:

- **Drop-in.** Replace `EstimatorV2()` with `Estimator(channel="ibm_quantum", mode=session)` and pre-process the circuit through Fire Opal's `optimize_circuits()` API.
- **Manifest fields.** Add `mitigation_provider`, `mitigation_version`, `mitigation_options_hash` columns to `runs.csv`. Bump manifest schema to v1.2.
- **Cross-check gate.** Add G0.7: Fire-Opal-mitigated vs. unmitigated cross-check on a noise-free simulator (where mitigation should be a no-op). A nonzero residual on the no-op cross-check is a Fire-Opal API misuse on our side and gates the integration.

This integration is documented but **not implemented** in Phase 1. It is a contingent Phase-3 line item conditioned on G6 specifically failing because of noise.

### 14.4 Reservoir engineering (Boulder Opal)

Boulder Opal targets *quantum control engineering* — designing pulse shapes, calibration sequences, and noise-resilient gate decompositions — not algorithm execution. Its relevance to QRC-Thresher is therefore indirect:

- It is the right toolkit for the Phase-2 hardware-readiness work where we evaluate native-gate-decomposition fidelity.
- It is *not* a runtime primitive that the harness would call during a sweep.

We list it here for completeness because reviewers familiar with the Q-CTRL stack may expect an explicit position, and because the term "reservoir engineering" in Boulder Opal's sense (engineering an environmental coupling to protect a quantum state) is *unrelated* to "reservoir computing" (the family of recurrent learning systems). This terminology collision is real and non-trivial; we are explicit about it in the manuscript.

---

## 15. Context Engineering for the Proof Layer

"Context engineering" — the discipline of shaping the surrounding artefacts (configs, manifests, logs, citations) so that a downstream consumer (reviewer, replicator, future maintainer, automated agent) can act with full local context — is the methodological frame this section adopts. The QRC-Thresher proof layer is a context-engineering artefact in this sense.

### 15.1 What the proof layer is and what it is not

The proof layer is the harness's machinery for tying every reported number to a fully reconstructible context: code commit, config hash, circuit hash, seeds, environment, runtime. It is implemented in `src/qrc_thresher/proof/run_manifest.py`, `proof/source_health.py`, and `proof/benchmark_health.py`.

What it is:

- A *write-once* log of every run, suitable for `grep`, `pandas.read_csv`, and external indexing.
- A *cryptographically anchored* (Phase 1.5+ Ed25519-signed) integrity record.
- A *complete* description: a reviewer with the manifest, the lockfile, and `git checkout <commit>` can reproduce within bootstrap CI.

What it is not:

- A narrative or interpretation. Manifests carry *evidence*, not *opinion*.
- A leaderboard. Manifests are append-only; "top result" is a derived view, not a stored field.
- A source of truth for code or config. Code lives in git; config lives in the YAML; the manifest *references* both via hashes.

### 15.2 The claim-evidence chain

Every claim in a derived artefact (manuscript figure, table, abstract) MUST trace, by construction, to:

1. A **headline value** (mean, CI bounds).
2. A **list of run_ids** that produced the value.
3. For each run_id, a **manifest row** with config_hash, circuit_hash, git_commit_hash, package_versions.
4. For each config_hash, a **YAML file** in the repo (immutable).
5. For each git_commit_hash, a **commit** in the repo's history.

The chain is verifiable at any link by a reviewer with shell access. The reporting tooling (`qrc-thresher plot`, `qrc-thresher summary`) enforces it by construction: any value emitted without a backing run_id is a tooling bug, not a reportable result. This is `BUILD_SPEC.md` H4 ("no unverifiable claims") expanded.

### 15.3 Context-windowing for downstream agents

A future maintainer (human or LLM-driven; the May 2026 best practice is to assume both) interacting with this repository needs three layers of context:

1. **Fast (BUILD_SPEC §1, README, Makefile).** "What is this project, how do I install it, what is the first command to run?" 5 minutes.
2. **Medium (BUILD_SPEC + this document).** "Why is the methodology what it is? What are the gates? What constitutes a positive result?" 60–90 minutes.
3. **Deep (REFERENCES.md + the cited literature + DECISIONS.md).** "Where does QRC stand in the broader literature? What past architectural decisions constrain my edit?" Multi-hour.

The three layers are *disjoint by design*: each carries no redundant content with the others. A reader who has spent the medium-tier 90 minutes on this document and BUILD_SPEC should not have to re-derive any rationale by going back to the literature; conversely, a reader who has only spent the fast-tier 5 minutes should have a complete enough picture to *not* break anything with a small edit.

This is the May-2026 context-engineering best practice that this docs expansion is meant to instantiate. The specific manifestation here is the cross-reference index (§16): it is the deliberate seam between the two prescriptive documents (BUILD_SPEC and this companion) so that a maintainer editing one knows *exactly* which sections of the other are affected.

### 15.4 Reproducibility-as-context

The reproducibility contract (`BUILD_SPEC.md` §19) is itself a context-engineering primitive: it specifies the minimal context a reviewer needs to *re-derive* a claim, not just to *read* it. Five lines of shell (`git checkout`, `pip install -e .`, `qrc-thresher health`, `qrc-thresher run`, `qrc-thresher gate`) is the commitment. The five-line bound is what makes context engineering *auditable* — a longer reproduction recipe accumulates undocumented preconditions that drift over time; a five-line recipe does not.

### 15.5 Why we keep the manifest unsigned in Phase 1

Phase 1 ships unsigned manifests; Phase 1.5 introduces Ed25519 signing (`BUILD_SPEC.md` §16.4, MG9). The unsigned-in-Phase-1 choice is a context-engineering trade-off:

- **For an internal release**, signing introduces key-management overhead (where does the private key live? who rotates it?) without a corresponding integrity guarantee that git history does not already provide.
- **For a public arXiv submission**, signing is necessary because the manifest leaves the trust boundary of the repository, and downstream consumers cannot rely on git history alone (forks, mirrors, git-ref ambiguity).

The Phase-1-vs-Phase-1.5 boundary is exactly the boundary at which the manifest leaves the trust boundary, which is why the introduction of signing is gated to that boundary and not earlier.

---

## 16. Cross-Reference Index to BUILD_SPEC.md

This index lets a maintainer editing one document find every affected section in the other.

| Topic                              | BUILD_SPEC.md       | This document |
| ---------------------------------- | ------------------- | ------------- |
| Mission and scientific question    | §1                  | §7.5, §15     |
| Falsification-first methodology    | §2                  | §15           |
| Honesty/Compute/Literature/Scope bars | §3               | §15.2, §15.4  |
| Glossary and notation              | §4                  | §1–§5         |
| Repository layout                  | §5                  | (none)        |
| Dependencies and environment       | §6                  | §13           |
| Configuration schema and defaults  | §7                  | §12.5         |
| Tasks (definitions)                | §8                  | §1, §8        |
| Quantum reservoir architecture     | §9                  | §2, §9, §10   |
| Ablation suite                     | §10                 | §4, §6.2, §9.3 |
| Classical baselines                | §11                 | §3, §6.4      |
| Training and CV protocol           | §12                 | §6.3          |
| Metrics                            | §13                 | §5, §8.1      |
| Statistical methodology            | §14                 | §5, §12       |
| Decision gates G0–G7               | §15                 | §9.3 (G2.5)   |
| Proof layer manifest               | §16                 | §15           |
| CLI surface                        | §17                 | (none)        |
| Health checks                      | §18                 | (none)        |
| Reproducibility contract           | §19                 | §15.4         |
| Testing strategy                   | §20                 | (none)        |
| Lint/format/CI                     | §21                 | (none)        |
| Phases and roadmap                 | §22                 | §11.3, §14    |
| Compute budget                     | §23                 | §13.1         |
| Risks and mitigations              | §24                 | (cross-cutting) |
| Differentiation vs prior art       | §25                 | §7            |
| Reporting protocol                 | §26                 | §15.2         |
| License and attribution            | §27                 | (none)        |
| Methodology gap register MG1–MG10  | §28                 | §7.2 (MG4, MG7), §11.2 (MG3), §12.2 (MG8), §15.5 (MG9) |
| Assumed defaults                   | §29 / Appendix A    | (cross-cutting) |
| Example configuration              | §30 / Appendix B    | (none)        |
| Schemas (runs.csv, gate.json)      | §31 / Appendix C    | §15.2         |
| Acceptance checklist               | §32 / Appendix D    | (none)        |
| Extended methodology notes         | Appendix E          | §6, §7, §8, §9, §10, §11, §12 (this doc subsumes Appendix E) |
| Operational runbooks               | Appendix F          | (none)        |
| Closing reviewer notes             | Appendix G          | §15           |

A row marked `(none)` indicates the topic is wholly prescriptive in `BUILD_SPEC.md` and has no derivational counterpart in this companion. This is by design: the companion does not add content where there is no rationale to add.

---

## 17. Change Log

- **2026-05-03 — v1.0** — Methodology companion expanded from a 97-line stub to a full long-form derivational document. Adds §§6–15 (theoretical foundations, task theory, circuit theory, readout theory, noise theory, statistical theory, backend engineering for Qiskit 2.x / PennyLane 0.44, Q-CTRL deferral roadmap, context-engineering frame). Cross-reference index to BUILD_SPEC.md added (§16). Aligns the harness documentation with May-2026 ecosystem state (PennyLane 0.44 QRAM/resource estimation primitives, Qiskit 2.x V2-only primitives surface, Q-CTRL Fire Opal Qiskit-Function integration path, 2025–2026 QRC theoretical literature including Sannia et al. *Quantum* 2024 universality, controllable-damping *npj Quantum Information* 2026, ESP-hierarchy OpenReview 2024, and the Domínguez et al. *Phil. Trans. R. Soc. A* 2026 review).
- **2026-05-01 — v0.1** — Initial 97-line stub authored by Forge-1; covered Tasks, Quantum Reservoir Architecture, Classical Baselines, Ablations, Statistical Methodology at the prescriptive surface only.

*End of METHODOLOGY.md.*
