# Simulation-only side-channel leakage assessment of an AES-128 FPGA core

Two AES-128 cores (unprotected / first-order masked), a testbench that emits a
**simulated** power trace per encryption, and a Python TVLA + CPA analysis.
Everything runs in free Vivado (XSIM + out-of-context synthesis) on
`xc7a35ticsg324-1L`. No board, no XDC, no measurement equipment.

```
rtl/aes_unprot.v    round-based AES-128, 11 cycles      <- leakage baseline
rtl/aes_masked.v    first-order masked AES-128, 268 cycles
rtl/sbox.hex        S-box ROM contents
sim/tb_trace.v      drives N encryptions, writes traces_*.csv
py/analyze.py       TVLA (Welch t-test) + CPA on the CSV
py/golden_model.py  cycle-accurate Python twin of both cores
py/reproduce_results.py  regenerates every number below without Vivado
scripts/run_sim.sh  four XSIM runs (unprot/masked x fixed-vs-random/random)
scripts/synth.tcl   OOC synth+impl, dumps utilization and timing reports
```

## Leakage model

The cores update a single 128-bit state register once per clock. The
simulated instantaneous power at cycle *t* is

    P_t = HW( state_t XOR state_{t-1} ) + N(0, sigma)

i.e. the Hamming distance of the state register, which is the standard
first-order model for register-based hardware AES. `sigma = 2.0` LSB of
Gaussian noise is added in the analysis script so the results are not
noise-free-toy results.

## Masking scheme (aes_masked.v)

Herbst-style masked table recomputation plus per-round mask refresh:

1. Fresh `(m_in, m_out)` per encryption; precompute `T[x] = S[x ^ m_in] ^ m_out`
   into 16 parallel 256x8 tables. **256 cycles.**
2. The state register always holds `s ^ mu` with `mu` a *uniform byte* mask.
   A uniform byte mask survives MixColumns unchanged, because
   `02 ^ 03 ^ 01 ^ 01 = 01` in GF(2^8) — that is what makes the scheme cheap.
3. Each round: correct `mu -> m_in`, table lookup (`-> m_out`), ShiftRows,
   MixColumns, AddRoundKey, then **re-mask to a fresh `mu'`**.
4. Final cycle unmasks; the ciphertext is public so this reveals nothing.

Step 3's refresh is the non-obvious part. Without it the mask is identical in
consecutive rounds and cancels inside the XOR of the Hamming distance —
`HW((s_r ^ mu) ^ (s_{r+1} ^ mu)) = HW(s_r ^ s_{r+1})` — so a "masked" design
would leak exactly as much as the unprotected one. This is worth a paragraph in
the thesis: the countermeasure is only sound with respect to a *stated*
leakage model.

Both cores were checked against the FIPS-197 vector
(`key 000102..0f`, `pt 00112233445566778899aabbccddeeff`
-> `ct 69c4e0d86a7b0430d8cdb78070b4c55a`).

## Results

All numbers from `py/reproduce_results.py`, seed 1, sigma = 2.0.

### 1. First-order TVLA, fixed-vs-random, N = 2000

| core | max abs(t) | leaking cycles (abs(t) > 4.5) |
|---|---|---|
| unprotected | **260.86** | 9 of 11 |
| masked | **1.93** | 0 of 12 |

Per-cycle abs(t), unprotected:
`260.9, 13.8, 0.6, 6.7, 9.5, 0.4, 5.0, 6.7, 24.3, 9.6, 58.3`

The two peaks are the initial AddRoundKey load (cycle 0) and the final round
(cycle 10) — the two points where the register transition is directly a
function of the plaintext and of round key 10.

### 2. CPA on the last round, HD model

Model: `HW( InvSbox(ct[j] ^ k) ^ ct[SR[j]] )`, attacking round key 10 byte
by byte at the final-round sample.

| core | N = 200 | N = 1000 | N = 5000 |
|---|---|---|---|
| unprotected | 9 / 16 bytes | **16 / 16** | 16 / 16 |
| masked | 0 / 16 | 0 / 16 | 0 / 16 |

Minimum traces to disclosure for byte 0 (true `0x13`):

| N | 50 | 100 | 200 | 400 | 800 |
|---|---|---|---|---|---|
| best guess | 0x5f | 0x08 | **0x13** | 0x13 | 0x13 |

So MTD is roughly 200 traces for a single byte and ~1000 for the full key.
From a recovered round key 10 the master key is obtained by running the key
schedule backwards, so the unprotected core is fully broken.

### 3. Second-order TVLA on the masked core, N = 20000

Centred-and-squared traces: **max abs(t2) = 70.78**, 10 of 12 samples above 4.5.

This is the honest and interesting result. The uniform byte mask makes the
*mean* of the Hamming distance independent of the data (first-order secure) but
not the *variance*, because all 16 state bytes share the same mask byte. So
first-order masking with a shared mask buys first-order security only; a
second-order attacker with enough traces still sees it. Fixing it means
independent per-byte masks, which multiplies the table-recomputation cost —
which is exactly the cost/security trade-off the thesis is about.

### 4. Cost

Latency, exact from simulation:

| core | cycles per encryption | overhead |
|---|---|---|
| unprotected | 11 | 1.0x |
| masked | 268 (256 recompute + 1 load + 10 rounds + 1 unmask) | **24.4x** |

If the table is recomputed once per *n* blocks instead of once per block the
amortised cost drops to `11 + 256/n` cycles, at the price of reusing
`(m_in, m_out)` across blocks — another trade-off axis to measure.

Area: run `scripts/synth.tcl` and fill in from
`reports/aes_unprot_util.rpt` and `reports/aes_masked_util.rpt`. The dominant
term in the masked core is the 16 x 256 x 8 bit masked table (4 KB), which
Vivado will map to either distributed RAM or block RAM; the unprotected core
needs only the single S-box ROM. Report LUTs, FFs, BRAMs and Fmax for both.

## How to run

```bash
# 1. traces (from aes_sca/)
bash scripts/run_sim.sh          # needs xvlog/xelab/xsim on PATH

# 2. analysis
cd rtl
python ../py/analyze.py traces_unprot.csv --tvla
python ../py/analyze.py traces_unprot.csv --cpa --sample 10
python ../py/analyze.py traces_masked.csv --tvla
python ../py/analyze.py traces_masked.csv --tvla --order 2

# 3. area / timing
cd ../scripts && vivado -mode batch -source synth.tcl

# no Vivado? reproduce every number above from the Python twin:
python py/reproduce_results.py
```

## What to say about this in the motivation letter

Do not describe the code. Describe the *question*: "how much security does a
countermeasure actually buy, per LUT and per cycle, and under which leakage
model does that claim hold?" Then note that you have already built a
simulation-level version of the flow — AES core, leakage model, TVLA, CPA,
countermeasure, cost accounting — and that the natural next step is to redo it
on real silicon measurements with EmLab's FPGA hardware and CRoCS's evaluation
setup, and to extend it from AES to a post-quantum primitive such as Kyber,
where masking costs are far less well characterised. That is a two-year
master's thesis with a clear first deliverable.
