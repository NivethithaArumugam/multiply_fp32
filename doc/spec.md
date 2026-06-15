# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by an FSM counter.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (active high, clears busy and counters) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when `busy==0` |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

**Internal state required:**
- `busy` (reg): Indicates operation in progress. Set to 1 when accepting a valid pulse. Set to 0 when stage 7 completes.
- `counter` (reg): Tracks current pipeline stage (1-7). Increments each clock when busy.

### Handshake contract
1. **Idle state** (`busy == 0`):
   - `out_valid` is 0
   - On rising clock edge where `valid == 1`:
     - Capture `a` → internal register `a_r`
     - Capture `b` → internal register `b_r`
     - Set `busy = 1`
     - Set `counter = 1` (start stage 1)

2. **Active state** (`busy == 1`):
   - Ignore new `valid` pulses (do not overwrite `a_r` / `b_r`)
   - Each clock cycle: increment `counter`
   - Execute current stage logic based on `counter` value

3. **Completion** (when `counter == 7` at end of current cycle):
   - Update output `z` with the result
   - Assert `out_valid = 1` for exactly 1 clock cycle
   - Clear `busy = 0`
   - On the next clock edge, `counter` resets implicitly (since `busy==0`)

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- Counted from the clock edge where `valid` is sampled (when `busy==0`) to the clock edge where `out_valid` asserts.
- Example: If `valid` asserts on clock cycle 0, `out_valid` asserts on clock cycle 7.

### Throughput
- **Single-issue (not pipelined)**.
- New operations cannot start until previous operation completes.
- Max throughput is **1 result per 7 cycles** minimum.

---

## Internal Data Model (IEEE-754 binary32)

### FP32 bit layout
For any 32-bit IEEE-754 number:
```
Bit 31:      sign
Bits 30-23:  exponent (biased by 127)
Bits 22-0:   mantissa (fraction)
```

### Interpretation rules
- **Normal numbers**: exponent ∈ [1..254], value = (-1)^sign × 1.mantissa × 2^(exponent - 127)
- **Denormal numbers**: exponent = 0, value = (-1)^sign × 0.mantissa × 2^(-126) [NOT REQUIRED for this task]
- **Zero**: exponent = 0, mantissa = 0
- **Infinity**: exponent = 255, mantissa = 0
- **NaN**: exponent = 255, mantissa ≠ 0

### Required internal signals
- `a_r`, `b_r` (32 bits each): Captured operands
- `a_s`, `b_s` (1 bit each): Sign bits extracted from `a_r[31]`, `b_r[31]`
- `a_e`, `b_e` (signed, 10 bits each): Unbiased exponent = `biased_exp - 127`
- `a_m`, `b_m` (24 bits each): Mantissa with implicit leading 1 prepended
  - If exponent ≠ 0: `a_m = {1'b1, a_r[22:0]}`
  - If exponent = 0 (subnormal): `a_m = {1'b0, a_r[22:0]}`
- `z_s` (1 bit): Result sign = `a_s ^ b_s`
- `z_e` (signed, 10 bits): Result unbiased exponent
- `z_m` (24 bits): Result mantissa before packing
- `product` (48 bits): Intermediate product = `a_m * b_m`
- Rounding support: `guard_bit`, `round_bit`, `sticky_bit`

---

## FSM / Pipeline Stages

The FSM is controlled by a single always block (always @(posedge clk)):
- On reset (`rst == 1`): Clear `busy`, clear `counter`
- On clock edge:
  - If `busy == 0` and `valid == 1`:
    - Capture `a` and `b`
    - Set `busy = 1`
    - Set `counter = 1`
  - Else if `busy == 1`:
    - Execute `case(counter)` to perform stage-specific logic
    - Increment `counter`
    - If `counter == 7` at the START of this cycle:
      - Update `z` with final result
      - Set `out_valid = 1`
      - Clear `busy = 0`
    - Else:
      - Set `out_valid = 0`

**Example timing:**
```
Cycle 0: valid=1, busy=0 → capture a,b → busy=1, counter=1, out_valid=0
Cycle 1: busy=1, counter=1 → stage1 logic → counter=2, out_valid=0
Cycle 2: busy=1, counter=2 → stage2 logic → counter=3, out_valid=0
...
Cycle 6: busy=1, counter=6 → stage6 logic → counter=7, out_valid=0
Cycle 7: busy=1, counter=7 → stage7 logic → update z, busy=0, out_valid=1
Cycle 8: busy=0 → out_valid=0 (ready for next valid)
```

### Stage 1 — Unpack and Special Case Detection
**Inputs:** `a_r`, `b_r` (captured from ports)

**Actions:**
1. Extract fields:
   - `a_s = a_r[31]`, `b_s = b_r[31]`
   - `a_biased_exp = a_r[30:23]`, `b_biased_exp = b_r[30:23]`
   - `a_frac = a_r[22:0]`, `b_frac = b_r[22:0]`

2. Compute unbiased exponents:
   - `a_e = {a_biased_exp} - 127` (store as signed 10-bit)
   - `b_e = {b_biased_exp} - 127` (store as signed 10-bit)

3. Detect special cases:
   - `a_is_zero = (a_biased_exp == 0) && (a_frac == 0)`
   - `a_is_inf = (a_biased_exp == 255) && (a_frac == 0)`
   - `a_is_nan = (a_biased_exp == 255) && (a_frac != 0)`
   - (Same for `b_*`)

4. Set implicit leading bit (insert hidden 1 for normal numbers):
   - If `a_biased_exp != 0`: `a_m = {1'b1, a_frac}` (24 bits)
   - Else: `a_m = {1'b0, a_frac}` (subnormal, if needed)
   - (Same for `b_m`)

**Note:** For this task, assume inputs are always normal (exponent ∈ [1..254]). Special case logic may be simplified or omitted if not critical.

### Stage 2 — Input Validation (Optional Simplification)
**Optional:** If inputs are guaranteed normal, this stage can be a no-op.

**Full logic (if needed):**
- Check if either operand is zero, infinity, or NaN.
- Set `result_is_zero`, `result_is_inf`, `result_is_nan` flags.
- For zero or infinity, prepare early-exit result.

### Stage 3 — Mantissa Normalization (Pre-multiply)
**Note:** For strictly normal inputs, this is typically a no-op.

**If needed:**
- Ensure `a_m` and `b_m` MSBs are set (they should be for normal numbers).
- If not, shift left and adjust exponent accordingly.

### Stage 4 — Multiply Core
**Inputs:** `a_m`, `b_m`, `a_e`, `b_e`, `a_s`, `b_s`

**Actions:**
1. Compute result sign:
   - `z_s = a_s ^ b_s`

2. Compute result exponent:
   - `z_e_temp = a_e + b_e`
   - After multiply, the product will be 48 bits (two 24-bit numbers).
   - The product will have an implicit leading 1 in bit 47 (since both mantissas start with 1).
   - To align properly: `z_e = z_e_temp + 1` (the +1 accounts for the mantissa normalization after multiply)

3. Compute mantissa product:
   - `product = a_m * b_m` (produces 48-bit result)
   - Then scale for rounding: Consider storing as `product_48bit` or proceeding directly to stage 5.

**Example:**
- `a_m = 24'b1.xxxxx...` (fixed-point 1.23 bits)
- `b_m = 24'b1.xxxxx...` (fixed-point 1.23 bits)
- `a_m * b_m ≈ 10.46 bits` (48 bits total, with leading 1 in bit 47)

### Stage 5 — Extract Mantissa and Rounding Bits
**Inputs:** `product` (from stage 4), `z_e` (exponent from stage 4)

**Actions:**
1. Extract the normalized mantissa and rounding support bits:
   - `z_m = product[47:24]` (24-bit mantissa, including implicit 1)
   - `guard_bit = product[23]`
   - `round_bit = product[22]`
   - `sticky = |product[21:0]` (logical OR of all lower bits)

2. Prepare for rounding in stage 6.

### Stage 6 — Normalize and Round-to-Nearest-Even (RNE)
**Inputs:** `z_m`, `z_e`, `guard_bit`, `round_bit`, `sticky_bit`

**Actions:**
1. **Normalization** (ensure mantissa MSB = 1):
   - If `z_m[23] == 0`:
     - Shift `z_m` left until MSB = 1
     - Decrement `z_e` for each shift
   - If `z_m[23] == 1`: No shift needed (already normalized)

2. **Round-to-Nearest-Even (RNE)**:
   - Compute rounding decision:
     - `round_up = 0`
     - If `guard_bit == 1`:
       - If `round_bit == 1` OR `sticky == 1`: `round_up = 1` (round away)
       - Else: `round_up = z_m[0]` (round to even: round up if LSB is 1)
     - Else: `round_up = 0` (round down)
   
   - If `round_up == 1`:
     - `z_m = z_m + 1`
     - If `z_m == 24'b1_0000...0` (overflow):
       - `z_e = z_e + 1`
       - `z_m` resets implicitly (the leading 1 moves to the exponent)

3. Store `z_m` and `z_e` for final packing in stage 7.

### Stage 7 — Pack Result
**Inputs:** `z_s`, `z_e`, `z_m`, `z_is_zero`, `z_is_inf`, `z_is_nan` (if computed)

**Actions:**
1. Determine final exponent and mantissa:
   - If overflow (`z_e > 127`):
     - Result is **infinity**
     - Set exponent field to 255, mantissa field to 0
   - Else if underflow/denorm boundary (`z_e < -126`):
     - Result is **zero** (forced denorm/underflow)
     - Set exponent field to 0, mantissa field to 0
   - Else (normal result, `-126 ≤ z_e ≤ 127`):
     - Biased exponent = `z_e + 127`
     - Mantissa field = `z_m[22:0]` (remove implicit 1)

2. Pack into 32-bit output:
   - `z = {z_s, exponent_field[7:0], mantissa_field[22:0]}`

3. Set control signals:
   - `out_valid = 1` (for 1 cycle)
   - `busy = 0`

---

## Assumptions & Constraints
- **Input operands**: Guaranteed to be normal FP32 numbers with exponent ∈ [1..254].
- **No special cases required** (zero, infinity, NaN) unless you choose to implement them.
- **Round-to-Nearest-Even** is the required rounding mode.
- Design must be synthesizable (no behavioral-only constructs).

---

## Verification Notes
Recommended testbench behavior:
- Drive `valid`, `a`, `b` **synchronously** on rising clock edges.
- Wait for `out_valid` to pulse before sampling `z`.
- Verify latency: `out_valid` should assert exactly 7 cycles after `valid`.
- Generate test vectors covering:
  - Small and large normal numbers
  - Numbers with different signs
  - Products that require rounding
  - Edge cases: results near overflow/underflow boundaries

---
