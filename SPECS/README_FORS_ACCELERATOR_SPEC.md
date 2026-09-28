### 7.2 Cycle Latency Model

Let $C_{\text{perm}}$ denote the measured cycle latency of one 24-round Keccak-f[1600] permutation in the integrated `hash_engine`. Let $C_{\text{absorb}}$ denote the adapter handshake and padding overhead per rate block.

The isolated transaction cost for single-block evaluations ($\text{PRF}, F, H$) is modeled as:

$$
C_{\text{hash}} = C_{\text{perm}} + C_{\text{absorb}}
$$

The root compression cost over 5 rate blocks is modeled as:

$$
C_{T_k} = 5 \cdot (C_{\text{perm}} + C_{\text{absorb}})
$$

Total execution latency is then formulated as:

$$
\begin{aligned}
\text{Cycles}_{\text{SIGN}} &= \left( 2k \cdot 2^a + k(2^a - 1) \right) \cdot C_{\text{hash}} + C_{T_k} + \text{Overhead}_{\text{FSM}} \\
\text{Cycles}_{\text{PK\_FROM\_SIG}} &= \left( k(a + 1) \right) \cdot C_{\text{hash}} + C_{T_k} + \text{Overhead}_{\text{FSM}}
\end{aligned}
$$

For illustration only, under an idealized iterative permutation core where $C_{\text{perm}} = 24\text{ cycles}$ and $C_{\text{absorb}} \approx 2\text{ cycles}$ ($C_{\text{hash}} = 26\text{ cycles}$):
- `FORS_SIGN` idealized computation: $\approx$ **151,600 clock cycles**.
- `FORS_PK_FROM_SIG` idealized computation: $\approx$ **5,700 clock cycles**.

*(These numbers are an illustrative analytical model, not a verified timing signoff claim for the group's IP).*
