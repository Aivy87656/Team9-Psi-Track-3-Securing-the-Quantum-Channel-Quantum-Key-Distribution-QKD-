# BB84 QKD Pipeline — Qiskit Aer Simulation

An end-to-end simulation of the BB84 quantum key distribution (QKD) protocol built in Qiskit, including an intercept-resend eavesdropper model, a depolarizing channel noise model, classical Cascade error reconciliation, Toeplitz-hash privacy amplification, and a Monte Carlo sweep that characterizes QBER, secret key rate, and abort behavior as a function of attack strength and channel noise.

## What this project does

1. **Generates quantum random numbers** (`qrng_bits`) using real H-gate + measurement circuits on the Aer simulator, rather than a classical PRNG, for Alice's bits/bases and Bob's bases.
2. **Simulates the full BB84 protocol** (`run_qiskit_bb84_pipeline`) as a single Qiskit circuit per trial:
   - Alice encodes bits into qubits using randomly chosen Z/X bases.
   - An eavesdropper ("Eve") optionally intercepts each qubit with probability `eta`, measures it in a random basis, and resends it (intercept-resend attack).
   - A depolarizing channel randomly applies an X, Y, or Z fault to each qubit with probability `p_depol`.
   - Bob measures each qubit in a randomly chosen basis.
3. **Sifts and estimates QBER**: keeps only qubits where Alice's and Bob's bases matched, then samples a subset of the sifted bits to estimate the quantum bit error rate (QBER).
4. **Applies a security check**: aborts if QBER ≥ 11.04% (the Shor-Preskill asymptotic security bound).
5. **Reconciles and distills a secure key**:
   - Cascade multi-pass error reconciliation corrects mismatches between Alice's and Bob's raw keys.
   - A Toeplitz-hash privacy amplification step compresses the reconciled key to the Shor-Preskill secure length, removing any information Eve could have gained.
6. **Runs a statistically robust parameter sweep**: instead of one simulation per (noise, η) point, it runs `N_TRIALS = 20` independent Monte Carlo repetitions per point and reports **mean ± standard deviation** for QBER and key rate, plus an **abort rate** broken down by failure mode.
7. **Visualizes results** in a 4-panel dashboard: QBER vs. attack strength, key rate vs. attack strength, a noise-vs-eavesdropping discrimination plot, and a key-distillation "waterfall" showing bit counts shrinking at each protocol stage.

## Why the Monte Carlo averaging matters

The original single-trial version produced a noisy, non-monotonic QBER curve (QBER occasionally *dropping* as attack strength increased), which is physically impossible for the true underlying probability — it was a small-sample artifact. Averaging over 20 independent trials per parameter point smooths the curve so it tracks the theoretical prediction:

```
QBER ≈ p/2 + (η/4)(1 − p)
```

where `p` is the depolarizing noise probability and `η` is Eve's intercept probability. It also turns the previously confusing single-shot "abort flips" into a genuine demonstration of **finite-key effects**: near the security boundary, whether a given block of qubits triggers an abort becomes probabilistic, which is exactly the finite-sample-size vs. asymptotic-security-proof gap real QKD systems have to account for.

## Key concepts

| Term | Meaning |
|---|---|
| **QBER** | Quantum Bit Error Rate — the fraction of mismatched bits between Alice and Bob in the sifted key sample. Used to detect eavesdropping. |
| **η (eta)** | Eve's intercept probability — the fraction of qubits she intercept-resends. |
| **p_depol** | Depolarizing channel noise probability per qubit. |
| **`ABORT_EAVESDROPPER_DETECTED`** | QBER ≥ 11.04%, so the protocol aborts on suspicion of eavesdropping. |
| **`ABORT_ZERO_KEY_RATE`** | QBER is below the detection threshold, but the sifted block was too small/noisy for the Shor-Preskill formula to yield even one secure bit after reconciliation leakage and privacy amplification. A distinct finite-key failure mode, not an error. |
| **`abort_rate`** | Fraction of the 20 trials at a given (noise, η) point that ended in either abort condition — the empirical signature of finite-key statistical fluctuation near the security boundary. |
| **Shor-Preskill rate** | Asymptotic secret key fraction `R = max(0, 1 − (1+f)·h₂(QBER))`, where `h₂` is the binary entropy function and `f` is the reconciliation efficiency (leakage overhead). |

## Project structure (notebook cells)

1. **Setup** — installs `qiskit` and `qiskit-aer`, imports.
2. **QRNG** — `qrng_bits`: generates genuinely random classical bits from quantum measurements.
3. **Classical post-processing primitives** — Cascade reconciliation, Toeplitz privacy amplification, Shor-Preskill rate formula.
4. **Single-trial BB84 pipeline** — `run_qiskit_bb84_pipeline`: the full protocol as one Qiskit circuit.
5. **Monte Carlo aggregation** — `run_sweep_point`: runs `N_TRIALS` repetitions of the pipeline at a fixed (noise, η) point and aggregates mean/std/abort statistics.
6. **Parameter sweep** — sweeps η from 0.0 to 1.0 (11 points) across 3 noise profiles (0%, 2%, 5%), 20 trials each.
7. **Results dashboard** — 4-panel `matplotlib` visualization, saved to `qkd_dashboard_fixed.png`.

## Requirements

- Python 3.9+
- `qiskit >= 2.1.0`
- `qiskit-aer >= 0.17.0`
- `numpy`, `matplotlib`

Install with:
```bash
pip install -q 'qiskit>=2.1.0' 'qiskit-aer>=0.17.0' numpy matplotlib
```

## Running

Run all notebook cells in order. The full parameter sweep (3 noise profiles × 11 η values × 20 trials, up to 1000 qubits per trial) typically completes in under two minutes on the Aer stabilizer simulator, since all gates used (X, Y, Z, H, measurement) are Clifford operations.

The dashboard image is saved to `qkd_dashboard_fixed.png` in the working directory.

## Notes / known limitations

- `sample_ratio` (default 0.25) sacrifices a fixed fraction of sifted bits for QBER estimation rather than sizing the sample via a statistical confidence bound (e.g. a Chernoff bound), which is the more rigorous approach for finite-key security analysis.
- The Shor-Preskill rate formula is asymptotic; applying it to small, finite simulated blocks is an intentional simplification to illustrate finite-key effects, not a substitute for a full finite-key security proof.
- Depolarizing noise is applied after Eve's resend step in the circuit; modeling it before interception is an equally valid alternative choice.
