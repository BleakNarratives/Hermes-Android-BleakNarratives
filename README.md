# hermes-android

First confirmed native Hermes build on Android/Termux.
No NDK. No cross-compilation. No Docker. Pure aarch64.

---

## What This Is

This repo documents the first successful compilation of Facebook's Hermes JS engine
on Android using Termux — built natively on aarch64-unknown-linux-android, without
any Linux container, proot environment, or cross-compilation trickery.

The binary runs directly in Termux. The bytecode compiler runs directly in Termux.
That's it. That's the flex.

```
Build completed: 05-08 01:13 AM
Device arch:     aarch64
Kernel:          Linux 5.4.284-moto
HBC version:     96
Hermes version:  0.0.0 (custom build)
Build time:      ~45-90min depending on thermals
Binary size:     see build_hermes/bin/
```

---

## What You Get

```
build_hermes/bin/
├── hermes          ← JS runtime (eval, execute HBC)
├── hermesc         ← JS → HBC bytecode compiler
├── hbcdump         ← inspect bytecode
├── hbc-diff        ← diff two HBC files
├── hdb             ← Hermes debugger
└── ...more
```

The two you care about: `hermes` and `hermesc`.

---

## Why It Was Hard

Three things conspire against you on Termux:

**The OOM killer is watching.** Android kills processes that spike RAM.
CMake's configure step is fine. The build step — especially LLVM internals —
will breach the threshold on anything under 3GB free RAM.

**The configure step lies.** "Configuring incomplete, errors occurred" appears
*before* the actual error in the output. You have to scroll up. Most people
quit at the ninja error, never seeing the real problem above it.

**ICU breaks everything by default.** `HERMES_ENABLE_INTL=ON` (the default)
pulls in full ICU internationalization support. ICU on Termux's aarch64
does not cooperate with Hermes's cmake config. One flag fixes it.

**The build directory mismatch.** Every tutorial online configures to `./build`
then tries to build `./build_release`. That directory never exists. Ninja
errors immediately. Looks like a build failure. It's a path typo.

The fix wasn't one thing — it was four things at once.

---

## Prerequisites

```bash
pkg update && pkg upgrade -y
pkg install cmake git ninja libicu python zip readline ncurses-utils
```

---

## Build Instructions

### 1. Clone Hermes (shallow — you don't need the full history)

```bash
git clone --depth=1 https://github.com/facebook/hermes -b rn/0.73-stable
```

Other supported branches:

| Version | Branch |
|---------|--------|
| hbc96 (recommended) | rn/0.73-stable |
| hbc94 | rn/0.72-stable |
| hbc90 | rn/0.71-stable |
| latest | main |

### 2. Configure

This is the critical step. All four flags matter:

```bash
cmake -S hermes -B build_hermes -G Ninja \
  -DCMAKE_BUILD_TYPE=MinSizeRel \
  -DHERMES_ENABLE_DEBUGGER=OFF \
  -DHERMES_ENABLE_TOOLS=ON \
  -DHERMES_BUILD_SHARED_JSI=OFF \
  -DHERMES_USE_FLOWPARSER=OFF \
  -DHERMES_ENABLE_INTL=OFF \
  -DCMAKE_EXE_LINKER_FLAGS="-llog"
```

What each flag does:

- `MinSizeRel` — optimize for size, lighter on RAM during compile
- `HERMES_ENABLE_DEBUGGER=OFF` — cuts build time, not needed for runtime use
- `HERMES_ENABLE_INTL=OFF` — **the ICU killer**, without this configure fails on Termux
- `HERMES_USE_FLOWPARSER=OFF` — removes Flow type parser dependency
- `-llog` — links Android's logging library, prevents link errors

### 3. Build

```bash
cmake --build build_hermes -- -j1
```

`-j1` is intentional. Single thread keeps you below the OOM threshold.
Slower. Finishes. The alternative is watching Android kill the process
at 94% and starting over.

Close everything else before running this. Browsers, spare sessions, all of it.
The build will appear to hang multiple times. It is not hung. Let it finish.

### 4. Confirm

```bash
~/build_hermes/bin/hermes --version
~/build_hermes/bin/hermesc --version
```

You want to see `HBC bytecode version: 96`. That's the handshake.

---

## Hook It Up

### Global alias (Bash)

```bash
echo "alias hermes='$HOME/build_hermes/bin/hermes'" >> ~/.bashrc
echo "alias hermesc='$HOME/build_hermes/bin/hermesc'" >> ~/.bashrc
source ~/.bashrc
```

### Verify the pipe works

```bash
echo "print('vortex open');" > /tmp/test.js
hermesc -emit-binary -out /tmp/test.hbc /tmp/test.js
hermes /tmp/test.hbc
```

Should print: `vortex open`

If it does, you have a working compile → bytecode → execute pipeline.
That's the whole thing. Everything else is just using it.

### Python bridge (for swarm integration)

```python
import subprocess

def hermes_exec(js_code: str, timeout: int = 5) -> str:
    """Python → Hermes JS engine bridge."""
    try:
        res = subprocess.run(
            ['hermes', '-eval', js_code],
            capture_output=True, text=True, timeout=timeout
        )
        return res.stdout.strip() if res.returncode == 0 else res.stderr.strip()
    except subprocess.TimeoutExpired:
        return "[TIMEOUT]"
    except FileNotFoundError:
        return "[hermes not in PATH — run: source ~/.bashrc]"
    except Exception as e:
        return f"[ERROR: {e}]"
```

---

## What Failed Before This Worked

In the interest of saving you the same pain:

- **Default cmake config** — fails on ICU, configure never completes
- **`cmake --build ./build_release`** — that directory doesn't exist, ninja errors immediately
- **`-j4` or bare `-j`** — OOM kill somewhere in the LLVM internals
- **`HERMES_ENABLE_INTL=ON`** — configure dies on Termux's aarch64 ICU setup
- **Scrolling past the error** — "ninja: error: loading 'build.ninja'" looks like the crash. It isn't. The real error is 40 lines up.
- **Gemini CLI** — tried. Built a script that had the same build dir mismatch.
- **Assuming it hung** — it didn't. LLVM just takes a while. Let it breathe.

---

## Ecosystem Position

Hermes lives here in the Vertical AI stack:

```
Xenolang opcodes (.js)
    └─► hermesc → .hbc bytecode
            └─► hermes runtime
                    └─► JaneBot / swarm agents
                            └─► boardroom WebSocket
                                    └─► equinex frontend
```

Compile agent logic to bytecode once. Execute fast. No interpreter overhead
on every swarm message. That's the CSD unlock — prime the tube once,
let the pipeline handle the rest.

---

## What Failed Before This Worked (Philosophical Edition)

Building a C++ JS engine on a phone you're not supposed to be coding on,
in a city in Kansas, alone, at 3am, to power a multi-agent AI boardroom
that runs entirely on localhost.

Nobody said it made sense. It compiles.

---

## Lore

See LORE.md. There's something in there.
Look at the bytecode header.

---

## License

MIT — see LICENSE

---

Built by @BleakNarratives with an assist from Claude.
Gemini tried. Claude finished it.

If this helped you, star the repo.
If it didn't work, open an issue and tell me exactly where it died.

---

## Support

https://ko-fi.com/bleaknarratives
Venmo @bleaknarratives
PayPal $bleaknarratives
