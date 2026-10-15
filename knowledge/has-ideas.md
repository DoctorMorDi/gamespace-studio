# Ideas for Hermes-Asset-Studio (HAS)

Working note from reviewing this repository (a Mr. Mak Workspace fork) as
inspiration for HAS. Started 2026-09-25. Status: ideas, nothing decided.

## Core insight

HAS already has the engines (ComfyUI, Blender, cloud APIs). What is missing is
an **asset layer above them**: what was made from what, which version is
approved, and what becomes stale when something upstream changes. Today that
job is done by the ComfyUI output folder plus copied experiment folders, which
makes fine-grained refinement and single-component regeneration hard.

Proposal: don't replace the ComfyUI graph. Add an **asset lineage graph** where
each node is a concrete intermediate artifact and each edge is a job (ComfyUI
workflow, Blender script, API call) with a receipt.

## Borrowable mechanisms from this repo

1. **One receipt per job, resume-safe.** `.agents/skills/fal-ai-generation/scripts/fal_job.py`
   writes `job.json` with an exclusive create before the submit completes; an
   existing receipt blocks resubmission ("use status/result"). Uploads get
   sha256 receipts and are reused. Rule: a timeout never authorizes a duplicate
   paid job. Maps directly to ComfyUI prompt IDs and Tripo/Meshy/etc. tasks.
2. **`state.json` as pipeline authority** (`.agents/skills/img2threejs/SKILL.md`, `forge/`).
   `next.py` reports the next step and exact command; `state.py mark <step>
   --evidence <file>` completes it; skipping requires a reason; correction loops
   are bounded (3 per pass, 6 total). "Conversation context is disposable."
3. **Approved source as a hashed anchor** (`character-sheet-pipeline`). All parts
   are extracted from the same approved strict-front A-pose image, with its hash
   recorded. Identity preservation becomes a process property.
4. **Quality gates with JSON verdicts** (`character-sheet-pipeline/prompts/08_3q_classifier.md`):
   pass / soft_fail / hard_fail, boolean checks, estimated camera rotation.
   Deterministic checks run before any vision model. Fits multiview validation.
5. **Three separate acceptance levels:** technically complete → visually
   approved → works in engine. The final gate is reimport into a clean scene
   (scale, bones, skin, texture links). Good gate before UE5/MetaHuman.
6. **Source vs runtime separation** (`materials-to-game`, `blender-game-animation`):
   keep high-poly and control rig; low-poly and runtime skeleton are separate outputs.
7. **App-level:** atomic JSON writes with compare-and-retry (`desktop/service/workspace.mjs`,
   `util.mjs`), `fs.watch` → live UI updates, agent session status read from the
   CLIs' real transcript files (`desktop/service/sessions.mjs`), and the overall
   pattern of agent chat beside a visual workspace.
8. **Weakness to avoid:** the Arachne card stores prompts only inside HTML
   reports, with no seed, parent artifact or machine-readable lineage. In HAS the
   manifest should be the source of truth and views should be generated from it.

## UI direction (from Lychee Studio examples and the HAS Figma draft)

- Lychee canvas: character → prop extraction → one branch per part → image-to-3D,
  each branch re-runnable alone. Good **data model**.
- Multi-view node: one step, several related outputs. Each view should be its
  own node with its own status so only "Right" can be regenerated.
- Figma draft (stage nav: Concept Studio, 3D Generator, Part Separator, Assembly,
  Animation, Export Studio): good **primary UI** for professional work.
  Recommendation: lineage graph as the data model, stage views as the main UI,
  graph view as a secondary tab.

Figma draft suggestions:

1. Status beyond READY/PENDING: *generated*, *approved*, *stale* (upstream changed).
2. Per-view actions: Regenerate, Edit (identity-preserving), Compare with previous,
   Approve, pick an earlier version.
3. Asset library shows lineage on click (source, workflow, parameters) and a
   version badge (e.g. v3 ★ for approved).
4. Settings panel values (head type, resolution, image count) plus workflow hash
   and seed are stored in the job receipt, making every output reproducible.
5. Keep the GPU / VRAM / queue status bar; wire it to the ComfyUI queue and history.

## Experiments as branches, not folder copies

Store files once (content-addressed by hash) and let a manifest point to them.
An experiment is a new branch from an approved node: no duplication, full trace
back. Readable folders for Blender/ComfyUI can still be exported or linked.

## Unreal / MetaHuman

Treat it as its own stage with its own acceptance: Blender + DNA tooling
(e.g. Polyhammer Character DNA) → DNA file + mesh → UE import → in-engine
validation. Open question: what the UE5 Python API and MetaHuman tooling can
automate in the version in use. Research before deciding.

## Positioning: HAS is not built on Mr. Mak

Mr. Mak is a general agent-work shell (chats, cards, files) for game development;
its pipeline skills are loose recipes. HAS is a domain system: a production line
with strict requirements for every intermediate. Borrow the patterns (receipts,
state file, acceptance levels), not the app. If an agent shell is wanted later,
HAS should be a module it talks to (MCP or API), never dependent on one shell.

## Modular stages as contracts

Each stage (multiview, segmentation, reconstruction, retopo, UV, texturing,
rigging, export) is a module with a fixed contract:

- **Inputs:** which artifacts it needs, and in which approved version.
- **Outputs:** what it produces, format and conventions.
- **Acceptance:** what must hold before the next stage may consume it.
- **Executors:** ComfyUI workflow, Blender script or cloud API. Several
  executors can satisfy the same contract (swap Tripo for Hunyuan or a local
  model without the rest noticing). This is what makes single-component
  regeneration possible.

## Target profiles (UE5/MetaHuman first, Unity possible)

The engine choice constrains upstream stages, so make it explicit: a **target
profile** set at project start that pushes requirements backwards.

| | UE / MetaHuman | Unity |
|---|---|---|
| Topology | Locked to the MetaHuman template (DNA / Mesh to MetaHuman) | Free, but must skin well |
| Rig | MetaHuman skeleton + DNA (facial rig) | Humanoid avatar (bone mapping) |
| Textures | Packed channels per UE convention | Different packing, differs URP vs HDRP |
| Hair | Strand groom (UE Groom) | Usually card/mesh hair |
| Scale / axes | cm, Z up | m, Y up |

MetaHuman is the strongest constraint (locks topology, rig, face); Unity is
more forgiving. Verify table details against the versions in use before
encoding them.

**Branch late:** keep sources target-neutral as long as possible. Concept art,
multiview references and high-poly sculpt are mostly engine-agnostic; the fork
happens around retopo and rigging, so a MetaHuman branch and a Unity branch can
split from the same approved high-poly.

Open question: Epic changed the MetaHuman license in 2025 to allow use outside
Unreal. If confirmed, Unity could be an export variant of the MetaHuman line
rather than a separate pipeline.

For now: treat the target as a parameter (e.g. `target: ue5-metahuman`), not an
assumption baked into the core, and put profile requirements into the
acceptance criteria of affected stages. No Unity work needed yet.

## Possible next steps

- Continue HAS work in its own clean repository (decided 2026-09-26); carry
  this note over as the starting point.

- Sketch the data model: artifact/job/receipt manifest, status model
  (generated/approved/stale), and how a ComfyUI run is registered. Use the
  MetaHuman head multiview pipeline as the worked example.
- Test the Mr. Mak desktop app to judge which app-level features are worth copying.
