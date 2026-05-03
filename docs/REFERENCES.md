# References

This file is the **Literature Bar** for QRC-Thresher. It is authoritative for the status of every cited work (`BUILD_SPEC.md` Authority clause). Every Phase 1 manuscript MUST cite only entries from the Published Bar (peer-reviewed, arXiv-permanent, or upstream-library official documentation). Forbidden entries MUST NOT be imported in code or referenced in derived artefacts.

The status tags below are normative:

- **[Published Bar]** — citable in the Phase 1 manuscript and in derived artefacts.
- **[Tooling]** — official upstream documentation. Citable in methodology sections; not citable as a *result*.
- **[Background]** — useful for grounding but not directly citable in headline claims.
- **[Forbidden]** — MUST NOT be imported, copied, or cited.

---

## 1. Closest competitor

### QRC-Lab — arXiv:2602.03522 (Feb 2026) [Published Bar]

Nearest architectural neighbour: 4 qubits, depth 3, ring entanglement, Pauli-Z readout, ridge regression on STM + parity + NARMA-10. **Differentiation memo required by G1** before any QRC-Thresher arXiv submission. Key differentiators:

- This repo emphasises falsification-first methodology over result maximisation.
- Full ablation suite (phase-randomised, entanglement-suppressed, Haar-random) is mandatory.
- All gates must pass before any positive claim is made.
- Negative results are treated as a publishable, valuable outcome.

Do **not** copy code from QRC-Lab without verifying license compatibility.

### arXiv:2510.25183 — Sustainable NARMA-10 Benchmarking for QRC [Published Bar]

Reports ESN NRMSE = 0.185 vs QRC NRMSE = 0.485 on NARMA-10. This is the expected performance gap on NARMA. Frame honest reporting accordingly.

---

## 2. QRC theory (2024–2026)

### Sannia, Giorgi, Palma, Zambrini — *Dissipation as a resource for Quantum Reservoir Computing*, *Quantum* 8:1291 (2024) [Published Bar]

Proves universality of dissipative spin networks for fading-memory functionals. Justifies the methodology companion's §7.2 universality claim and bounds the strongest possible Phase-1.5 (stateful, dissipative variant) interpretation.

### *Quantum reservoir computing induced by controllable damping*, *npj Quantum Information* (2026), DOI 10.1038/s41534-026-01229-8 [Published Bar]

Connects the "sweet spot" of memory capacity to controllable physical damping. Cited in §7.2 of `METHODOLOGY.md`. Relevant to Phase 2 hardware-readiness work.

### *Connection between Memory Performance and Optical Absorption in Quantum Reservoir Computing*, *Phys. Rev. Lett.* (2025), 10.1103/vp79-8t1l [Published Bar]

BEC reservoirs; identifies an experimentally accessible observable (optical absorption) corresponding to the optimal memory-capacity dissipation rate. Phase 2 cross-platform reference.

### Domínguez et al. — *From quantum feature maps to quantum reservoir computing: an overview*, *Phil. Trans. R. Soc. A* 384:20250085 (2026) [Published Bar]

The kernel-view review that grounds the QRC-as-feature-map framing in §7.1 of `METHODOLOGY.md`. The right single-source overview for a reviewer who wants the May-2026 state of the QRC field.

### Tran & Nakajima — *Hierarchy of the echo state property in quantum reservoir computing*, OpenReview (2023, accepted 2024) [Published Bar]

Defines non-stationary, subspace, and subset variants of the ESP for quantum systems. Referenced in §7.3. Relevant to Phase 2 (when device drift forces non-stationarity into the comparison).

### Jaeger, H. — *The "echo state" approach to analysing and training recurrent neural networks*, GMD Report 148 (2001) [Background]

The foundational ESN paper. The Memory Capacity bound (`MC ≤ N` for $N$-state linear reservoirs) used in §8.1 is from this report.

### Maass, Natschläger, Markram — *Real-time computing without stable states: A new framework for neural computation based on perturbations*, *Neural Computation* 14:11 (2002) [Background]

The Liquid State Machine companion to Jaeger's ESN. Establishes the Boyd–Chua-style universality argument that underwrites §6.2.

---

## 3. Tooling — primary references

### PennyLane 0.44 [Tooling]

- Release notes: <https://github.com/PennyLaneAI/pennylane/releases/tag/v0.44.0>
- API reference: <https://docs.pennylane.ai/en/stable/>
- Lightning plugins (0.44.x): <https://docs.pennylane.ai/projects/lightning/en/stable/>

Phase 1 backend pin: `pennylane>=0.44,<0.45` and `pennylane-lightning>=0.44,<0.45`. The 0.44 line introduced QRAM primitives (`qml.BBQRAM`, `SelectOnlyQRAM`, `HybridQRAM`) and resource-estimation routines that may be relevant to Phase 1.5; for Phase 1 the relevant API surface is `qml.device('default.qubit', wires=n)` / `qml.device('lightning.qubit', wires=n)` plus the `qml.qnode` decorator on a function returning a list of `qml.expval(qml.PauliZ(i))`.

### Qiskit 2.x [Tooling]

- Migration guide (V1 → V2 primitives): <https://quantum.cloud.ibm.com/docs/en/guides/v2-primitives>
- Aer EstimatorV2 / SamplerV2 reference: <https://qiskit.github.io/qiskit-aer/>
- Native primitives reference: <https://docs.quantum.ibm.com/api/qiskit/primitives>

Phase 1 pin: `qiskit[all]>=2.3,<3` and `qiskit-aer>=0.17`. The cross-check (`reservoirs/qiskit_crosscheck.py`) uses `qiskit_aer.primitives.EstimatorV2` exclusively. V1 primitives and `QuantumInstance` are removed in qiskit-ibm-runtime 0.20+ and MUST NOT be reintroduced.

### Qiskit Aer noise model [Tooling]

- Reference: <https://qiskit.github.io/qiskit-aer/apidocs/aer_noise.html>

Used for the Phase-1 two-channel noise model (`depolarizing_error`, `ReadoutError`). See `METHODOLOGY.md` §11 for the model rationale.

### ReservoirPy ≥ 0.3.11 [Tooling]

- Documentation: <https://reservoirpy.readthedocs.io/>
- Source: <https://github.com/reservoirpy/reservoirpy>

Phase 1 ESN baseline reference. Used in `baselines/esn.py`. The `reservoirpy.nodes.Reservoir` + `reservoirpy.nodes.Ridge` composition is the only API surface the harness depends on.

### scikit-learn ≥ 1.4 [Tooling]

- Documentation: <https://scikit-learn.org/stable/>

Used for `RidgeCV` (training, §12.3 of `BUILD_SPEC.md`) and for `bootstrap`-adjacent utilities. `scipy.stats.bootstrap` (BCa) is the manuscript-track CI primitive — see `METHODOLOGY.md` §12.2.

---

## 4. Noise mitigation ecosystem (Phase 2/3 only)

These are referenced in `METHODOLOGY.md` §14 as Phase-2/3 contingent integration paths. They are **not** Phase-1 dependencies and MUST NOT be added to `pyproject.toml` until the corresponding gate triggers.

### Q-CTRL Fire Opal [Tooling, Phase 3 only]

- Product page: <https://q-ctrl.com/fire-opal>
- Qiskit Function integration guide: <https://quantum.cloud.ibm.com/docs/guides/q-ctrl-performance-management>
- AWS Braket integration: <https://aws.amazon.com/blogs/quantum-computing/improve-quantum-workload-performance-with-fire-opal-error-suppression-for-ionq-processors-on-amazon-braket/>

Triggered only if Phase 2 fails G6 specifically due to noise. Integration introduces manifest schema v1.2 (`mitigation_provider`, `mitigation_version`, `mitigation_options_hash`).

### Q-CTRL Boulder Opal [Tooling, Phase 2 hardware only]

- Documentation: <https://docs.q-ctrl.com/boulder-opal>

Quantum-control engineering toolkit; relevant for Phase 2 hardware-readiness pulse design and filter-function analysis. **Not** a runtime primitive for the QRC-Thresher harness. Note the terminology collision: "reservoir engineering" in Boulder Opal's sense (engineering an environmental coupling to protect a quantum state) is unrelated to "reservoir computing" (the ML paradigm).

### Mitiq [Tooling, Phase 2 only]

- Source: <https://github.com/unitaryfund/mitiq>

Open-source ZNE / PEC / DD library. Optional Phase-2 alternative to Fire Opal for ZNE; introduces the same manifest-schema considerations.

---

## 5. Statistical methodology

### Efron & Tibshirani — *An Introduction to the Bootstrap*, Chapman & Hall (1993) [Background]

The canonical BCa derivation reference. Underwrites `METHODOLOGY.md` §12.2.

### Holm, S. — *A simple sequentially rejective multiple test procedure*, *Scand. J. Statistics* 6:65–70 (1979) [Background]

Holm–Bonferroni reference. Underwrites `METHODOLOGY.md` §12.3.

### `scipy.stats.bootstrap` documentation [Tooling]

<https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.bootstrap.html>

Manuscript-track BCa implementation. Use `method='BCa'`, `n_resamples=10000`. The harness's percentile bootstrap (`metrics/stats.py:bootstrap_ci`) is the floor.

---

## 6. Forbidden references

### QHACK23_QRC [Forbidden]

Unlicensed code repository. Do **not** import, copy, or reference its code. If a result in QHACK23_QRC is referenced in prose, it MUST be marked "external; not reproduced; not citable as evidence" per `BUILD_SPEC.md` L3.

---

## 7. License compatibility note

All code in this repo is original or derived from Apache-2.0 / BSD / MIT / PSF compatible libraries (PennyLane Apache-2.0, Qiskit Apache-2.0, scikit-learn BSD-3, NumPy/SciPy BSD-3, Matplotlib PSF, ReservoirPy MIT, Click BSD-3, Pydantic MIT). All compatible with Apache-2.0 distribution.

Any third-party reservoir code requires explicit license check before inclusion. Q-CTRL Fire Opal and Boulder Opal are commercial products with their own EULAs; their *integration* (calling the API from our code) does not require us to redistribute their code, and is therefore compatible with our Apache-2.0 distribution.

---

## 8. Citation hygiene

Phase 1 manuscripts MUST:

- Cite only **[Published Bar]** entries in result-supporting positions.
- Cite **[Tooling]** entries only in Methods sections, with the version pin from `pyproject.toml` quoted explicitly.
- Cite **[Background]** entries only for foundational claims (e.g., the Jaeger MC bound).
- **Never** cite **[Forbidden]** entries.

A reviewer challenging the status of any entry should open a `literature-bar` issue and reference its section above.
