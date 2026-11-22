# 2hbj's Obfuscator

> **Quick answer:** 2hbj's Obfuscator is a free obfuscator for Luau (Roblox's Lua dialect) that does not just scramble your script — it destroys the readable source entirely, compiles it into encrypted custom bytecode, and runs it on an embedded virtual machine with per-file randomized opcodes, multi-layer encryption, HMAC-SHA256 tamper detection, anti-hooking, and anti-analysis traps. Protection is tunable through a documented set of command-line flags, from a near-zero-overhead `--light` mode to full VM plus control-flow flattening. The result is a single, dependency-free `.lua` file. Official Discord: [discord.gg/d3A2uXPnHS](https://discord.gg/d3A2uXPnHS).

Most free obfuscators put a fancier lock on your diary. 2hbj's Obfuscator pours the diary into a bank vault, welds the door shut, wires the vault to fight back — and then hands you the switchboard, because you decide how paranoid to be. Every protected file announces itself with one line: `-- the file is protected by 2hbj`.

## What Is 2hbj's Obfuscator?

2hbj's Obfuscator is a community-made, **free** source protection tool built for **Luau**, the language behind Roblox scripting and most Roblox executor scripts. You hand it an ordinary `.lua` file — through the Discord bot's **#obfuscate** channel or the CLI — and it hands you back one self-contained output file that behaves identically at runtime but no longer contains your script in any readable form. There is nothing to install, no runtime library to ship — the entire decryption machinery travels inside the protected file itself. Since **the 1st generation of 2hbj's obfuscator, v4.5**, Luau-targeted output is the default.

| Quick Facts | |
|---|---|
| Price | Free |
| Target language | Luau / Roblox executor environments |
| Interface | Discord bot (#obfuscat.e channel) + CLI with documented flags |
| Output | One `.lua` file, no external dependencies |
| Protection class | Bytecode virtualization + layered encryption + runtime defenses |
| Distribution | Discord server ([discord.gg/d3A2uXPnHS](https://discord.gg/d3A2uXPnHS)) |

## What Do Users Actually Get From It?

Obfuscation only matters if it solves real problems. Here is what a script author gains the moment they protect a file:

- **Your logic stops being public.** On the default (VM) path, the output contains no readable functions, variables, strings, or structure. What loads is a virtual machine; your code exists only *inside* it.
- **Edits become self-defeating.** The file continuously verifies its own integrity. Change one byte — even a "harmless" one — and the script does not politely complain. It corrupts its own keys and dies messily, leaving nothing useful behind.
- **Code-lifting stops working.** Copy-pasting someone's obfuscated script into an AI assistant or a beautifier produces noise, not your source. That alone kills the most common form of "script stealing" in the Roblox scene.
- **Reuploads lose their value — and get traced.** A leaked file is a locked box, and with the `--watermark` flag every build carries a unique ID you can label per customer, so a leak tells you *who* leaked it.
- **Hooking gets caught.** Tools that swap out standard Roblox/Lua functions to intercept scripts mid-execution are detected and shut down at runtime.
- **It still runs like a normal script.** Despite all the armor, the loader pools memory, caches decoded data, and probes the environment (`getfenv`/`getgenv`, `bit32` availability) so it works across different executor setups.
- **You choose the trade-off.** Speed versus armor is a dial, not a switch — see the flag system below.

## The Protection Stack, Layer by Layer

### 1. Total source destruction (bytecode virtualization)
Readable Lua statements are gone. The obfuscator compiles your script into a **custom bytecode format** executed by an **embedded virtual machine written in Lua**. Function boundaries, control flow, and variable names exist only inside the VM at runtime — a decompiler has nothing Lua-shaped to chew on. (In a hurry and okay with weaker protection? `--light` skips the VM entirely and only hides strings — near-original speed, but beautified code is readable again.)

### 2. Per-file randomization
The VM is not the same twice. Opcode numbering, instruction encoding, internal data layouts, even whether instructions live in tables or buffers are **randomized per file**, so two protections of the same script produce two unrelated machines. Knowledge someone gains cracking file A does not transfer to file B.

### 3. Layered encryption with no skeleton key
The payload passes through **several stacked ciphers**, not one: key-scheduled byte ciphers, keystream schemes, S-box shuffles, chained transformations, and bit-rotation layers. Keys are never stored in the file — they are rebuilt at runtime from **integrity-derived keys** bound to the file's own checksum data (enabled on every build, no flag needed). There is no single lock to pick, because there is no single lock.

### 4. An active tamper defense ("poison pill")
A pure-Lua **SHA-256/HMAC implementation** is embedded in every protected file and used to authenticate the header, the constants, and the bytecode itself — re-checked block-by-block *while the script runs*. On any mismatch, the script **deliberately poisons its own state** before failing: keys degrade, opcode tables drift, behavior breaks in ways that are painful to diagnose. Tampering does not produce a clean error; it produces a wreck.

### 5. Anti-hook validation
The loader captures private references to dozens of standard functions — `string`, `math`, `table`, `bit32`, `rawequal`, `select`, and more — and **re-validates them during execution** against the live global table. If something replaced a builtin to spy on the script, execution fails. This is a defense almost no free obfuscator bothers with.

### 6. Anti-analysis traps
- **Time stalls:** on suspected analysis, the script burns several seconds in a calibrated busy loop before failing — automated tracing becomes slow and miserable.
- **Console traps:** poisoned tables whose `__tostring` metamethod throws, breaking naive `print()` and table-dump inspection.
- **Junk code:** dozens of dead blocks stuffed with random strings, dummy tables, and throwaway arithmetic that mimic real logic — including Luau-only junk shapes unlocked by default.
- **Path diversification:** the VM tracks what it has executed and shapes instruction behavior from that history, so forced jumps *quietly misbehave* later instead of erroring — an analyst who bypasses a check doesn't get a crash, they get a slow, confusing lie.
- **One-shot key rotation:** the master key rotates after the first VM call, and already-verified data is re-checked against the new key.

### 7. Encoded everything
Strings, numbers, error messages — even the *names of standard library functions* — are encoded through a toolbox of custom encoders (Base85 variants with shuffled alphabets, a modified Base64, hex, binary-string encodings). Numbers become arithmetic puzzles mixing decimal, hex (`0x`), and binary (`0b`) notations. Nothing in the file is meant to be read by a human.

## Pick Your Armor: The Flag System

2hbj's Obfuscator ships with a documented set of flags, grouped into families. The defaults are the balanced choice; the extras are for pushing either speed or paranoia to the limits.

### Core modes and presets

| Flag | What it does | The trade-off |
|---|---|---|
| `--luau` *(default on since v2.6)* | Unlocks Luau-only junk shapes and binary literals | Output only parses under Luau — fine for Roblox. Use `--no-luau` on the CLI for stock Lua test suites; the bot always targets Roblox |
| `--light` | Skips the VM completely; hides strings only. Ignores every other VM flag | Near-original speed, but the code is readable again if someone beautifies it |
| `--lean` | Turns off optional cipher layers, drops decoy canaries, spaces out periodic tamper rechecks — keeps the load-time checks | Smaller and faster, less armor. Validate on a real client before shipping with it |
| `--no-tamper` | Strips the anti-tamper harness entirely | Smaller build, faster load — but nothing checks the environment and it can be instrumented or edited freely. Fine for your own testing; don't hand it to someone you don't trust |
| Build attestation *(beta)* | The artifact checks the host it runs in and does **nothing at all** if that host isn't a real Roblox client — no print, no error, nothing to trace | Bigger artifact, slightly slower start. Build and test on a real client before shipping. Overrides `--no-tamper` if both are set |

### Speed and size

| Flag | What it does |
|---|---|
| `--ssa-opt` | Constant folding and dead-code removal up front. No real downside — the script gets smaller and a touch faster |

### Control-flow flattening (CFF)

| Flag | What it does |
|---|---|
| `--cff` | Every if/else/loop edge goes through a **randomized-key dispatcher** instead of a direct jump. Brutal for static analysis; up to ~5x slower on loop-heavy code |
| `--cff-coverage` | Only flattens a random slice of functions. Approximate speed cost: `0.25` ≈ 1.35x, `0.5` ≈ 3.2x, `1.0` up to ~5x. Use `0.25` if `--cff` made your script laggy |
| `--cff-nest` | Runs the flattener over its own dispatcher output a second time. Better protection, occasionally breaks — test the build first. Requires `--cff` |
| `--cff-min-instrs` | Functions shorter than N instructions stay unflattened — the dispatcher isn't worth it on one-liners |
| `--cff-max-instrs` | Functions longer than N instructions stay unflattened — perfect for exempting one huge hot function while flattening the rest |
| `--cff-exclude` | Skip functions by exact source name (`myFunc`, `someTable:method`). Anonymous functions can't be named — use `--cff-max-instrs` for those |
| `--cff-include` | The opposite of exclude: only these functions get flattened. Wins over `--cff-exclude` if you set both |

### VM internals

| Flag | What it does |
|---|---|
| `--vm-sharding` | Dispatch lives in **two separate closures**, so an analyst who finds one doesn't automatically get the other. Roughly 17–49% runtime cost on call-heavy scripts |
| `--no-path-div` | Turns off path diversification (see layer 6). Gives speed back for benchmarking — ship with it on otherwise |
| `--no-buffer` | Forces table-based instruction storage: reproducible builds, and safe on runtimes without Luau's `buffer` library |
| `--use-buffer` | The other storage mode — variety only, don't pick it for speed. Never set both of these |

### Leak tracing

| Flag | What it does |
|---|---|
| `--watermark buyer1` | Records your label next to the build's unique watermark ID in the log — if your script leaks, you know who it leaked from |

### Proven recipes

- **Shipping to strangers:** defaults + `--ssa-opt`, plus one `--watermark` per customer.
- **Maximum paranoia:** defaults + `--cff` + `--cff-nest` + `--vm-sharding` — and test on a real client before release.
- **Loops got laggy:** keep `--cff`, add `--cff-coverage 0.25`, exempt the hot function with `--cff-max-instrs`.
- **Pure speed, light risk:** `--light` + `--ssa-opt`.
- **Your own test builds:** `--lean` or `--no-tamper`.

## How It Helps, By Use Case

- **Script sellers / premium hub developers:** your monetized logic stays yours. Customers get working products; leakers get locked boxes — with a watermark ID pointing back at them.
- **Private or paid-per-user scripts:** anti-tamper and anti-hook mean cracked copies break and hooked environments get detected.
- **Anyone sharing scripts publicly:** obfuscation filters out casual skids instantly. The people who give up first are the people who were never going to respect your work anyway.
- **Performance-sensitive projects:** the flag system lets you flatten the crown-jewel functions while leaving hot loops native — protection where it counts, speed where it matters.

## Why It Beats Most Free Obfuscators (Yes, Including Prometheus)

**Prometheus** is the most popular free Lua obfuscator — a well-built, open-source project offering minification, string and constant encryption, control-flow flattening, opaque predicates, and a register-VM virtualization step. It is genuinely good software. So why does 2hbj's Obfuscator provide stronger protection in practice?

| Dimension | 2hbj's Obfuscator | Prometheus (typical free pipeline) |
|---|---|---|
| Approach | Compile to encrypted bytecode on a custom VM | AST transformations, source-to-source |
| Output shape | No Lua-shaped code left to decompile | Still parseable Lua, just scrambled |
| Opcode/structure randomization | Randomized per file | Documented, consistent design |
| Runtime integrity checks | HMAC-SHA256, self-corrupting | None — output is passive |
| Anti-hook of builtins | Yes, re-checked at runtime | None |
| Anti-analysis traps | Yes (time stalls, console traps, path diversification) | None |
| Tuning granularity | Per-function CFF targeting, storage modes, presets | Pipeline-level steps only |
| Leak tracing | Built-in build watermarking | None |
| Attacker's reference material | Only the protected files themselves | The obfuscator's full source is public |
| Price | Free | Free (open source) |

**The open-source paradox.** Prometheus publishes the blueprint of every lock it sells. Great for transparency, great for the community — and a gift to attackers, who can study each transformation and write automated deobfuscation tooling against a known target. With 2hbj's Obfuscator, the only public artifacts are the protected files themselves. An attacker must reverse-engineer the VM *from the output alone*, with no reference implementation, no documentation, and a different machine waiting in the next file.

**Defense in depth vs. a checklist.** Prometheus protects a script with a series of standalone transforms. 2hbj's Obfuscator interlocks its layers: encryption depends on integrity-derived keys, keys depend on checksums, checksums trigger self-corruption, hook detection watches from the side, and sharding splits the dispatch so no single discovery unravels the machine. Peeling one layer does not expose the next — it usually breaks the file.

**Free without feeling free.** VM virtualization, HMAC-based tamper detection, anti-hooking, anti-analysis measures, per-function flattening control, and leak-tracing watermarks are features associated with paid tools in this space. 2hbj's Obfuscator bundles them at zero cost, which is exactly what makes it unusual in the free tier.

## Honest Limitations

No obfuscation is unbreakable forever, and pretending otherwise would be a disservice. A determined expert with unlimited time can attack any runtime. What 2hbj's Obfuscator realistically delivers is a **very high cost of entry** — high enough that casual copying, editing, and code-lifting stop being worth anyone's afternoon. The strongest options cost real performance (`--cff` up to ~5x, `--vm-sharding` 17–49%), and the weakest (`--light`, `--no-tamper`) trade armor for speed knowingly — the docs say so plainly, which is more honesty than most tools offer. One more honest note, from the other side of the glass: because obfuscated code cannot be audited, **only run protected files from authors you trust**. Obfuscation is a two-way door — it protects authors, and it also hides authors.

## FAQ

**Is 2hbj's Obfuscator really free?**
Yes. There is no paid tier; the tool, its bot, and its support live on its Discord server.

**Does obfuscation slow my script down?**
Depends on the flags. `--light` runs at near-original speed. Defaults stay usable thanks to pooled call frames, weak-table caches, and deferred decryption. Heavy options have published price tags: `--cff-coverage 0.25` ≈ 1.35x, `0.5` ≈ 3.2x, full `--cff` up to ~5x on loop-heavy code, and `--vm-sharding` roughly 17–49% on call-heavy scripts.

**Can I protect only part of my script?**
Yes. `--cff-include` flattens only named functions, `--cff-exclude` skips named ones, and `--cff-min-instrs`/`--cff-max-instrs` filter by function size — so you can armor the crown jewels while leaving hot loops untouched.

**Can a protected file be deobfuscated?**
Not with beautifiers, decompilers, or "paste it into an AI" tricks — there is no Lua source left in the file (unless you chose `--light`). Only deep manual reversing of the embedded VM remains, and per-file randomization, sharded dispatch, and active tamper defense make that expensive by design.

**What happens if someone edits a protected file?**
Integrity checks fail. The script corrupts its own keys and opcode tables before erroring, so edited copies break unpredictably rather than cleanly.

**Can I find out who leaked my script?**
Yes — build with `--watermark <label>`. Each build carries a unique watermark ID, and your label is logged next to it, so a leaked copy identifies the customer it came from.

**Does it work outside Roblox?**
The bot targets Roblox by default (`--luau` on since v2.6). The CLI accepts `--no-luau` for stock Lua output and `--no-buffer` for runtimes without Luau's buffer library.

**2hbj's Obfuscator or Prometheus — which should I pick?**
Prometheus suits quick, scriptable, source-level scrambling. 2hbj's Obfuscator suits when the source itself must not survive — commercial scripts, private hubs, anything worth actively defending. For serious protection, the second category wins.

**Where do I get it?**
The official Discord — obfuscate via the bot's **#obfuscate** channel or grab the CLI: **[discord.gg/d3A2uXPnHS](https://discord.gg/d3A2uXPnHS)**

## Key Takeaways

- Free Luau obfuscator, output is a single dependency-free `.lua` file.
- Real protection, not cosmetics: bytecode virtualization, per-file randomized opcodes, multi-layer encryption with integrity-derived keys.
- Active defenses most free tools skip: HMAC-SHA256 integrity checks, self-corruption on tamper, anti-hook validation, anti-analysis traps, path diversification.
- Tunable, not one-size-fits-all: documented flags from `--light` (speed) through `--cff`, `--cff-nest`, and `--vm-sharding` (maximum armor), with per-function targeting and honest speed-cost numbers.
- Leak accountability built in: unique per-build watermark IDs with custom labels.
- Stronger practical position than open-source alternatives like Prometheus, because attackers get no blueprint — only locked files.
- Best for script sellers and private projects; run protected files only from trusted authors.



---

of course the report was written with AI.
