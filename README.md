# regpressure

### RISC-V Breakout — a register file game

**Live:** [regpressure.riscvthai.org](https://regpressure.riscvthai.org)  
**Part of:** [riscvthai.org](https://riscvthai.org) — open RISC-V curriculum for universities

\---

## What is register pressure?

In any architecture, **register pressure** is the condition where a program
needs more simultaneously live values than the register file has slots to hold.

In RISC-V, you have 32 general-purpose registers (`x0`–`x31`), 32
floating-point registers (`f0`–`f31`), and up to 32 vector registers
(`v0`–`v31`). That sounds generous — until you account for the ABI
reservations. `x0` is wired to zero. `ra`, `sp`, `gp`, `tp` are spoken for.
The calling convention partitions the rest into caller-saved and callee-saved
sets, which constrains the compiler's register allocator across every function
call boundary.

When the allocator runs out of room, it *spills* — writes a live value to the
stack, frees the register, reloads it later. Spills cost instructions, cost
memory bandwidth, cost time. Minimizing register pressure is one of the central
jobs of an optimizing compiler's backend.

The name cuts both ways:

* **reg** — the register file, the central noun of any ISA
* **pressure** — the short-e assonance that makes the compound stick, and the
architectural reality it names

Most RISC-V teaching material focuses heavily on instruction mnemonics — the
*verbs* of the ISA. This game focuses almost entirely on the *nouns*: the
register file itself, its ABI groupings, its privilege layers, its encoding
constraints. That emphasis is deliberate and, we think, underserved.

\---

## The game

A brick-breaking game where every brick is a RISC-V register.

**The ball** is a `LR/x1` instruction token — the link register that makes
subroutine calls return. It alternates with `t1/x6` (a tail-call register)
as you hit vector register bricks, toggling between call and tail-call modes.

**The paddle** is the `PC` — the program counter, with pipeline-stage tick
marks. Move it with arrow keys, `H`/`L` (vi-style), or mouse. Spin it by
moving fast just before contact; lateral velocity bleeds into the ball's
trajectory, just as a mispredicted branch affects the pipeline.

**`x0` / zero** is always unbreakable. It always returns zero. No matter what
you throw at it.

**Space** (mid-game) toggles between **RV32I** (full-width paddle) and **RVC**
(compressed, half-width paddle). In RVC mode, only registers reachable via
C-extension 3-bit encodings — `x8`–`x15` and a few others — can be broken.
The rest bounce the ball back. This is not a metaphor; it is the actual
constraint of the C extension's register addressing.

\---

## Level progression

Each level is cleared by breaking every breakable brick. Lives carry forward
across levels. You start with five.

|Level|Name|Scramble|Register pool|Ball behavior|
|-|-|-|-|-|
|1|The Textbook|None — canonical order|GPR + FPR + VR|Uniform speed|
|2|Shuffled Deck|Within type|GPR + FPR + VR|Typed (GPR speeds up, FPR drifts)|
|3|Register Pressure|Full cross-type|GPR + FPR + VR|Typed + ghost trail on VR|
|4|Privileged|Full + CSR bands|+ CSR M-mode + CSR S-mode|+ paddle flicker on CSR hit|
|5|The Debugger Is Watching|Full|+ Debug regs + Vector CSRs|+ debug path reveal + VCSR diagonal reflect|
|6|Advanced|Full|+ Performance counters|+ fake score display on PERF hit|
|7|PMP — good luck|Full|+ PMP address + PMP config|+ multi-hit pmpcfg, TOR chaining, lock bits, PMP stall|

### Level 1 — The Textbook

Registers appear exactly where Waterman's diagram puts them: VR on top, FPR in
the middle, GPR on the bottom, `x0`/zero at position \[0] in the GPR row. No
surprises. The ball moves at uniform speed regardless of what it hits. RVC
paddle toggle is not yet available. A good place to learn the ABI color coding.

### Level 2 — Shuffled Deck

Each register type shuffles internally. GPR bricks are somewhere in the GPR
band; FPR bricks somewhere in the FPR band; VR somewhere in the VR band — but
not where you expect them. `x0`/zero is hiding. Brick widths become natural
(proportional to label length: `mvendorid` is wider than `v1`). Typed ball
behavior begins: GPR hits accelerate the ball; FPR hits add lateral drift
(floating-point imprecision, literally); VR hits toggle call/tail mode and
leave a ghost trail. RVC paddle toggle unlocks.

### Level 3 — Register Pressure

The full cross-type scramble. GPR, FPR, and VR bricks are all shuffled into a
single pool and packed into rows. `zero` is somewhere. Good luck. The player
is now functioning as a register allocator under time pressure — which is
precisely what the name means. As any register type thins below 8 surviving
bricks, survivors pulse with a faint warning glow.

### Level 4 — Privileged

M-mode and S-mode CSR bands appear above the scrambled GPR/FPR/VR pool,
initially in canonical order (ordered CSRs are new; learn them here). CSR hits
cause the paddle to flicker — "privilege level negotiation." `satp` is
slightly wider and a different shade. It is load-bearing in real systems too.

### Level 5 — The Debugger Is Watching

A static debug register band (`dcsr`, `dpc`, `tselect`, `tdata1/2/3`,
`dscratch0/1`) sits at the very top of the field, watching. Breaking a debug
brick reveals the ball's predicted trajectory for three seconds — the debugger
has been triggered and briefly exposes state. Vector CSRs (`vl`, `vtype`,
`vlenb`, `vstart`, `vxsat`, `vxrm`, `vcsr`) are injected into the scrambled
pool. Hitting `vl` or `vtype` swaps `vx` and `vy` — the shape of the
computation changes, because architecturally it does.

### Level 6 — Advanced

Performance counter registers (`mcycle`, `minstret`, `mhpmcounter3–5`,
`time`, `cycle`, `instret`) are scattered through the pool. Hitting one causes
the displayed score to show a wrong value for two seconds. Performance counters
lie; this is known. The ghost trail becomes permanent (always on, faint). The
debug band still watches from the top.

### Level 7 — PMP — good luck

Physical Memory Protection registers enter the field.

* **`pmpaddr0`–`pmpaddr15`**: brick width encodes the address-matching mode.
NAPOT (naturally aligned power-of-two) = wide brick. NA4 = narrow. TOR
(top of range) = medium. OFF = ghost brick, ball passes through.
* **`pmpcfg0`–`pmpcfg3`**: multi-hit bricks requiring 8 contacts each, one per
embedded configuration byte. Each hit reveals a byte field label: `L·A·A·X·W·R`.
* **TOR chaining**: a TOR-mode `pmpaddr\[i]` cannot be broken until
`pmpaddr\[i-1]` is already cleared — the range bottom must go first.
* **Lock bit**: \~30% of pmpcfg bytes are locked. A locked byte cannot be
cleared until its paired pmpaddr is gone.
* **RV64 ghost bricks**: `pmpcfg1` and `pmpcfg3` are rendered as crossed-out
ghost bricks. The ball passes through them. Flash message: "RV64: odd pmpcfg
undefined." This is not a game mechanic. This is the specification.
* **PMP stall**: hitting any PMP brick briefly slows the ball to 60% speed for
\~40 frames before snapping back. Memory access latency, simulated.

Leaderboard entries for Level 7 completions carry a small lock glyph. No
explanation is provided in the UI.

\---

## Scoring and bonus multiplier

Base points per register type:

|Type|Points|
|-|-|
|GPR|10|
|FPR|20|
|VR|30|
|CSR (M/S)|40|
|VCSR / Debug|50–60|
|PERF|45|
|PMP addr|80|
|PMP cfg|10 per hit (80 total)|

A **bonus multiplier** (×1–×6) increases when VR or VCSR bricks are broken
without having first touched any GPR brick. Touch GPR first and the multiplier
locks at ×1 for the remainder of the level. This encodes the real preference:
keep hot data in upper register files, avoid GPR spills.

\---

## Earning lives back

Lives are finite across the entire run (not per level). You start with five.
Hard cap: seven. Earn-backs:

|Trigger|Reward|Available from|
|-|-|-|
|Clear an entire register type without dropping the ball|+1 life|Level 1|
|Break the first brick of a level without wall contact first (clean launch)|+1 life|Levels 1–2|
|Break 5 consecutive RVC-accessible bricks in RVC mode without a miss|+1 life|Level 2|
|Ball contacts `x0`/zero 3 times in one level without dying|+1 life|Level 2|
|Break a locked `pmpcfg` byte after properly clearing its `pmpaddr` first|+1 life|Level 7|

Lives are displayed as a row of register-chip icons labeled `ra`→`t2` (the
first seven caller/callee saved registers). When you lose one it grays out;
when you earn one back it lights up briefly. You are managing a calling
convention stack of survival tokens.

\---

## The clock

A single continuous clock runs across all levels from first launch to final
brick or death. It pauses on inter-level screens (thinking time is not play
time). Per-level split times are recorded internally and shown on the
inter-level pause screen and in the name-entry summary at game end.
Leaderboard tiebreaking uses elapsed seconds, then milliseconds.

\---

## ABI color coding

|Color|Group|Registers|
|-|-|-|
|Gray|zero|x0 (unbreakable)|
|Blue|ra/sp/gp/tp|x1–x4|
|Green|temp|t0–t2, t3–t6|
|Amber|saved|s0–s1, s2–s11|
|Red|argument|a0–a7|
|Teal (dark)|FPR temp|ft0–ft7, ft8–ft11|
|Teal (mid)|FPR saved|fs0–fs1, fs2–fs11|
|Blue (mid)|FPR argument|fa0–fa7|
|Purple|Vector|v0–v31|
|Magenta|CSR M-mode|mstatus, misa, mtvec…|
|Indigo|CSR S-mode|sstatus, stvec, satp…|
|Dark gray|Debug|dcsr, dpc, tselect…|
|Dark indigo|Vector CSR|vl, vtype, vlenb…|
|Dark orange|Perf counters|mcycle, minstret…|
|Dark stone|PMP|pmpaddr0–15, pmpcfg0–3|

In RVC mode, RVC-accessible bricks gain a gold border highlight.
Non-accessible bricks dim to 35% opacity.

\---

## Controls

|Input|Action|
|-|-|
|← → arrow keys|Move paddle|
|H / L|Move paddle (vi-style)|
|Mouse|Aim paddle|
|Space (before launch)|Launch ball|
|Space (mid-game)|Toggle RVC / RV32I paddle width|
|Space (game over / win)|Restart|

\---

## Leaderboard fields

|Field|Description|
|-|-|
|name|Up to 16 characters, sanitized|
|score|Total score across all levels cleared|
|level|Highest level cleared (not just reached)|
|lives\_left|Lives remaining at game end|
|elapsed|Total wall-clock seconds|
|isa|RV32I or RVC (mode at game end)|
|perfect ★|Never touched GPR before VR/CSR|
|rvc\_used|RVC paddle was toggled at least once|
|pmp\_lock 🔒|Level 7 cleared|
|splits|Per-level elapsed seconds|

\---

## Technical

* Pure client-side HTML/CSS/JavaScript — no framework, no build step
* Web Audio API for synthesized sound (square wave, pitch-mapped by register type)
* Leaderboard fallback chain: CGI server → `window.storage` (Claude artifact) → `localStorage`
* CGI backend: two plain C programs, flat pipe-delimited text file, `flock` for
concurrent write safety, per-IP rate limiting via filesystem timestamps
* Server: FreeBSD / Apache 2.4 — same infrastructure as
[games.riscvthai.org](https://games.riscvthai.org)

\---

## Origin and context

This game grew out of a persistent observation made after the
[JCSSE 2023](https://jcsse2023.org) RISC-V hands-on workshop: almost all
RISC-V teaching material, including interactive tools and visualizers, focuses
on instruction mnemonics — the *verbs* of the ISA. The register file — the
*nouns*, the named storage that instructions operate on — is treated as
background furniture.

A student who can recite every instruction format may still be unable to
identify which registers are caller-saved, which are argument registers, why
`x0` always reads zero, or what "register pressure" means to a compiler. This
game is a small attempt to correct that imbalance by making the register file
itself the subject of play, not the backdrop.

The pedagogical ladder:

1. **Level 1** teaches register names and ABI groupings through canonical layout
2. **Level 2** tests whether you've internalized the groupings well enough to
find them when shuffled
3. **Level 3** simulates register pressure — the allocator's problem, made physical
4. **Levels 4–5** introduce the privilege architecture and debug infrastructure
that operating systems and debuggers depend on
5. **Levels 6–7** surface the performance monitoring and physical memory
protection layers that matter for systems programmers and security researchers

The RVC paddle toggle at Level 2 teaches a real constraint: the C extension
can only encode 3-bit register fields, so compressed instructions can only
directly address `x8`–`x15`. This is not trivia; it affects code generation
for embedded targets.

The PMP mechanics at Level 7 are a compressed interactive reference for the
most confusing part of the RISC-V privileged specification: the interaction
between address-matching modes (NAPOT, NA4, TOR, OFF), the lock bit, the
7-bit configuration byte encoding, and the RV64 oddity where `pmpcfg1` and
`pmpcfg3` do not exist. If you can clear Level 7, you understand PMP.

\---

## Files

```
regpressure/
├── index.html              — the game (standalone, no dependencies)
├── rvbreakout\_board.c      — GET leaderboard CGI
├── rvbreakout\_score.c      — POST score submission CGI
├── DEPLOY.txt              — server setup and Apache VirtualHost config
└── README.md               — this file
```

Build the CGI pair on the server:

```sh
cc -O2 -o rvbreakout\_board.cgi rvbreakout\_board.c
cc -O2 -o rvbreakout\_score.cgi rvbreakout\_score.c
```

\---

## Related

* [riscvthai.org](https://riscvthai.org) — Thai RISC-V curriculum
* [games.riscvthai.org](https://games.riscvthai.org) — other RISC-V educational games
* [RISC-V International Academia \& Training SIG](https://riscv.org)
* *The RISC-V Reader* — Patterson \& Waterman (Thai edition in preparation)

\---

*RISC-V is a trademark of RISC-V International. This project is independent
open educational material and is not affiliated with or endorsed by RISC-V
International.*

