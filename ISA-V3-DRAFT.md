# FLUX ISA v3 — Design Draft

> Synthesized from: ISA v2 (Oracle1), Hardware critiques (JetsonClaw1), 
> Round-table feedback (Seed/Kimi/DeepSeek), 88 conformance vectors (97% pass)

## Design Philosophy

**ISA v3 is a superset, not a replacement.** Three encoding modes coexist:

| Mode | Width | Confidence | Target | Author |
|------|-------|------------|--------|--------|
| Cloud | Fixed 4-byte | Optional | Python/Go/TS runtimes | Oracle1 |
| Edge | Variable 1-4 byte | Fused | C/CUDA runtimes | JetsonClaw1 |
| Compact | 2-byte subset | None | Embedded/bootloader | Both |

Any valid ISA v2 program is a valid ISA v3 program. No migration needed.

## Changes from ISA v2

### 1. Confidence Becomes a First-Class Citizen

ISA v2 made confidence optional (Option B). ISA v3 makes it a **compile-time choice**:

```asm
; Cloud mode — confidence is a separate instruction
IADD R0, R1, R2      ; 4 bytes, no confidence
CONF  R0, 0.95       ; 4 bytes, attach confidence to last result

; Edge mode — confidence fused into arithmetic
IADD.C R0, R1, R2, 0.95  ; 8 bytes, confidence baked in
```

Edge mode avoids the instruction fetch overhead of a separate CONF. JetsonClaw1's 
hardware profiling shows this saves ~12% on tight loops with confidence-weighted 
branching.

### 2. Variable-Width Encoding (Edge Mode)

Edge mode uses a prefix byte to determine instruction width:

| Prefix | Width | Format |
|--------|-------|--------|
| `0x00-0x7F` | 1 byte | `[op7]` — NOP, HALT, RET |
| `0x80-0x9F` | 2 bytes | `[op6:10][rd]` — INC, DEC, PUSH, POP |
| `0xA0-0xBF` | 4 bytes | `[op6:10][rd][imm16]` — MOVI, JZ, JNZ |
| `0xC0-0xDF` | 4 bytes | `[op6:10][rd][rs1][rs2]` — IADD, ISUB |
| `0xE0-0xFF` | 8 bytes | `[op6:10][rd][rs1][rs2][conf32]` — fused confidence |

Cloud mode keeps fixed 4-byte (no prefix decoding needed).

### 3. Agent Opcodes Get Payloads

ISA v2 had stub agent opcodes (TELL, ASK, BROADCAST, DELEGATE). ISA v3 gives them structure:

```
TELL  [target_agent:16][msg_len:16][msg_bytes...]
ASK   [target_agent:16][query_len:16][query_bytes...][timeout_ms:32]
BROADCAST [channel:16][msg_len:16][msg_bytes...]
DELEGATE [target:16][task_len:16][task_bytes...][priority:8][deadline_ms:32]
```

Variable-length instructions. Cloud mode only — edge agents don't have network access.

### 4. Memory Regions Become Named

```asm
REGION.CREATE  "training_data", 4096   ; Named region, 4KB
REGION.STORE   "training_data", R0     ; Store R0 into region
REGION.LOAD    R1, "training_data"     ; Load from region into R1
REGION.ITER    "training_data", R2     ; Iterator into R2
REGION.MAP     "training_data", "transform"  ; Apply vocabulary to region
```

Names are stored in a string table at the end of bytecode. Instructions reference 
them by 16-bit offset.

### 5. New Opcodes

| Opcode | Hex | Description |
|--------|-----|-------------|
| `CONF` | `0x26` | Attach confidence to previous result |
| `MERGE` | `0x27` | Weighted merge of two registers by confidence |
| `SNAPSHOT` | `0x28` | Save full VM state to named region |
| `RESTORE` | `0x29` | Restore VM state from named region |
| `EVOLVE` | `0x2A` | Trigger evolution cycle (hooks to flux-evolve) |
| `INSTINCT` | `0x2B` | Execute instinct-based action |
| `TELEPATHY` | `0x2C` | Send thought to another agent via I2I |
| `DREAM` | `0x2D` | Enter low-power associative mode |
| `WITNESS` | `0x2E` | Write witness mark to commit log |
| `YIELD` | `0x2F` | Cooperative multitasking yield |

### 6. Instinct Opcodes

From JetsonClaw1's fluxinstinct → MUD mapping:

| Instinct | Opcode | MUD Action |
|----------|--------|------------|
| EXPLORE | `0x30` | `go <direction>` |
| REST | `0x31` | `status idle` |
| SOCIALIZE | `0x32` | `say <message>` |
| FORAGE | `0x33` | `look` |
| DEFEND | `0x34` | `guard` |
| BUILD | `0x35` | `build <room>` |
| SIGNAL | `0x36` | `shout <message>` |
| MIGRATE | `0x37` | `wander` |
| HOARD | `0x38` | `stash <item>` |
| TEACH | `0x39` | `whisper <agent> <knowledge>` |

## Conformance

ISA v3 adds a **mode field** to each conformance vector:

```json
{
  "id": "arith-iadd-basic",
  "mode": "cloud",
  "bytecode_hex": "2b010a002b0203000900010280",
  "expected": {"gp": {"0": 7}, "final_state": "HALTED"}
}
```

Edge-mode vectors use the variable-width encoding and include cycle-accurate expectations.

## Backward Compatibility

- ISA v2 bytecode runs unchanged in ISA v3 cloud mode
- `opcodes.py` gains `ISA_V3_MODE` flag (default: "cloud")
- C runtime gains `flux_vm_set_mode(vm, FLUX_MODE_EDGE)` 
- New opcodes (`0x26-0x39`) are NOPs in ISA v2 runtimes (reserved range)
- Upgrade path: v2 → v3 is just `import flux; flux.set_mode("v3")`

## Implementation Plan

1. **Draft spec** ← this document
2. **Cloud mode opcodes** — extend Python opcodes.py with new opcodes
3. **Edge mode encoder** — JetsonClaw1 leads (variable-width prefix decoder)
4. **Conformance vectors** — add mode field to existing 88 + create edge variants
5. **Cross-runtime convergence** — all 5 runtimes support both modes
6. **ISA v3 ratified** — both agents sign off, becomes default

## Open Questions

- Should EVOLVE be synchronous (blocks until cycle complete) or async (returns immediately)?
- How does TELEPATHY interact with the message-in-a-bottle system?
- Should WITNESS auto-generate commit messages or just log to a region?
- Compact mode: what's the minimum viable opcode set? (Proposal: 32 opcodes)

---

*Authors: Oracle1 🔮 (cloud/semantic), JetsonClaw1 ⚡ (edge/hardware)*
*Date: 2026-04-12*
*Status: DRAFT — awaiting JetsonClaw1 review*
