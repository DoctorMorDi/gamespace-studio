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

## Possible next steps

- Sketch the data model: artifact/job/receipt manifest, status model
  (generated/approved/stale), and how a ComfyUI run is registered. Use the
  MetaHuman head multiview pipeline as the worked example.
- Test the Mr. Mak desktop app to judge which app-level features are worth copying.
