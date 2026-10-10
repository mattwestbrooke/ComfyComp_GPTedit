# Sketch-guided GPT edit: first concept

Status: **draft for discussion**, written 2026-10-10 before any wayfinder grilling. Nothing here is
decided. It exists to give the first `/wayfinder` charting session something concrete to push against.

## The idea in one paragraph

An artist in Nuke picks a frame, draws a rough sketch over it (scribbles, arrows, outlines, a few
words), types a short prompt, and sends it. ComfyUI runs a GPT image edit node with the clean frame,
the sketch and the prompt. The edited frame comes back into Nuke as a Read, versioned, next to the
original, so the artist can compare, retry or comp it in.

```
 Nuke                         ComfyUI (API)                         Nuke
 ────                         ─────────────                         ────
 plate frame ─┐
              ├─► package ──► workflow: load images ─► GPT image ─► save ──► Read node
 sketch layer ┘   (pngs +       + prompt                 edit node           (v001, v002…)
 prompt knob       request)
```

## How the user makes the sketch

The sketch is the part that needs a concept. Four candidates, roughly cheapest first:

| | Where the drawing happens | For | Against |
|---|---|---|---|
| **A. RotoPaint in Nuke** | A RotoPaint node upstream of a `GPTEdit` gizmo, painted over the plate in the Viewer | Native; artists already know it; the sketch is saved in the script and can be tweaked and resent; no UI to build | No text tool worth using; colour meaning is by convention only |
| **B. Custom sketch panel** | A PySide canvas the gizmo opens, showing the frame with a small brush/arrow/box/text palette | Purpose-built: fixed colour legend (e.g. red = remove, green = add, blue = move), text labels, undo, one click send | A real UI to build and maintain inside Nuke's Qt |
| **C. Browser canvas** | A page served beside ComfyUI (or a ComfyUI painter node), opened from Nuke like ComfyComp's Open Workflow (ADR-0006) | Same canvas on any device, including from home on a laptop or tablet; one codebase | Leaves Nuke; round trip back is a separate step |
| **D. External paint app** | Krita/Photoshop on an exported frame | Best brushes | Most friction; file juggling |

**Leaning:** start with **A** for the first working slice (zero UI cost, proves the pipeline end to
end), and prototype **B/C** as a single browser canvas, because a browser prototype can be tried
remotely today and its interaction design carries over to a PySide panel if that wins.

### What the sketch should contain

A small, fixed **sketch vocabulary** makes results more predictable than free drawing:

- **Outline / fill colour carries intent**: e.g. red = remove, green = add/replace, blue = move here.
- **Arrows** for "move this there".
- **Short text labels** drawn on the canvas ("window", "make wet").
- The **prompt** says what the sketch cannot ("night, keep the lighting").

### How the sketch reaches the model

Open question, to be researched, not assumed:

1. **Two images**: clean frame + sketch-over-frame as separate inputs, prompt explains the legend.
2. **Burned in**: one annotated image, prompt says "apply the annotations, then remove them".
3. **Sketch → mask**: the sketch's coverage becomes an edit mask, so pixels outside it are protected.

1 or 3 (or both together) look most promising, because 2 asks the model to also erase the
scribbles cleanly.

## Process in Nuke (first slice)

1. `GPTEdit` gizmo with inputs `plate` and `sketch`, knobs `prompt`, `frame`, `size`, `Send`.
2. Send renders the frame and the sketch to PNG (GPT edit wants 8-bit sRGB; ComfyComp's interchange is
   already sRGB-encoded EXR per ADR-0009, so this is a conversion at the edge).
3. Submits the workflow to ComfyUI over the API (`/prompt`), polls `/history`, fetches the result.
4. Creates a versioned Read beside the gizmo. Each send is a new version, the prompt and sketch saved
   with it.

## Things that may break this concept

- **Frames leaving the building.** ComfyComp's studio premise is that no media leaves (CONTEXT.md).
  A GPT image edit node calls OpenAI's cloud, so the frame and sketch do leave. Either this project
  runs somewhere that allows it, or it needs a studio exception, or it needs a local model instead.
  **This is the first question to settle.**
- **API-node auth over `/prompt`.** ComfyUI's paid API nodes authenticate with a Comfy account; driven
  over the API, the key has to be passed with the request. To be confirmed by research.
- **Resolution.** GPT edit outputs fixed sizes well below plate resolution, so results come back
  small and need a plan (crop to region, upscale, or accept as a look-dev tool).
- **Stills only.** One frame per request; sequences are likely out of scope for v1.

## Draft wayfinder map

**Destination (proposed):** a spec for v1 of sketch-guided GPT edit from Nuke, ready to hand to
`/implement`.

Tickets that can be stated now:

1. **Grilling: Where does this run, and may frames go to OpenAI?** Blocks almost everything.
2. **Research: GPT image edit in ComfyUI.** Which node and model ("GPT2 edit"), its inputs (multiple
   images, mask), sizes, cost, and how auth works when submitted over `/prompt`.
3. **Research: Encoding a sketch for GPT image edit.** Two images vs burned in vs mask; prior art.
4. **Prototype: Sketch canvas.** Browser canvas over a frame with the sketch vocabulary; react to it.
5. **Grilling: Relationship to ComfyComp.** Standalone package here, or a feature inside ComfyComp
   reusing its core, node pack and the API route ADR-0006 deferred.
6. **Grilling: Nuke round trip.** Gizmo shape, versioning, where outputs land.

Not yet specified: resolution strategy; cost tracking and limits; edit history and comparison UI;
whether the prompt is free text or assisted.

Out of scope (proposed): image sequences; local (non-GPT) edit models.
