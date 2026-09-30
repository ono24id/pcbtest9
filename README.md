# PROJECT_NAME

**English** | [Bahasa Indonesia](README.id.md)

KiCad 10 hardware project.

> ⚠️ **Generated Code Disclaimer:** This project contains code generated with the assistance of AI tools. While the core functionality has been thoroughly tested and validated, please review all code before use in production environments.

**Contents**
- [Using this template](#using-this-template)
- [Layout](#layout)
- [Adding parts and libraries](#adding-parts-and-libraries)
  - [A. KiCad built-in libraries](#a-kicad-built-in-libraries)
  - [B. Git repository of a KiCad library (submodule)](#b-git-repository-of-a-kicad-library-submodule)
  - [C. Vendor download (zip)](#c-vendor-download-zip)
  - [D. Draw it yourself](#d-draw-it-yourself)
  - [After adding a part](#after-adding-a-part)
- [AI assistants (MCP)](#ai-assistants-mcp)
- [CI & releases](#ci--releases)

## Using this template

1. Click **Use this template → Create a new repository** on GitHub. The repository name
   becomes the KiCad project name (e.g. `sensor-board`).
2. The **KiCad** workflow runs once:
   - job **Rename project**: replaces `PROJECT_NAME` with the repository name in file names and contents
     (`sources/sensor-board/sensor-board.kicad_pro`, `libraries/sensor-board.kicad_sym`, etc.),
     regenerates the root schematic UUID, removes this section from both READMEs and commits the
     result as `github-actions[bot]`
   - job **Build**: ERC, DRC and fabrication outputs for the renamed project
3. `git pull`, then open the `.kicad_pro` file in KiCad.

After that, every push shows a single *"Build: …"* run (with **Rename project** skipped).

Cloned locally without GitHub? Run it yourself:

```bash
scripts/init.sh my-project   # no argument: uses the repository folder name
```

> If the rename commit is rejected, go to **Settings → Actions → General → Workflow permissions**,
> select **Read and write permissions**, then start the workflow manually
> (Actions tab → KiCad → Run workflow).
<!-- template:end -->

## Layout

```
sources/PROJECT_NAME/        KiCad project (.kicad_pro/.kicad_sch/.kicad_pcb) + lib tables
libraries/
  PROJECT_NAME.kicad_sym     project-specific symbols
  PROJECT_NAME.pretty/       project-specific footprints
  PROJECT_NAME.3dshapes/     3D models (STEP/WRL)
  external/<name>/           external libraries (git submodules)
scripts/                     init.sh, add/remove-library.sh, mcp-kicad.sh
.mcp.json, .vscode/, .cursor/ MCP config for AI assistants
AGENTS.md, CLAUDE.md         instructions for AI agents
.github/workflows/kicad.yml  ERC, DRC and fabrication outputs
```

The project libraries are registered in the project's `sym-lib-table` / `fp-lib-table` using
`${KIPRJMOD}/../../libraries/...`, so they keep working wherever the repository is cloned.
For 3D models, set the footprint model path to `${KIPRJMOD}/../../libraries/PROJECT_NAME.3dshapes/<file>.step`.

The title block uses the text variables `${PROJECT}`, `${REVISION}` and `${CURRENT_DATE}`.
`REVISION` is `dev` inside KiCad and is filled in by CI from the git tag / commit.

## Adding parts and libraries

First ask: **where does the part come from?** That decides the path.

| The part is… | Path | Ends up in |
| --- | --- | --- |
| in KiCad's built-in libraries | **A.** use it directly | – |
| in a git repository of a KiCad library | **B.** `scripts/add-library.sh <url>` | `libraries/external/<name>/` (submodule) |
| a vendor download (SnapEDA, Ultra Librarian, Mouser/DigiKey zip) | **C.** import it | `libraries/PROJECT_NAME.*` |
| not available anywhere | **D.** draw it | `libraries/PROJECT_NAME.*` |

### A. KiCad built-in libraries

Always check here first: resistors, capacitors, LEDs, pin headers, common regulators, the
Raspberry Pi 40-pin header, ... Press **A** in the schematic editor and search. Built-in libraries
come with KiCad (and the CI container), so nothing needs to be registered.

### B. Git repository of a KiCad library (submodule)

```bash
scripts/add-library.sh https://github.com/<owner>/<kicad-lib>.git          # -> libraries/external/<kicad-lib>
scripts/add-library.sh https://github.com/<owner>/<kicad-lib>.git mylib -b main
git commit -m "Add mylib library"
```

The script runs `git submodule add`, then registers every `*.kicad_sym` and `*.pretty` found in
the submodule in the project's `sym-lib-table` / `fp-lib-table` (nickname = file name, path via
`${KIPRJMOD}`). Nicknames that already exist are skipped. Reopen the project in KiCad afterwards.

Submodules keep the repository small, pin the exact library version per board revision, and can be
updated when the vendor fixes something. **Never edit files in `libraries/external/`**; to change a
part, copy it into the project library (path C/D).

```bash
git clone --recursive <repo-url>              # clone including libraries
git submodule update --init --recursive       # after a normal clone / pull
git submodule update --remote libraries/external/mylib # update a library to its latest commit
```

To remove a library:

```bash
scripts/remove-library.sh mylib
git commit -m "Remove mylib library"
```

This deinitialises and removes the submodule (including its copy in `.git/modules`) and drops its
entries from the lib tables. It refuses to run while the schematic or board still use a symbol or
footprint from that library; replace those parts first, or pass `--force`.

Use `https://` URLs so CI can fetch them. For private library repositories, add a repository
secret `SUBMODULE_TOKEN` (a PAT with read access); the CI checkout uses it automatically.

### C. Vendor download (zip)

Import the files into the project's own library, which is already registered:

| File | Put it in | How |
| --- | --- | --- |
| Symbol (`.kicad_sym`) | `libraries/PROJECT_NAME.kicad_sym` | Symbol Editor → select library `PROJECT_NAME` → **File → Import Symbol** |
| Footprint (`.kicad_mod`) | `libraries/PROJECT_NAME.pretty/` | copy the file, or Footprint Editor → **File → Import Footprint** |
| 3D model (`.step`) | `libraries/PROJECT_NAME.3dshapes/` | copy the file; in the footprint's **Properties → 3D Models** use `${KIPRJMOD}/../../libraries/PROJECT_NAME.3dshapes/<file>.step` |

Then set the symbol's **Footprint** field to `PROJECT_NAME:<footprint>`.

> Always check a downloaded footprint against the datasheet (pad size, pitch, pin 1). Footprint
> mistakes are only discovered once the boards arrive.

### D. Draw it yourself

1. Symbol Editor → **New Symbol** in library `PROJECT_NAME`. Set every pin's **electrical type**
   (Input, Output, Power input, ...) correctly: ERC relies on it.
2. Footprint Editor → **New Footprint** in `PROJECT_NAME.pretty`, using the datasheet's recommended
   land pattern (or the **Footprint Wizard** for standard packages such as QFN or SOIC).
3. Set the symbol's **Footprint** field and, optionally, a 3D model as in C.

### After adding a part

1. Place it in the schematic and run **Update PCB from Schematic** (F8).
2. Run ERC and DRC (or ask the AI assistant to).
3. `git push`. If CI fails although it passes locally, a library path is usually absolute
   (`/Users/...`) instead of `${KIPRJMOD}`-relative.

## AI assistants (MCP)

The repository ships a ready-to-use [Model Context Protocol](https://modelcontextprotocol.io) server
for KiCad, [kicad-mcp-pro](https://github.com/oaslananka/kicad-mcp-pro), already pointed at this project:

| Client | Config |
| --- | --- |
| Claude Code | `.mcp.json` (approve the `kicad` server on first start) |
| VS Code / Copilot | `.vscode/mcp.json` |
| Cursor | `.cursor/mcp.json` |
| Other clients | command `bash scripts/mcp-kicad.sh` (stdio) |

Requirements: [uv](https://docs.astral.sh/uv/getting-started/installation/) (`uvx`) and KiCad 10
with `kicad-cli` on `PATH`. On Windows, run through Git Bash / WSL.
The server finds `sources/*/*.kicad_pro` itself, so nothing needs to be edited per project.

Behaviour can be tuned with environment variables read by `scripts/mcp-kicad.sh`:
`KICAD_MCP_OPERATING_MODE` (`readonly`, `write` *(default)*, `manufacturing`),
`KICAD_MCP_PROFILE` (`default`, `review`, `build`, `release`, `full`, ...) and
`KICAD_MCP_PACKAGE` (e.g. `kicad-mcp-pro==3.35.0` to pin a version).

Project conventions for AI agents live in [`AGENTS.md`](AGENTS.md) (`CLAUDE.md` imports it).

## CI & releases

Every push / pull request runs [`kicad.yml`](.github/workflows/kicad.yml) in the
`kicad/kicad:10.0` container:

| Step | Output |
| --- | --- |
| ERC, DRC (+ schematic parity) | `reports/erc.rpt`, `reports/drc.rpt` — the job fails on errors |
| Schematic | `PROJECT_NAME-schematic.pdf`, `PROJECT_NAME-bom.csv` |
| PCB | `gerbers/` + `PROJECT_NAME-gerbers.zip`, drill + drill map, `PROJECT_NAME-pos.csv`, `PROJECT_NAME-pcb.pdf`, `PROJECT_NAME.step` |

Outputs can be downloaded from the **Actions** tab (artifacts). Gerber layers follow the
**File → Plot** settings saved in the board.

For a production release:

```bash
git tag v1.0 && git push origin v1.0
```

If ERC/DRC are clean, a GitHub Release `v1.0` is created with the full zip, Gerbers, schematic, BOM and placement file.
