# Single-Lane FORS Accelerator Core Specification
## Standalone Hardware Architecture, Interface Contracts, and Verification Signoff

**Target Algorithm:** NIST FIPS 205 FORS (Forest of Random Subsets) — SLH-DSA-SHAKE-128f (Baseline) and 256f (Scalable)  
**Document Version:** 1.0  
**Document Status:** Formal Standalone Core Specification & Design Contract  
**Design Scope:** Standalone FORS IP Core only (Decoupled from SoC, DMA, and external bus architectures)  
**Target Platforms:** Vendor-Neutral SystemVerilog (AMD Artix-7 / Intel Cyclone IV EP4CE115 "DE2-115")  

`shall`, `shall not`, `must`, and `must not` state mandatory requirements. `should` states recommendations. `TBD` identifies unresolved design points. `BASELINE` identifies architectural choices proposed by this specification.

---

## 1. PURPOSE AND SCOPE

This specification establishes the hardware architecture, functional requirements, signal protocols, and verification gates for a parameterizable **Single-Lane FORS Hardware Accelerator Core** compliant with **NIST FIPS 205 (Stateless Hash-Based Digital Signature Standard)**.

The core executes both signature generation (`fors_sign`, Algorithm 15) and candidate public key reconstruction (`fors_pkFromSig`, Algorithm 16). The design is strictly confined to the standalone FORS primitive. It does not assume an enclosing XMSS hypertree context and intentionally decouples top-level SoC DMA or memory-mapped bus fabrics, exposing standardized, latency-insensitive **valid/ready streaming handshakes with packet delimiters (`last`)**.

| Hardware Opcode (`cmd_op`) | FIPS 205 Algorithm | Inputs (`din`) | Outputs (`dout`) |
|---|---|---|---|
| **`FORS_SIGN` (`2'b00`)** | Algorithm 15 (`fors_sign`) | Context (80 B) | FORS signature ($k(a+1)n$ bytes) + Root PK ($n$ bytes) |
| **`FORS_PK_FROM_SIG` (`2'b01`)** | Algorithm 16 (`fors_pkFromSig`) | Context (80 B) + Signature ($k(a+1)n$ bytes) | Reconstructed candidate Root PK ($n$ bytes) |
| **`FORS_ABORT` (`2'b10`)** | Hardware Control | None | None (triggers immediate pipeline halt and zeroization) |

### 1.1 Four-Component Subsystem Architecture

To ensure strict modularity and clear verification boundaries, the standalone core is partitioned into four internal components:

| Component | Primary Technical Responsibilities |
|---|---|
| **`fors_controller`** | Master FSM, command decoding, streaming input/output beat counting, message digest bit-unpacking into target leaf indices, outer tree loop sequencing ($i = 0 \dots k-1$), ADRS coordinate derivation, and sibling authentication node detection. |
| **`fors_memory`** | Synchronous register storage: Context buffer ($PK.seed, SK.seed, ADRS, md$), in-place tree-hash stack registers of depth $a$ (Zero-BRAM), root accumulation array ($k \times n$ bytes), and hardware zeroization scrubbing logic. |
| **`hash_adapter_fors`** | Formats canonical 48-byte hash prefixes ($PK.seed \parallel ADRS$), constructs rate blocks for PRF, Leaf, Node, and multi-block root compression $T_k$, applies FIPS 202 padding, and isolates combinatorial timing paths. |
| **`hash_engine`** | Cryptographic permutation core (Keccak-f[1600] / SHAKE256) supporting single-block and multi-block absorption across all operations. |

### 1.2 Normative References and Precedence

1. **NIST FIPS 205 (August 2024)**: *Stateless Hash-Based Digital Signature Standard*. Controls algorithmic correctness.
2. **NIST FIPS 202 (August 2015)**: *SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions*. Controls SHAKE256 sponge operations.
3. **Bai et al. (IEEE AsianHOST 2025)**: *FPGA-Accelerated SPHINCS+: An Area-Optimized and High-Throughput Design*. (DOI: [`10.1109/AsianHOST68425.2025.11370338`](https://doi.org/10.1109/AsianHOST68425.2025.11370338)).
4. **SPHINCSLET (ACM TECS 2024)**: *SPHINCSLET: An Area-Efficient Accelerator for the Full SPHINCS+ Digital Signature Algorithm*. (DOI: [`10.1145/3728469`](https://doi.org/10.1145/3728469)).
5. **Amiet et al. (IEEE DSD 2020)**: *FPGA-based PQC Accelerator Benchmarking on Cyclone IV*. (DOI: [`10.1109/DSD51259.2020.00046`](https://doi.org/10.1109/DSD51259.2020.00046)).

FIPS 205 controls algorithmic behavior. This specification controls the RTL microarchitecture and signal contracts. Academic publications serve as informative benchmarks; neither overrides FIPS 205.

### 1.3 Non-Goals

- Implementation of XMSS, WOTS+, hypertree addressing, or top-level SLH-DSA envelope framing.
- Full SoC memory-mapped buses (AXI4-MM, Avalon-MM, DMA controllers, or external DRAM interfaces); the core exclusively exposes streaming handshakes.
- Silicon-level physical side-channel security (power/EM masking, DPA hiding, or glitch fault injection resistance) in baseline release 1.0 (`NOT CLAIMED`).

---

## 2. CONFIGURATION AND REPRESENTATION

### 2.1 Fixed Algorithm Parameters

The datapath supports compile-time parameterization matching FIPS 205:

| Symbol | Parameter Meaning | `SLH-DSA-SHAKE-128f` (Baseline) | `SLH-DSA-SHAKE-256f` (Scalable) |
|---|---|:---:|:---:|
| **$n$** | Security parameter / hash digest length | **16 bytes** | **32 bytes** |
| **$k$** | Number of independent FORS subtrees | **33** | **35** |
| **$a$** | Height of each individual subtree | **6** | **9** |
| **$2^a$** | Leaves evaluated per subtree | **64** | **512** |
| **$ka$** | Total bits extracted from message digest $md$ | **198 bits** | **315 bits** |
| **$\lceil ka/8 \rceil$** | Raw message digest slice $md$ | **25 bytes** | **40 bytes** |
| **$Ctx$** | Aligned Context block size ($PK.seed + SK.seed + ADRS + md + pad$) | **80 bytes** | **112 bytes** |
| **$\sigma_{FORS}$** | FORS Signature payload size ($k(a+1)n$) | **3,696 bytes** | **11,200 bytes** |
| **$T_k$ Input** | Unpadded root compression message ($n + 32 + kn$) | **576 bytes** | **1,184 bytes** |
| **SHAKE Rate** | Sponge rate block width ($r$) | **136 bytes** (1,088 bits) | **136 bytes** (1,088 bits) |

### 2.2 Baseline Microarchitecture Parameters

| Parameter | Baseline Value | Description |
|---|:---:|---|
| `NUM_LANES` | 1 | Single-lane sequential tree processing ($M = 1$) |
| `STACK_DEPTH` | $a$ ($6$ or $9$) | Number of in-place tree-hash stack registers (Zero-BRAM) |
| `STREAM_WIDTH` | 64 bits | Native width of `din_data` and `dout_data` |
| `HASH_WIDTH` | 64 bits | Internal streaming bus between adapter and hash engine |
| `PIPELINE_STAGES`| 1 | Register slice isolation stages on handshake boundaries |

### 2.3 Byte and Bit Ordering Conventions

The specification distinguishes between three separate data-ordering levels: **streaming bus byte placement**, **ADRS multi-byte integer representation**, and **message digest bit extraction**.

#### FORS-REP-001 (Bus Byte Placement)
Byte arrays—including $PK.seed$, $SK.seed$, signature nodes, and public keys—retain their canonical FIPS 205 byte order. Across the 64-bit streaming bus (`din_data[63:0]` and `dout_data[63:0]`), the earliest byte of an array (Byte 0) is transferred in bits `[7:0]`, Byte 1 in bits `[15:8]`, and Byte 7 in bits `[63:56]`. This is purely an interconnect transport convention and does not alter the logical order of the underlying byte sequence.

#### FORS-REP-002 (ADRS Word Representation)
All 32-bit integer fields within the ADRS structure (`Layer Address`, `Address Type`, `Key-Pair Address`, `Tree Height`, and `Tree Index`) are encoded as canonical big-endian four-byte strings as mandated by FIPS 205 Section 4.3. The most significant byte of each 32-bit field is stored at the lowest byte offset.

#### FORS-REP-003 (Message Digest Bit Extraction)
The message digest $md$ is an ordered array of sequential bytes ($md[0], md[1], \dots, md[\lceil ka/8 \rceil - 1]$), output directly by the enclosing hash function. It is not a multi-byte integer and does not possess endianness.

To derive the $k$ target leaf indices ($idx_0, \dots, idx_{k-1}$), the hardware processes $md$ as a continuous bitstream following FIPS 205 Algorithm 4 (`base_2b`):

1. **Byte Traversal:** Bytes are consumed in strict ascending order from left to right: $md[0]$, followed by $md[1]$, $md[2]$, and so on.
2. **Bit Consumption Within Each Byte:** Bits within any single byte are read from the Most Significant Bit (MSB, bit 7) down to the Least Significant Bit (LSB, bit 0).
3. **Cross-Byte Index Slicing:** Each leaf index $idx_i$ is formed by taking the next $a$ consecutive bits from this stream to form an unsigned integer.
4. **Boundary Realignment:** Because tree height $a$ (6 bits for 128f, or 9 bits for 256f) does not divide 8, indices naturally cross byte boundaries:
   - Index $0$ ($idx_0$) consists of the top 6 bits of $md[0]$ (bits 7 down to 2).
   - Index $1$ ($idx_1$) consists of the remaining 2 bits of $md[0]$ (bits 1 down to 0) acting as the high bits, concatenated with the top 4 bits of $md[1]$ (bits 7 down to 4) acting as the low bits.
   - This sequential extraction continues without gaps until all $k$ indices are populated.

---

## 3. TOP-LEVEL ARCHITECTURE & COMPONENT SPECIFICATION

The single-lane FORS accelerator core is composed of four internal components: `fors_controller`, `fors_memory`, `hash_adapter_fors`, and `hash_engine`.

### 3.1 Component Descriptions and Interconnect Logic

1. **`fors_controller` (Central Control & Stream Engine):**
   - **External Interfaces:** Directly drives and samples command signals (`cmd_*`), streaming input (`din_*`), and streaming output (`dout_*`).
   - **Stream Ingress & Slicing:** Ingests the 80-byte context header from `din_data`, writes the seeds and base ADRS into `fors_memory`, and feeds $md$ bytes into an internal shift register to unpack $k$ target leaf indices ($idx_0 \dots idx_{k-1}$).
   - **Tree Traversal Coordination:** Drives outer loop counter $i \in [0, k-1]$ and leaf counter $j \in [0, 2^a - 1]$. In `FORS_SIGN` mode, coordinates PRF key expansion, leaf hashing, and stack merge operations. In `FORS_PK_FROM_SIG` mode, ingests signature nodes from `din` and controls direct upward path climbing.
   - **Stream Egress:** In `FORS_SIGN`, streams revealed leaf keys ($sk_i$) and authentication path siblings onto `dout`. Once all $k$ trees are processed and compressed, streams final Root PK onto `dout`.

2. **`fors_memory` (Synchronous Storage & Stack Arrays):**
   - **Context Registers:** Dedicated flip-flop storage for $PK.seed$ (16B), $SK.seed$ (16B), base $ADRS$ (32B), and target leaf indices ($33 \times 6\text{ bits}$).
   - **In-Place Tree-Hash Stack:** Consists of exactly $a$ register slots of $n$ bytes (`REG_STACK[a-1:0]`). During tree traversal, parent nodes replace child nodes in-place. Total Block RAM consumed for tree traversal: **0 BRAM blocks**.
   - **Root Accumulator Buffer:** Register bank storing the $k$ committed tree roots ($33 \times 16\text{ B} = 528\text{ B}$ for 128f).
   - **Hardware Scrubbing:** Synchronously zeros all secret-bearing registers ($SK.seed$, secret leaf keys, stack) within 2 clock cycles of `FORS_ABORT` or operation completion.

3. **`hash_adapter_fors` (Message Formatting & Prefix Construction):**
   - **Prefix Assembly:** Prepends the canonical 48-byte prefix ($PK.seed \parallel ADRS$) to every hash message before absorbing payload bytes.
   - **Operation Multiplexing:** Selects appropriate data sources: PRF secret seed (16B), leaf hash key (16B), node hash child pair (32B), or $T_k$ root buffer (528B across 5 rate blocks).
   - **Padding Insertion:** Appends FIPS 202 delimiter bits (`0x1F`) and multi-rate padding.
   - **Pipeline Isolation:** Embeds register slices on handshake boundaries to prevent combinational path compounding between controller logic and the permutation core.

4. **`hash_engine` (Cryptographic Permutation Core):**
   - Implements the 24-round Keccak-f[1600] / SHAKE256 permutation.
   - Exposes a 64-bit unpadded streaming absorb interface and rate-squeezing output port.
   - Sequentially reused across all PRF, Leaf, Node, and Root compression operations without resource duplication.

### 3.2 Storage Lifetime and Ownership

| Storage Object | Logical Capacity (`128f`) | Producer $\to$ Consumer | Lifetime / Policy |
|---|---:|---|---|
| `REG_PK_SEED` | 16 bytes | `din` stream $\to$ `hash_adapter_fors` | Retained across entire command; non-secret |
| `REG_SK_SEED` | 16 bytes | `din` stream $\to$ `hash_adapter_fors` | Active during `FORS_SIGN`; zeroized immediately upon completion or abort |
| `REG_BASE_ADRS` | 32 bytes | `din` stream $\to$ `fors_controller` | Retained across entire command; immutable |
| `REG_TARGET_IDX`| $33 \times 6$ bits | Unpacker $\to$ `fors_controller` | Latched during context ingest; cleared upon command finish |
| `REG_STACK` | $6 \times 16$ bytes | `fors_controller` tree traversal | Active per tree; cleared before starting tree $i+1$ |
| `REG_ROOT_BUFFER`| $33 \times 16$ bytes | `fors_controller` $\to$ `hash_adapter_fors` | Stores roots $0 \dots k-1$; consumed by $T_k$; overwritten on next job |

---

## 4. FUNCTIONAL REQUIREMENTS & DATAPATH CONTRACT

| Requirement ID | Formal Functional Contract |
|---|---|
| **FORS-FUNC-001** | The accelerator shall execute `FORS_SIGN` (Algorithm 15) and `FORS_PK_FROM_SIG` (Algorithm 16) compliant with FIPS 205. |
| **FORS-FUNC-002** | The accelerator shall latch operation mode, parameters, and 80-byte context exactly once per accepted command. |
| **FORS-FUNC-003** | Unsupported opcodes or parameters shall terminate with an error without modifying internal storage or exposing stale output. |
| **FORS-FUNC-004** | Input stream framing shall be validated; premature or missing `din_last` assertions shall halt execution and report error. |
| **FORS-FUNC-005** | Tree traversal shall evaluate all $k$ trees sequentially ($0 \le i < k$) without skipping or repeating trees. |
| **FORS-FUNC-006** | In `FORS_SIGN`, exactly $k$ secret keys and $k \times a$ authentication siblings shall be emitted in canonical FIPS order. |
| **FORS-FUNC-007** | In `FORS_PK_FROM_SIG`, tree roots shall be reconstructed from incoming signature nodes and compressed via $T_k$ to produce candidate PK. |
| **FORS-FUNC-008** | Final output public key `fors_pk` shall match an approved FIPS 205 reference model for identical inputs. |
| **FORS-FUNC-009** | Hardware abort (`FORS_ABORT`) shall immediately halt all execution, flush FIFOs, and zeroize secret-bearing registers. |
| **FORS-FUNC-010** | Uninitialized storage contents shall never be utilized as algorithmic inputs. |

### 4.1 Input Stream (`din`) Formatting

Payloads are ingested over `din_data[63:0]`:
1. **In `FORS_SIGN` Mode (`cmd_op = 2'b00`)**:
   - Total length: **80 bytes** (10 beats of 64 bits).
   - Beats 0–1: `PK.seed` (16B). Beats 2–3: `SK.seed` (16B). Beats 4–7: Base ADRS (32B). Beats 8–9: Digest $md$ (25B + 7B pad).
   - `din_last` **shall** be asserted on Beat 9.
2. **In `FORS_PK_FROM_SIG` Mode (`cmd_op = 2'b01`)**:
   - Total length: **80 bytes Context + 3,696 bytes Signature** (472 beats for `128f`).
   - Beats 0–9: Context block (`SK.seed` ignored/zeroed). Beats 10–471: Signature nodes ($k(a+1)n$ bytes).
   - `din_last` **shall** be asserted on Beat 471.

### 4.2 Output Stream (`dout`) Formatting

Results are emitted over `dout_data[63:0]`:
1. **In `FORS_SIGN` Mode (`cmd_op = 2'b00`)**:
   - Total length: **3,696 bytes Signature + 16 bytes Root PK** (464 beats for `128f`).
   - Beats 0–461: Signature stream ($sk_i \parallel auth_{i,0} \dots auth_{i,a-1}$ for $i = 0 \dots k-1$).
   - Beats 462–463: Compressed Root PK `fors_pk`. `dout_last` asserted on Beat 463.
2. **In `FORS_PK_FROM_SIG` Mode (`cmd_op = 2'b01`)**:
   - Total length: **16 bytes Candidate Root PK** (2 beats for `128f`). `dout_last` asserted on Beat 1.

### 4.3 ADRS Generation Contract

All ADRS fields are big-endian byte strings compliant with FIPS 205:

| Byte Offset | Field Name | Width | Description / Value |
|---|---|:---:|---|
| 0–3 | Layer Address | 32-bit | Fixed at `0` for FORS |
| 4–15 | Tree Address | 96-bit | Canonical tree address from base ADRS |
| 16–19 | Address Type | 32-bit | `FORS_PRF = 6`, `FORS_TREE = 3`, `FORS_ROOTS = 4` |
| 20–23 | Key-Pair Address | 32-bit | Keypair address from base ADRS |
| 24–27 | Tree Height ($h$) | 32-bit | Node height $0 \dots a$ |
| 28–31 | Tree Index | 32-bit | Global node index across forest |

| Function | ADRS Type Field | Tree Height ($h$) | Tree Index Field |
|---|:---:|:---:|---|
| **$\text{PRF}$** (Secret leaf key) | `FORS_PRF = 6` | `0` | $i \cdot 2^a + j$ |
| **$F$** (Leaf node hash) | `FORS_TREE = 3` | `0` | $i \cdot 2^a + j$ |
| **$H$** (Internal node merge) | `FORS_TREE = 3` | Height $h \in [1, a]$ | $i \cdot 2^{a-h} + \lfloor j / 2^h \rfloor$ |
| **$T_k$** (Root compression) | `FORS_ROOTS = 4` | `0` | `0` |

- **FORS-ADRS-001**: Forming an ADRS type shall have the effect of setting type, restoring keypair address, and populating height/index. Retaining stale height/index data from previous levels is strictly prohibited.
- **FORS-ADRS-002**: `fors_controller` derives ADRS combinatorially from base registers, tree counter $i$, leaf counter $j$, and height $h$.

### 4.4 Hash Operations in FORS

```text
PRF    = SHAKE256_n(PK.seed || ADRS_FORS_PRF   || SK.seed)
F(sk)  = SHAKE256_n(PK.seed || ADRS_FORS_TREE  || sk)
H(L,R) = SHAKE256_n(PK.seed || ADRS_FORS_TREE  || left || right)
T_k    = SHAKE256_n(PK.seed || ADRS_FORS_ROOTS || root_0 || root_1 || ... || root_{k-1})
```

| Operation | Total Message Bytes (`128f`) | Rate Block Absorption | Squeeze Output |
|---|:---:|:---:|:---:|
| **$\text{PRF}$** | $16\text{B } (PK.seed) + 32\text{B } (ADRS) + 16\text{B } (SK.seed) = \mathbf{64\text{ B}}$ | 1 block (64 B $\le 136$ B) | $n = 16$ bytes |
| **$F$ (Leaf)** | $16\text{B } (PK.seed) + 32\text{B } (ADRS) + 16\text{B } (sk) = \mathbf{64\text{ B}}$ | 1 block (64 B $\le 136$ B) | $n = 16$ bytes |
| **$H$ (Node)** | $16\text{B } (PK.seed) + 32\text{B } (ADRS) + 32\text{B } (left \parallel right) = \mathbf{80\text{ B}}$ | 1 block (80 B $\le 136$ B) | $n = 16$ bytes |
| **$T_k$ (Roots)** | $16\text{B } (PK.seed) + 32\text{B } (ADRS) + 528\text{B } (33 \text{ roots}) = \mathbf{576\text{ B}}$ | 5 blocks ($\lceil 577/136 \rceil$) | $n = 16$ bytes |

- **FORS-HASH-001**: Pre-absorption prefix ($PK.seed \parallel ADRS$) occupies the first 48 bytes of every hash message.
- **FORS-HASH-002**: Every $\text{PRF}$, $F$, and $H$ invocation starts from an empty, freshly initialized sponge state.
- **FORS-HASH-003**: $T_k$ root compression executes as a single continuous multi-block hash across all 5 rate blocks without intermediate re-initialization.

---

## 5. INTERFACE AND SIGNAL CONTRACT

### 5.1 Signal Definitions

| Signal Group | Port Name | Width | Dir | Description |
|---|---|:---:|:---:|---|
| **Global** | `clk` | 1 | In | Master synchronous clock |
| | `rst_n` | 1 | In | Synchronous active-low reset |
| **Command** | `cmd_valid` | 1 | In | Command request strobe |
| | `cmd_ready` | 1 | Out | Core ready for new command |
| | `cmd_op[1:0]` | 2 | In | `00`: SIGN, `01`: PK_FROM_SIG, `10`: ABORT |
| | `cmd_param` | 1 | In | Parameter set (`0`: 128f, `1`: 256f) |
| | `cmd_done` | 1 | Out | Single-cycle completion pulse on success |
| | `cmd_error` | 1 | Out | Single-cycle pulse on abnormal termination |
| | `cmd_error_code[1:0]`| 2 | Out | Latched error status |
| **Input Stream** | `din_valid` | 1 | In | Input stream beat valid |
| | `din_ready` | 1 | Out | Core ready for input beat |
| | `din_data[63:0]` | 64 | In | 64-bit input data beat |
| | `din_last` | 1 | In | End-of-packet delimiter |
| **Output Stream**| `dout_valid` | 1 | Out | Output stream beat valid |
| | `dout_ready` | 1 | In | Consumer ready for output beat |
| | `dout_data[63:0]`| 64 | Out | 64-bit output data beat |
| | `dout_last` | 1 | Out | End-of-packet delimiter |

### 5.2 Handshake Requirements

| Requirement ID | Formal Handshake Rule |
|---|---|
| **FORS-CMD-001** | A command transfer occurs if and only if `cmd_valid && cmd_ready` is asserted on the rising clock edge. |
| **FORS-CMD-002** | All command lines (`cmd_op`, `cmd_param`) shall remain stable while `cmd_valid` is high and `cmd_ready` is low. |
| **FORS-CMD-003** | `cmd_done` and `cmd_error` shall be mutually exclusive pulses. |
| **FORS-CMD-004** | Streaming beats transfer if and only if `valid && ready` is true. Stalled data lines shall remain completely stable. |
| **FORS-CMD-005** | Deasserting `dout_ready` (backpressure) shall stall the traversal FSM gracefully without dropping data. |
| **FORS-CMD-006** | Out-of-order completion is prohibited; completion corresponds strictly to the active accepted command. |

---

## 6. CLOCK, RESET, AND ERROR HANDLING

| Requirement ID | Formal Clock / Reset Rule |
|---|---|
| **FORS-CLK-001** | The entire FORS core operates on a single synchronous clock domain `clk`. |
| **FORS-CLK-002** | Target frequency: $F_{max} \ge 150\text{ MHz}$ on AMD Artix-7; $F_{max} \ge 100\text{ MHz}$ on Intel Cyclone IV DE2-115. |
| **FORS-RST-001** | System reset `rst_n` is active-low, asserted asynchronously, and deasserted synchronously with `clk`. |
| **FORS-CDC-001** | Clock-domain crossings (if attached to a different system clock) shall use reviewed dual-clock FIFOs with Gray-coded pointers. |
| **FORS-RDC-002** | Reset deassertion shall be synchronized to prevent meta-stability in state machines. |

### 6.1 Error Codes (`cmd_error_code[1:0]`)

| Code | Symbol | Trigger Condition |
|:---:|---|---|
| `2'b00` | `ERR_NONE` | Successful execution |
| `2'b01` | `ERR_INVALID_OP` | Undefined opcode received on `cmd_op` |
| `2'b10` | `ERR_STREAM_MISMATCH` | Premature or missing `din_last` delimiter |
| `2'b11` | `ERR_ABORTED` | Host asserted `cmd_op = FORS_ABORT` |

---

## 7. PERFORMANCE AND RESOURCE REQUIREMENTS

No cycle, frequency, LUT, BRAM, or speedup number below is an achieved result. They represent mathematical work counts, parameterized latency formulations, and post-synthesis verification reporting requirements.

### 7.1 Exact Mathematical Work Counts

The cryptographic work performed by the hardware is strictly determined by FIPS 205 parameters and is independent of microarchitectural clock cycle latency:

| Operation Mode | PRF Invocations | Leaf Hash ($F$) Invocations | Node Hash ($H$) Invocations | Root Hash ($T_k$) Invocations | Total Keccak Permutations (`128f`) |
|---|:---:|:---:|:---:|:---:|:---:|
| **`FORS_SIGN`** | $k \cdot 2^a$ ($2,112$) | $k \cdot 2^a$ ($2,112$) | $k(2^a - 1)$ ($2,079$) | $1$ ($5$ rate blocks) | **6,308** |
| **`FORS_PK_FROM_SIG`** | 0 | $k \cdot 1$ ($33$) | $k \cdot a$ ($198$) | $1$ ($5$ rate blocks) | **236** |

### 7.2 Cycle Latency Model

Let $C_{\text{perm}}$ denote the measured cycle latency of one 24-round Keccak-f[1600] permutation in the integrated `hash_engine`. Let $C_{\text{absorb}}$ denote the adapter handshake and padding overhead per rate block.

The isolated transaction cost for single-block evaluations ($\text{PRF}, F, H$) is modeled as:
$$C_{\text{hash}} = C_{\text{perm}} + C_{\text{absorb}}$$

The root compression cost over 5 rate blocks is modeled as:
$$C_{T_k} = 5 \cdot (C_{\text{perm}} + C_{\text{absorb}})$$

Total execution latency is then formulated as:

$$
\text{Cycles}_{\text{SIGN}} = \left( 2k \cdot 2^a + k(2^a - 1) \right) \cdot C_{\text{hash}} + C_{T_k} + \text{Overhead}_{\text{FSM}}
$$

$$
\text{Cycles}_{\text{PK-FROM-SIG}} = \left( k(a + 1) \right) \cdot C_{\text{hash}} + C_{T_k} + \text{Overhead}_{\text{FSM}}
$$

For illustration only, under an idealized iterative permutation core where $C_{\text{perm}} = 24\text{ cycles}$ and $C_{\text{absorb}} \approx 2\text{ cycles}$ ($C_{\text{hash}} = 26\text{ cycles}$):
- `FORS_SIGN` idealized computation: $\approx \mathbf{151,600\text{ clock cycles}}$.
- `FORS_PK_FROM_SIG` idealized computation: $\approx \mathbf{5,700\text{ clock cycles}}$.
*(These numbers are an illustrative analytical model, not a verified timing signoff claim for the group's IP).*

### 7.3 Stream Traffic Sizing

For successful operations without bus stalls:

| Operation Mode | Input Bytes Read (`din`) | Algorithm Output Bytes (`dout`) | Delimiter (`last`) Asserted On |
|---|:---:|:---:|:---:|
| **`FORS_SIGN`** | 80 bytes (10 beats) | 3,712 bytes (464 beats: 3,696B Sig + 16B PK) | Beat 9 (`din`), Beat 463 (`dout`) |
| **`FORS_PK_FROM_SIG`** | 3,776 bytes (472 beats: 80B Ctx + 3,696B Sig) | 16 bytes (2 beats: Candidate PK) | Beat 471 (`din`), Beat 1 (`dout`) |

### 7.4 Required Implementation Reports and Synthesis Deliverables

| Requirement ID | Formal Verification & Reporting Obligation |
|---|---|
| **FORS-PERF-001** | The implementation release shall report measured cycle counts, post-synthesis and post-route utilization (LUTs, FFs, BRAMs, and DSPs), operating frequency ($F_{max}$), and worst negative slack ($WNS$) for both AMD Artix-7 and Intel Cyclone IV. |
| **FORS-PERF-002** | The synthesis report shall demonstrate Zero-BRAM utilization in the `lane-fors` and `fors_memory` tree-traversal logic. Memory blocks may only be instantiated in top-level FIFO buffers if explicitly configured. |
| **FORS-PERF-003** | The design team shall provide waveform evidence showing backpressure stall recovery, continuous streaming throughput, and hardware zeroization timing. |

---

## 8. SECURITY REQUIREMENTS

| Requirement ID | Security Principle | Hardware Enforcement Rule |
|---|---|---|
| **FORS-SEC-001** | **Constant Execution Time** | The hardware shall always evaluate all $k$ trees and all $2^a$ leaves deterministically. It shall never exit early based on secret key or message values, preventing timing attacks. |
| **FORS-SEC-002** | **Automatic Key Scrubbing** | Within 2 clock cycles of completing an operation, aborting, or resetting, all internal registers holding $SK.seed$ and ephemeral secret leaf keys must be automatically overwritten with zeros. |
| **FORS-SEC-003** | **No Secret Readback** | Secret registers are write-only. Hardware shall provide no internal wiring or command path for the host CPU or debug ports to read back $SK.seed$. |
| **FORS-SEC-004** | **No Leakage on Error** | If an error, stream mismatch, or abort occurs, the core shall immediately close the output stream (`dout`) and output zero bytes. Partial or corrupted signature data shall never be emitted. |
| **FORS-SEC-005** | **Physical Side-Channel Boundary** | This design guarantees algorithmic correctness and constant-time execution. Resistance to physical attacks (measuring power consumption or electromagnetic radiation) is NOT CLAIMED in this release. |

---

## 9. VERIFICATION AND TRACEABILITY

### 9.1 Reference Strategy
Verification utilizes the official [NIST FIPS 205 reference implementation (C)](https://github.com/pq-code-package/slhdsa-c) as the golden software oracle. Testbenches compare intermediate digests, stack states, and final signature outputs byte-by-byte against the C model.

### 9.2 Verification Matrix

| Test ID | Requirement Coverage | Stimulus and Pass Condition |
|---|---|---|
| **V01** | `REP-001..003`, `HASH-001..003` | FIPS 202 SHAKE256 KATs; multi-block 576-byte $T_k$ absorption test |
| **V02** | `FUNC-002`, `REP-003` | Digest bit-slicing verification: zero, all-ones, alternating bit patterns |
| **V03** | `ADRS-001..002` | Canonical ADRS generation across all 33 trees and heights $0 \dots a$ |
| **V04** | `FUNC-001`, `FUNC-008` | Full NIST FIPS 205 KAT verification for `FORS_SIGN` and `FORS_PK_FROM_SIG` |
| **V05** | `CMD-004`, `CMD-005` | Pseudo-random backpressure on `dout_ready` (10% to 90% duty cycles); zero data loss |
| **V06** | `FUNC-004`, `CMD-002` | Delimiter fault injection: premature and missing `din_last`; verify `ERR_STREAM_MISMATCH` |
| **V07** | `FUNC-009`, `SEC-002..004` | Mid-execution `FORS_ABORT` assertion; verify 2-cycle halt, stream closure, and zeroization |
| **V08** | `FUNC-003`, `CMD-003` | Reserved opcode injection; verify immediate rejection and `ERR_INVALID_OP` |
| **V09** | `CLK-001..002`, `RST-001` | Clock jitter tolerance, reset deassertion synchronization, post-route timing closure |

---

## 10. IMPLEMENTATION AND RELEASE SIGNOFF GATES

### 10.1 Acceptance Gates

| Gate | Required Evidence for Version 1.0 Signoff |
|---|---|
| **1. Specification Review** | Document reviewed and approved; parameter packages and signal contracts frozen |
| **2. Functional Closure** | 100% pass on all V01–V09 verification tests against FIPS 205 reference vectors |
| **3. Static Code Quality** | SpyGlass / Verilator lint clean (zero fatal warnings); structural CDC/RDC signoff |
| **4. Timing Closure** | Fully routed timing closure with non-negative Slack ($WNS \ge 0$) at target $F_{max}$ |
| **5. Resource Compliance**| Synthesis reports proving Zero-BRAM traversal and logic fit on Artix-7 and Cyclone IV |

### 10.2 Release Manifest
A release package shall include:
1. SystemVerilog RTL source files and filelists.
2. Self-checking testbench suite and NIST KAT regression scripts.
3. Lint, CDC, synthesis, and post-implementation timing reports.
4. Reproducible simulation and build scripts (Vivado / Quartus / Verilator).

---

## 11. OPEN DECISIONS & SCALING

| ID | Decision Item | Baseline Choice | Proposed Upgrade Path | Status |
|---|---|---|---|:---:|
| **FORS-OPEN-001** | Multi-Lane FORS Scaling ($M = 2$) | Single-Lane ($M = 1$) | Dual `lane-fors` cores assigning even/odd trees in parallel, cutting sign time by ~50% ([Trident, 2026](https://eprint.iacr.org/2026/837)) | Deferred |
| **FORS-OPEN-002** | Multi-Lane FORS Scaling ($M = 4$) | Single-Lane ($M = 1$) | 4-lane parallel architecture for high-end FPGAs (~38,000 cycles) | Open |
| **FORS-OPEN-003** | Generic Hash IP Plugability | SHAKE256 Engine | Modular adapter supports drop-in SHA-256 core for SHA-2 SLH-DSA | Open |
| **FORS-OPEN-004** | ASIC Synthesis & Signoff | FPGA Target | TSMC 28nm / SkyWater 130nm standard cell synthesis | Future Scope |

---

## APPENDIX A. RESEARCH BASIS AND DESIGN RATIONALE

### A.1 Relevant Hardware Studies

| Publication | Context & Evidence | Lessons Applied Here | Scope Distinction |
|---|---|---|---|
| **Bai et al. (IEEE AsianHOST 2025)**<br>DOI: [`10.1109/AsianHOST68425.2025.11370338`](https://doi.org/10.1109/AsianHOST68425.2025.11370338) | Section III-C, Fig. 6: Zero-BRAM tree traversal, pre-absorption prefix caching | In-place stack registers of depth $a$; 48B prefix pre-absorption; on-the-fly sibling streaming | Full SPHINCS+ design; FORS module extracted and adapted for standalone core |
| **SPHINCSLET (ACM TECS 2024)**<br>DOI: [`10.1145/3728469`](https://doi.org/10.1145/3728469) | Section 4, Table 4: Width Align Registers (WAR), skid buffers, throughput optimization | Register slices between FSM and hash core to isolate combinatorial paths and boost $F_{max}$ | Full-chip accelerator; our design isolates the FORS primitive with clean streaming interfaces |
| **Amiet et al. (IEEE DSD 2020)**<br>DOI: [`10.1109/DSD51259.2020.00046`](https://doi.org/10.1109/DSD51259.2020.00046) | Table 1: PQC benchmarks on Intel Cyclone IV | Baseline area and frequency targets on low-cost Intel FPGA devices | Pure Keccak evaluation; adapted here for complete FORS tree operations |
| **Trident (IACR ePrint 2026/837)**<br>URL: [https://eprint.iacr.org/2026/837](https://eprint.iacr.org/2026/837) | Section III: Multi-lane parallel tree hashing | Scalability model for $M=2$ and $M=4$ tree parallelization in Section 11 | XMSS hypertree focus; concepts applied to parallel FORS trees |

### A.2 Architectural Decisions Made by this Specification

| Architectural Choice | Technical Rationale | Alternative Deferred |
|---|---|---|
| **Single-Lane Sequential ($M=1$)** | Fits easily on budget FPGAs; Zero Block RAM consumed | Multi-lane parallel hashing (deferred to Section 11) |
| **Zero-BRAM In-Place Stack** | Depth $a$ registers consume negligible logic while eliminating RAM access bottlenecks | BRAM-based node storage buffers |
| **Streaming Valid/Ready with `last`** | Latency-insensitive, vendor-neutral integration with SoC DMA or FIFO wrappers | Rigid cycle-fixed bus interfaces |
| **Pre-Absorption 48B Prefix** | Caches $PK.seed \parallel ADRS$ across hash evaluations, saving absorb cycles | Re-absorbing full prefix on every hash |
| **Hash Core Reuse for $T_k$** | Sequentially reuses single hash engine for final root compression | Instantiating a dedicated second hash engine |
