# System-on-Chip (SoC) Specification for FORS Accelerator
## Concise Architecture, Subsystems, and Operational Specification

**Target Algorithm:** NIST FIPS 205 FORS — SLH-DSA-SHAKE-128f (Baseline) and 256f (Scalable)  
**Document Version:** 1.0  
**Scope:** Minimal, clear System-on-Chip (SoC) integration contract coupling CPU, Shared BRAM, DMA, and the standalone FORS IP.  

---

## 1. PURPOSE & SYSTEM OVERVIEW

The FORS IP accelerator is a pure streaming core (`din`/`dout`). Operating it directly from a CPU via word-by-word register writes would stall the processor for thousands of cycles. 

This SoC integrates the FORS IP into a complete, autonomous hardware system:
- **Host CPU:** Sets up cryptographic parameters and triggers operations.
- **Shared BRAM (True Dual-Port):** Staging memory holding input contexts, signatures, and public keys.
- **Register-Driven DMA Controller:** Automatically bursts data between BRAM and the FORS IP over a 64-bit streaming interface with zero CPU intervention.
- **FORS IP Core:** Standalone cryptographic engine performing tree hashing and public key computation.

---

## 2. TOP-LEVEL SoC ARCHITECTURE

```text
+---------------------------------------------------------------------------------------+
|                                      FORS SoC                                         |
|                                                                                       |
|   +---------------+              System Bus (AXI4-Lite / Simple MMIO)                 |
|   |   Host CPU    |<==================================================+               |
|   | (RISC-V/Host) |                                                   |               |
|   +-------+-------+                                                   v               |
|           |                                               +-----------------------+   |
|           | Port A (32-bit CPU RW)                        |  Register-Driven DMA  |   |
|           v                                               | - 11 MMIO Registers   |   |
|   +---------------+                                       | - Beat Counters       |   |
|   |  Shared BRAM  |                                       | - Interrupt Logic     |   |
|   |  (True Dual-  |                                       +-----------+-------^---+   |
|   |   Port RAM)   |                                                   |       |       |
|   +-------+-------+                                   64-bit Burst    |       |       |
|           ^                                           Read / Write    |       |       |
|           | Port B (64-bit DMA RW)                                    |       |       |
|           +-----------------------------------------------------------+       |       |
|                                                                               |       |
|                     din_data[63:0], din_valid, din_ready, din_last            |       |
|                     --------------------------------------------------------->|       |
|                                                                               |       |
|                     <---------------------------------------------------------+       |
|                     dout_data[63:0], dout_valid, dout_ready, dout_last                |
|                                         |                                             |
|                                         v                                             |
|                           +---------------------------+                               |
|                           |   FORS Accelerator IP     |                               |
|                           | - fors_controller         |                               |
|                           | - fors_memory             |                               |
|                           | - hash_adapter_fors       |                               |
|                           | - hash_engine (SHAKE256)  |                               |
|                           +---------------------------+                               |
+---------------------------------------------------------------------------------------+
```

---

## 3. SUBSYSTEM FUNCTIONS & RESPONSIBILITIES

### 3.1 Host CPU
- Prepares the 80-byte Context block (and candidate signature if verifying) in Shared BRAM.
- Programs DMA registers (`SRC_ADDR`, `DST_ADDR`, `CONFIG`) via MMIO and issues `START`.
- Enters low-power wait (WFI/WFE) or polls `STATUS.DONE`.
- Retrieves the final signature or public key from BRAM after the operation finishes.

### 3.2 Shared BRAM (True Dual-Port)
- **Port A (CPU Side):** 32-bit synchronous read/write port mapped to CPU memory address space.
- **Port B (DMA Side):** 64-bit synchronous read/write port dedicated to DMA burst transfers (1 cycle per 8-byte beat).
- Provides concurrent access: CPU can prepare data for the next operation while DMA is transferring data for the current operation.

### 3.3 Register-Driven DMA Controller
- **Autonomous Stream Bridge:** Converts sequential BRAM words into AXI4-Stream beats for `din`, and sinks `dout` beats directly back into BRAM.
- **Auto Beat Framing:**
  - In `FORS_SIGN`: Bursts 10 beats (80 bytes) of context and automatically asserts `din_last` on beat 9.
  - In `FORS_PK_FROM_SIG`: Bursts 10 beats of context followed immediately by 462 beats (3,696 bytes) of signature, asserting `din_last` on beat 471.
- **Stream Termination:** Watches for `dout_last` from the FORS IP to complete the write burst, record execution cycles, assert `STATUS.DONE`, and pulse the `irq` interrupt line.

### 3.4 FORS Accelerator IP
- Operates strictly on AXI4-Stream handshakes (`din`, `dout`, `cmd`).
- Decoupled from bus protocols, BRAM addressing, and CPU architecture.

---

## 4. BRAM MEMORY MAP & BUFFER SIZING

A minimal **8 KiB BRAM** allocation accommodates full single-slot operations for `SLH-DSA-SHAKE-128f`. A **16 KiB BRAM** enables dual-slot ping-pong execution:

| Address Offset Range | Buffer Size | Content / Purpose |
|---|:---:|---|
| `0x0000 - 0x004F` | 80 Bytes | **Context Header:** $PK.seed$ (16B), $SK.seed$ (16B), Base $ADRS$ (32B), $md$ (16B sliced). |
| `0x0050 - 0x007F` | 48 Bytes | Alignment padding to 128-byte boundary. |
| `0x0080 - 0x0EEF` | 3,696 Bytes | **Signature Buffer:** Output for `FORS_SIGN`; Input for `FORS_PK_FROM_SIG`. |
| `0x0EF0 - 0x0EFF` | 16 Bytes | **Public Key Buffer:** Output for `FORS_PK_FROM_SIG` (candidate Root PK). |
| `0x0F00 - 0x0FFF` | 256 Bytes | Guard band / Reserved. |
| `0x1000 - 0x1FFF` | 4 KiB | **Slot 1:** Mirror buffer for ping-pong pipelining (Optional). |

---

## 5. CONTROL & STATUS REGISTERS (CSR MAP)

The DMA controller exposes 11 memory-mapped 32-bit registers (Base Address: `0x4000_0000`):

| Offset | Register Name | Access | Reset | Description |
|:---:|---|:---:|:---:|---|
| `0x00` | `ID_VERSION` | RO | `0x464F5231` | ASCII `"FOR1"` (FORS IP Architecture v1.0). |
| `0x04` | `CONTROL` | W1P | `0x00000000` | Bit 0: `START`, Bit 1: `ABORT`, Bit 2: `SOFT_RESET`. |
| `0x08` | `STATUS` | RO | `0x00000000` | Bit 0: `BUSY`, Bit 1: `DONE`, Bit 2: `ERROR`, Bit 3: `ABORTED`. |
| `0x0C` | `ERROR_CODE` | RO | `0x00000000` | `0`: None, `1`: Length error, `2`: Protocol error, `3`: IP core error. |
| `0x10` | `SRC_ADDR` | RW | `0x00000000` | BRAM byte offset of Context Header (e.g., `0x0000`). |
| `0x14` | `SIG_SRC_ADDR`| RW | `0x00000000` | BRAM byte offset of Input Signature (for Verify mode, e.g., `0x0080`). |
| `0x18` | `DST_ADDR` | RW | `0x00000000` | BRAM byte offset for Output Destination (Signature or PK). |
| `0x1C` | `CONFIG` | RW | `0x00000001` | Bits `[3:0]`: `OP_MODE` (`1` = SIGN, `2` = VERIFY); Bit `[4]`: `PARAM` (`0` = 128f, `1` = 256f). |
| `0x20` | `IRQ_ENABLE` | RW | `0x00000000` | Bit 0: `DONE_IE` (Interrupt on completion); Bit 1: `ERR_IE` (Interrupt on error). |
| `0x24` | `IRQ_STATUS` | RW1C | `0x00000000` | Bit 0: `DONE_FLAG`; Bit 1: `ERR_FLAG`. Write 1 to clear. |
| `0x28` | `EXEC_CYCLES` | RO | `0x00000000` | Total hardware execution cycles measured from `START` to `DONE`. |

---

## 6. OPERATIONAL WORKFLOW (STEP-BY-STEP)

```text
[Host CPU]                     [Shared BRAM]                  [DMA Controller]             [FORS IP Core]
    |                                |                               |                           |
    | 1. Write Context & Inputs      |                               |                           |
    |------------------------------->|                               |                           |
    |                                |                               |                           |
    | 2. Program CSRs & START        |                               |                           |
    |--------------------------------------------------------------->|                           |
    |                                |                               | 3. Read Context Burst     |
    |                                |<------------------------------|                           |
    |                                |                               |                           |
    |                                |                               | 4. Stream din data        |
    |                                |                               |-------------------------->|
    |                                |                               |                           |
    |                                |                               |                           | 5. Tree Hashing
    |                                |                               |                           |    (Compute)
    |                                |                               |                           |
    |                                |                               | 6. Stream dout data       |
    |                                |                               |<--------------------------|
    |                                | 7. Write Result Burst         |                           |
    |                                |<------------------------------|                           |
    |                                |                               |                           |
    | 8. Receive IRQ (or poll DONE)  |                               |                           |
    |<===============================================================|                           |
    |                                |                               |                           |
    | 9. Read Signature / PK         |                               |                           |
    |<-------------------------------|                               |                           |
```

1. **Staging:** CPU writes the 80-byte Context block into BRAM at `0x0000` (and signature at `0x0080` if verifying).
2. **Configuration:** CPU configures DMA registers (`SRC_ADDR = 0x0000`, `DST_ADDR = 0x0080`, `CONFIG = 0x01` for Sign), enables interrupts (`IRQ_ENABLE = 0x01`), and writes `CONTROL = 0x01` (`START`).
3. **Ingress Streaming:** DMA reads Context from BRAM Port B and streams it to `din`. DMA asserts `din_last` on the final beat.
4. **Autonomous Computation:** FORS IP evaluates all $k$ trees. In Sign mode, it streams revealed leaf keys and authentication siblings onto `dout`.
5. **Egress Streaming:** DMA receives `dout` stream beats and writes them sequentially into BRAM at `DST_ADDR`.
6. **Completion:** Upon detecting `dout_last`, DMA latches `EXEC_CYCLES`, sets `STATUS.DONE = 1`, and asserts the `irq` interrupt line.
7. **Result Retrieval:** CPU receives interrupt, clears `IRQ_STATUS`, and reads the completed 3,696-byte signature from BRAM at `0x0080`.

---

## 7. SYSTEM INTEGRATION VERIFICATION

To verify correct SoC operation before physical board bring-up, the testbench must validate:
1. **End-to-End KAT Test:** CPU stages NIST FIPS 205 test vectors into BRAM, triggers DMA, and checks that the generated BRAM signature matches the golden C model byte-for-byte.
2. **Sign-then-Verify Loopback:** CPU executes `FORS_SIGN` to produce a signature at `0x0080`, then immediately points DMA to `0x0080` and executes `FORS_PK_FROM_SIG`. Verifies that the candidate public key generated at `0x0EF0` matches the original public key.
3. **Port Contention Safety:** CPU continuously accesses Port A while DMA is bursting on Port B to confirm that dual-port arbitration introduces zero data corruption.
4. **Soft Reset Recovery:** Asserting `CONTROL.SOFT_RESET` mid-execution must abort active bursts, clear DMA FIFOs, and return both DMA and FORS IP to idle within 2 clock cycles.
