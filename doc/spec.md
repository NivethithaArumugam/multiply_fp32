# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a multi-cycle single-precision floating-point multiplier processing one operation at a time using a valid/out_valid handshake. It features a fixed 7-cycle latency tracking from `counter = 1` to `counter = 7`.

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`       | in  | 1     | Clock |
| `rst`       | in  | 1     | Async active-high reset (`posedge rst`) |
| `valid`     | in  | 1     | 1-cycle start pulse. Ignored if design is busy. |
| `a`         | in  | 32    | Operand A (FP32 bits) |
| `b`         | in  | 32    | Operand B (FP32 bits) |
| `z`         | out | 32    | Result (FP32 bits). Stays stable until next out_valid. |
| `out_valid` | out | 1     | 1-cycle pulse active exactly when `z` updates. |

### Handshake Contract
- When `busy == 0`, a high `valid` on a rising clock edge registers operands `a` and `b` into internal registers `a_r` and `b_r`, sets `busy <= 1`, and initializes `counter <= 1`.
- While `busy == 1`, new `valid` pulses are strictly ignored.
- At `counter == 7`, the output `z` is written, `out_valid <= 1` is pulsed for one cycle, and `busy <= 0` is cleared on the following edge.

---

## Latency and Internal Registers
To prevent timing accumulation across pipeline stages, all intermediate variables listed below must be implemented as sequential registers updated non-blockingly (`<=`) inside the FSM case statement.

### Internal Registry Map
- `a_s, b_s, z_s`: 1-bit sign registers.
- `a_e, b_e, z_e`: 10-bit signed exponent registers (must use `reg signed [9:0]` to prevent unsigned wrap during bias math).
- `a_m, b_m`: 24-bit unsigned mantissa registers (includes the explicit leading 1).
- `z_m`: 24-bit unsigned destination mantissa register.
- `product`: 48-bit unsigned product register.
- `guard_bit, round_bit, sticky`: 1-bit rounding registers.

---

## FSM / Pipeline Stages (1 to 7)
The control logic must be driven by a sequential block `always @(posedge clk or posedge rst)` evaluating `case(counter)`. 

### Stage 1 — Unpack Operands
- Capture signs: `a_s <= a_r[31]; b_s <= b_r[31];`
- Unbias exponents: `a_e <= $signed({2'b0, a_r[30:23]}) - 10'd127; b_e <= $signed({2'b0, b_r[30:23]}) - 10'd127;`
- Extract mantissas with explicit hidden bit: `a_m <= {1'b1, a_r[22:0]}; b_m <= {1'b1, b_r[22:0]};`

### Stage 2 — Pre-Calculation Setup
- Compute final sign: `z_s <= a_s ^ b_s;`
- Add unbiased exponents: `z_e <= a_e + b_e;`
- *Note: Since inputs are strictly normal numbers, skip all NaN, Infinity, and subnormal decoding logic.*

### Stage 3 — Idle Pipeline Delay
- No operation. Let intermediate expressions settle. (Increments `counter`).

### Stage 4 — Mantissa Multiplication
- Perform the $24 \times 24$ bit unsigned multiplication: `product <= a_m * b_m;`

### Stage 5 — Extract Mantissa and Rounding Bits
Analyze the 48-bit `product` result to extract the raw 24-bit mantissa (`z_m`) and rounding bits. 
- **Case A: Product Overflowed (`product[47] == 1`):**
  - `z_m <= product[47:24];`
  - `guard_bit <= product[23];`
  - `round_bit <= product[22];`
  - `sticky <= (product[21:0] != 0);`
  - `z_e <= z_e + 10'd1; // Increment exponent due to product bit shift`
- **Case B: No Product Overflow (`product[47] == 0`):**
  - `z_m <= product[46:23];`
  - `guard_bit <= product[22];`
  - `round_bit <= product[21];`
  - `sticky <= (product[20:0] != 0);`

### Stage 6 — Round-to-Nearest-Even (RNE)
Evaluate the rounding bits extracted in Stage 5:
- Condition for rounding up: `if (guard_bit && (round_bit || sticky || z_m[0]))`
  - If true: `z_m <= z_m + 1;`
  - If `z_m + 1` overflows 24 bits (i.e., `z_m == 24'hFFFFFF`), handle the round-overflow by setting `z_m <= 24'h800000;` and incrementing the exponent `z_e <= z_e + 10'd1;`

### Stage 7 — Re-bias and Pack Output
- Convert unbiased exponent back to standard bias: `wire [9:0] final_biased_exp = z_e + 10'd127;`
- Construct final result:
  - If `final_biased_exp >= 255`, force overflow value: `z <= {z_s, 8'hFF, 23'b0};` (Infinity)
  - Else: `z <= {z_s, final_biased_exp[7:0], z_m[22:0]};`
- Assert `out_valid <= 1;` and clear `busy <= 0;`
---

## Implementation Guidelines & Verification

### Pipeline Stability
- **Stage 3 Constraint**: This stage must be implemented as a distinct FSM state (`counter == 3`). It must explicitly hold the current data registers for exactly one clock cycle before allowing the pipeline to proceed to Stage 4. Do not optimize this state away or combine it with other stages.

### Sanity Check (Agent Self-Verification)
Before completing the implementation, the agent should verify the following edge cases mentally or via testbench to ensure IEEE-754 correctness:
- **1.0 * 1.0**: Expected Result = `0x3F800000` (Sign: 0, Exp: 127, Mantissa: 0).
- **2.0 * 2.0**: Expected Result = `0x40800000` (Sign: 0, Exp: 129, Mantissa: 0).
- **Mantissa Overflow Logic**: Ensure that if rounding causes the mantissa to exceed `24'hFFFFFF`, the mantissa wraps to `24'h000000` and the exponent increments correctly.