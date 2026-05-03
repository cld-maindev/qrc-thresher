# Decision Log

This file is append-only. Never rewrite history.

---

## 2026-05-01: D001 — Initial Phase 1 scaffold

**Decision**: Build complete Phase 1 scaffold per the build spec brief.

**Rationale**: Establish reproducible benchmark harness with all required components before
running any experiments.

**Components included**:
- Task generators: STM, temporal parity (NARMA-10 gated behind G3)
- Quantum reservoir: PennyLane QRC (default.qubit + lightning.qubit)
- Ablations: phase-randomized, entanglement-suppressed, Haar-random
- Classical baselines: ESN (reservoirpy), random kitchen sinks, GRU stub
- Metrics: MC, NRMSE, accuracy, bootstrap CIs, paired t-test
- Proof layer: schema v1.1 run manifests, health checks
- CLI: health, run, ablation, gate, plot, summary commands

**Constraints applied**:
- CPU-only (no CUDA/GPU paths)
- No LLM/transformer references
- Synthetic data only
- All randomness via seeded `numpy.random.Generator`

**Status**: Implemented. 68 tests pass.

---

## 2026-05-03: D002 — Methodology companion expansion

**Decision**: Expand `docs/METHODOLOGY.md` from a 97-line prescriptive stub into a full long-form derivational companion to `docs/BUILD_SPEC.md`, and refresh `docs/REFERENCES.md` with the May-2026 ecosystem state (PennyLane 0.44, Qiskit 2.x V2 primitives, Q-CTRL Fire/Boulder Opal as Phase-2/3 contingent integrations, and the 2024–2026 QRC-theory literature on fading memory, ESP hierarchy, and dissipation universality).

**Rationale**: `BUILD_SPEC.md` explicitly designates `METHODOLOGY.md` as the "companion: long-form derivations" (BUILD_SPEC §5.1, §G.1, Authority clause). The 1434-line prescriptive document had outpaced the 97-line companion to the point where rationale lived only in BUILD_SPEC's Appendix E (extended methodology notes). Lifting that rationale into the dedicated companion, and expanding it with theoretical foundations the prescriptive document deliberately omits, restores the intended document split: BUILD_SPEC is *what to do*, METHODOLOGY is *why*.

**Components added to METHODOLOGY.md**:
- §6 Theoretical foundations of (classical) reservoir computing — ESP, FMP, separation, the linear-readout scope cut.
- §7 Theoretical foundations of QRC — kernel view, dissipation universality, ESP hierarchy in non-stationary quantum systems, the barren-plateau exemption, and bounds on what a positive Phase-1 result could mean.
- §8 Task theory — Jaeger MC bound, parity hardness at $d \geq 4$, NARMA-10 stability and seed-replacement justification.
- §9 Circuit theory — HEA expressivity / frame potential, Haar-ablation as the universality control, classical-RC-to-QRC analogue mapping table, circuit hash as a debugging primitive.
- §10 Readout theory — observable space, shot-noise variance bounds, EstimatorV2 alignment.
- §11 Noise theory — two-channel-model rationale, T1/T2 deferral to Phase 2, Q-CTRL deferral to Phase 3.
- §12 Statistical theory — frequentist vs Bayesian framing, BCa derivation, Holm vs BH-FDR, power table at the pre-registered effect sizes, pre-registration-as-code convention.
- §13 Backend engineering — PennyLane 0.44 (QRAM, Lightning, default.qubit vs lightning.qubit choice), Qiskit 2.x (V2-only primitives, AerSimulator integration), ReservoirPy 0.3 alignment, dependency-pin discipline.
- §14 Noise-mitigation ecosystem roadmap — ZNE, DD, Q-CTRL Fire Opal Qiskit-Function path, Boulder Opal terminology-collision note.
- §15 Context engineering for the proof layer — claim-evidence chain, three-tier context windowing for human and LLM-driven maintainers, reproducibility-as-context, Phase-1-vs-Phase-1.5 signing-boundary rationale.
- §16 Cross-reference index between BUILD_SPEC.md and this companion (line-by-line, by topic).
- §17 Document change log.

**REFERENCES.md changes**: Reorganised into status-tagged sections (Published Bar / Tooling / Background / Forbidden). Added the 2024–2026 QRC-theory papers cited above, the upstream Qiskit/PennyLane/ReservoirPy/SciPy/Q-CTRL documentation pointers, and explicit citation-hygiene rules tied to BUILD_SPEC §3.3 (Literature Bar) and L3.

**Constraints respected**:
- No code changes; documentation only.
- No new dependencies (Q-CTRL/Mitiq listed as Phase-2/3 contingent and explicitly **not** added to `pyproject.toml`).
- BUILD_SPEC.md authority preserved: where BUILD_SPEC and METHODOLOGY overlap on prescriptive content, BUILD_SPEC wins; where BUILD_SPEC defers a derivation, METHODOLOGY now carries it.
- Triple-license intact (CC BY 4.0 for docs/).

**Status**: Implemented.

---
