# Setup Guide

> How to set up your PC so that Primordia can be developed, and so AI can drive the tools — Blender included — directly on your machine.

**Status:** v1, 2026-09-30. Companion to [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md).

Read §1 first — it answers the question you actually asked, and it changes how you work.

---

## 1. Can AI use programs on your PC?

**Yes, with one change: run Claude Code locally instead of in the browser.**

Right now this session is running in a cloud container. It has no access to your machine — it can't see your files, can't launch Blender, can't render anything for you. That's why this plan is documents rather than a working app.

Claude Code also runs **on your own computer**, as a desktop app or a terminal command. There, it can:

- read and write files in your project folder
- run shell commands — which means running Blender, Python, Rust, git, anything installed
- launch and script Blender headlessly, then look at what came out
- run the app it's building, take a screenshot, and iterate on what it sees

That last one matters more than it sounds. Locally, the loop is *write code → build → run → look → fix*, without you relaying anything. In the cloud, every one of those steps needs you in the middle.

**Recommendation: do the actual development locally.** Use cloud sessions like this one for planning, research and writing.

### 1.1 What "letting it run Blender" actually looks like

Claude Code asks permission before running a command. You can pre-approve patterns so it doesn't ask every time. In `.claude/settings.json` inside the project:

```json
{
  "permissions": {
    "allow": [
      "Bash(cargo:*)",
      "Bash(pnpm:*)",
      "Bash(npm:*)",
      "Bash(uv:*)",
      "Bash(python:*)",
      "Bash(pytest:*)",
      "Bash(blender --background --python assets/bake/*)",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ]
  }
}
```

Two deliberate choices: the Blender rule only allows **headless** runs of scripts inside `assets/bake/`, and git writes (`commit`, `push`) are absent so they still prompt. Start narrow; widen as you get comfortable.

There are also MCP servers that give AI live, interactive control of a running Blender instance. Useful for exploratory work, unnecessary for this project — a scripted, reproducible bake step is strictly better for an accuracy-first pipeline, because the output is regenerable rather than hand-made.

---

## 2. Core toolchain

Install these first. Everything else is optional.

### 2.1 Git

- **Windows:** [git-scm.com/download/win](https://git-scm.com/download/win)
- **macOS:** `xcode-select --install`
- **Linux:** `sudo apt install git`

### 2.2 Rust

The simulation core and the Tauri backend. From [rustup.rs](https://rustup.rs):

```bash
# macOS / Linux
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows: download and run rustup-init.exe from rustup.rs
```

Verify: `rustc --version` and `cargo --version`.

### 2.3 Node.js

The UI. Install the current LTS from [nodejs.org](https://nodejs.org), then enable the package manager:

```bash
corepack enable
corepack prepare pnpm@latest --activate
```

Verify: `node --version` (want 20+) and `pnpm --version`.

### 2.4 Python

The reference lane and the asset pipeline. Use [uv](https://docs.astral.sh/uv/) — it manages Python versions as well as packages, which keeps things reproducible:

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Then: `uv python install 3.12`. Verify: `uv --version`.

### 2.5 Tauri prerequisites

Platform-specific system libraries. Check the [current Tauri prerequisites page](https://tauri.app/start/prerequisites/) — this list changes.

**Windows**
1. **Microsoft C++ Build Tools** — install with the "Desktop development with C++" workload.
2. **WebView2** — preinstalled on Windows 11 and current Windows 10; install the Evergreen Runtime if missing.

**macOS**
```bash
xcode-select --install
```

**Linux (Debian/Ubuntu)**
```bash
sudo apt update
sudo apt install libwebkit2gtk-4.1-dev build-essential curl wget file \
  libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev
```

### 2.6 Verify

```bash
cargo install create-tauri-app
cargo create-tauri-app --help
```

If that runs, the toolchain is complete.

---

## 3. Blender

Used as an **offline asset compiler**, never at runtime. You bake molecular geometry to glTF once; the app just loads the result.

### 3.1 Install Blender

Download the current stable release from [blender.org/download](https://www.blender.org/download/). Install normally.

**Then make sure it's on your PATH**, so scripts can call it:

- **Windows:** add `C:\Program Files\Blender Foundation\Blender 5.x\` to your PATH environment variable (System Properties → Environment Variables → Path → New).
- **macOS:** `echo 'export PATH="/Applications/Blender.app/Contents/MacOS:$PATH"' >> ~/.zshrc` then restart the terminal. (The binary is named `Blender` — you may want an alias `blender`.)
- **Linux:** symlink the extracted binary into `~/.local/bin/blender`.

Verify:

```bash
blender --version
blender --background --python-expr "import bpy; print('bpy OK', bpy.app.version_string)"
```

The second command is the important one: it proves headless scripting works, which is the mode everything here uses.

### 3.2 Molecular Nodes

A Blender add-on for importing and rendering molecular data, built on Blender's Geometry Nodes system. It ships a large library of molecule-specific nodes and handles the standard structure formats.

**Install:** in Blender, `Edit → Preferences → Get Extensions`, search for **Molecular Nodes**, click Install. (It's on the official Blender Extensions platform, so this is a one-click install with no manual downloads.)

**Test it:** in Blender's Scripting workspace,

```python
import bpy
print([a for a in bpy.context.preferences.addons.keys() if "molecular" in a.lower()])
```

**Important caveat, and the reason for the pinning advice below:** the Molecular Nodes Python API is explicitly experimental and has changed substantially between releases. Treat every bake script as version-locked.

### 3.3 Pin your versions

Because the add-on API moves, record exact versions in `assets/bake/VERSIONS.md` and check the baked outputs into the repo (or a cache). The app must never depend on Blender being installed — only the *rebuild* step does.

```markdown
Blender:         5.1.2
Molecular Nodes: 4.5.13
Baked:           2026-09-30
```

### 3.4 Alternative: Blender as a Python package

If you'd rather not install the full application, Blender's Python module can be installed directly:

```bash
uv pip install bpy==<version matching your target Blender>
```

This gives headless Blender inside a normal Python environment — convenient for CI. It does **not** include add-ons, so Molecular Nodes would need installing separately into that environment, which is fiddly. **Recommendation: use the full Blender install for baking, and consider `bpy` only if you later want asset rebuilds in CI.**

---

## 4. Scientific tooling (optional, for the reference lane)

Not needed to start. Add when you reach Phase 5.

```bash
cd primordia/reference
uv venv
uv pip install biopython numpy scipy pandas matplotlib pytest
uv pip install ViennaRNA          # RNA folding; check the license terms for your use
uv pip install cobra              # constraint-based metabolic modeling, later phases
```

**ChimeraX** ([cgl.ucsf.edu/chimerax](https://www.cgl.ucsf.edu/chimerax/)) is worth installing separately — it's excellent for inspecting structures by hand and for scripted preprocessing (chain selection, surface generation) before the Blender step. It has a command line and a Python API.

---

## 5. The asset bake pipeline

The shape of the workflow, so you can see what Claude will be running.

```
data/structures/*.cif          ← downloaded from PDB / AlphaFold DB
        │
        ▼  reference/ingest/prepare_structure.py
   cleaned structure + coarse bead model
        │
        ▼  blender --background --python assets/bake/bake_structure.py
   LOD meshes, baked ambient occlusion + normal maps, impostor sprite atlas
        │
        ▼  glTF + KTX2
   assets/baked/<entry>/
        │
        ▼
   loaded by the app at runtime (Blender not required)
```

A bake script looks roughly like this — illustrative, since the add-on API will have moved by the time it's written:

```python
# assets/bake/bake_structure.py
# Run: blender --background --python assets/bake/bake_structure.py -- 1EHZ cartoon
import bpy, sys, pathlib

argv = sys.argv[sys.argv.index("--") + 1:]
entry, style = argv[0], argv[1]

# start from an empty scene
bpy.ops.wm.read_factory_settings(use_empty=True)

# import the structure via Molecular Nodes, apply a representation,
# generate LOD levels by decimation, bake AO and normals to textures,
# then export glTF to assets/baked/<entry>/
...

out = pathlib.Path("assets/baked") / entry
out.mkdir(parents=True, exist_ok=True)
bpy.ops.export_scene.gltf(filepath=str(out / f"{style}.glb"), export_format="GLB")
print(f"baked {entry}/{style}")
```

**The rule that keeps this maintainable:** bake scripts are checked in, deterministic, and take all inputs as arguments. Nothing is done by hand in the Blender GUI, because hand-made assets can't be regenerated when something changes.

---

## 6. Running Claude Code locally

1. Install it: see [claude.com/claude-code](https://claude.com/claude-code) for the current installation method (there's a desktop app and a terminal CLI).
2. Clone your repo and open a session in the project folder.
3. Add the `.claude/settings.json` permissions from §1.1.
4. Point it at these documents: *"Read primordia/IMPLEMENTATION_PLAN.md and ACCURACY_ROADMAP.md, then start Phase 0."*

**Worth adding early:** a `CLAUDE.md` at the repo root with the project's standing rules, which every session reads automatically. For this project the important ones are the accuracy constraints:

```markdown
# Primordia

An accuracy-first genetic code simulator. See primordia/IMPLEMENTATION_PLAN.md.

## Non-negotiable rules
- No bare numeric constants in simulation code. Every parameter goes through
  the provenance system with a citation or an explicit `Assumed` tag.
- Units are checked at compile time. Never use a bare f64 for a physical quantity.
- All randomness comes from the seeded engine RNG. Never call a system RNG.
- New simulation behavior ships with a validation test. See the validation suite.
- Fast lane (Rust) and reference lane (Python) must agree within declared tolerance.

## Commands
- `cargo test --workspace` — Rust tests
- `pnpm -C ui test` — frontend tests
- `uv run pytest reference/` — reference lane tests
- `cargo tauri dev` — run the app
```

---

## 7. Verification checklist

Work through this before starting Phase 0. Every line should succeed.

```bash
git --version
rustc --version                 # 1.80+
cargo --version
node --version                  # 20+
pnpm --version
uv --version
blender --version
blender --background --python-expr "import bpy; print('bpy OK')"
cargo install create-tauri-app  # proves the native toolchain links
```

Then, the real test — scaffold a throwaway Tauri app somewhere outside the project and confirm a window opens:

```bash
cargo create-tauri-app primordia-smoketest
cd primordia-smoketest
pnpm install
pnpm tauri dev
```

If a window appears, your machine is ready. Delete the smoke test and start Phase 0.

---

## 8. Hardware notes

| Component | Minimum | Comfortable | Why |
|---|---|---|---|
| CPU | 4 cores | 8+ cores | Rust compiles and multithreaded simulation |
| RAM | 16 GB | 32 GB | Blender bakes and large genomes |
| GPU | Anything with current drivers | 8 GB+ VRAM | 3D viewport; Blender rendering |
| GPU (for local AI models) | — | 24 GB+ VRAM | Only if you run genomic language models locally (see plan §11); renting by the hour is the sensible first move |
| Disk | 20 GB | 100 GB | Rust build artifacts are large; structure data accumulates |

Nothing here needs a workstation. The GPU line only matters for the optional local-AI work, and that can wait indefinitely.
