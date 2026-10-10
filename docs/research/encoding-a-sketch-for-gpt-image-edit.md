# Research: encoding a sketch for GPT image edit

Ticket: GitHub issue #4, "How to give GPT image edit a sketch" (ticket 3 in
[`docs/concept/sketch-guided-edit.md`](../concept/sketch-guided-edit.md)).
Researched 2026-10-10. Sources are first-party OpenAI docs and cookbook unless labelled
**[community]** or **[paper]**. Anything not confirmed in a source is marked **[unverified]**.

## Short answer

- **No encoding gives a hard guarantee from the model.** OpenAI documents that masking for GPT image
  models is "entirely prompt-based" and that the model "uses the mask as guidance, but may not follow
  its exact shape with complete precision". The whole image goes back through the model. Pixels
  outside the edit can only be protected **by us, after the call**, by compositing the original back
  over the result outside a (feathered) mask.
- **Recommended v1: two images plus a mask, then composite back.** Image 1 = clean frame (the image
  to edit; the mask and the extra fidelity both apply to the first image). Image 2 = sketch over the
  frame, used as an annotated reference. The prompt explains the legend and names the images by
  index. The mask is the dilated sketch coverage, sent as a soft hint and then reused in Nuke/ComfyUI
  to comp the original back outside the edit.
- **Burning the sketch into the only image (option 2) is the weakest default.** It makes the model
  both obey and erase the marks; the one benchmark that grades this counts leftover arrows, boxes
  and strokes as a defect, and GPT-image-1 was the lowest-scoring proprietary model on that
  benchmark (newer GPT image models were not tested).
- **Prior art supports a small, fixed visual vocabulary** (colour for intent, arrows, boxes, short
  text labels) and warns that handwriting, arrows and custom symbols get misread. Keep labels short
  and say in the prompt what each colour means.
- **The experiment** should compare 1-image-burned-in, 2-image, 2-image + mask, and mask-only, each
  with and without composite-back, across a fixed set of edit types (remove, add, replace, move),
  scoring intent followed, marks left behind, and drift outside the edit.

## 1. What OpenAI documents about masks

Source for this section, unless stated: the image generation guide,
<https://developers.openai.com/api/docs/guides/image-generation> (markdown form
<https://developers.openai.com/api/docs/guides/image-generation.md>).

| Question | What the docs say |
|---|---|
| Hard constraint or soft guidance? | **Soft.** "Masking with GPT Image is entirely prompt-based." "The model uses the mask as guidance, but may not follow its exact shape with complete precision." "When using a mask with GPT Image, additional instructions are sent to the model to help guide the editing process accordingly." |
| Alpha channel? | "The mask image must also contain an alpha channel." "If you're using an image editing tool to create the mask, make sure to save the mask with an alpha channel." Transparent pixels mark the area to edit (the docs' wording: the mask "identifies the areas to replace"). |
| Size / format | "The image to edit and mask must be of the same format and size (less than 50MB in size)." |
| Multiple input images | "If you provide multiple input images, the mask will be applied to the first image." |
| Responses API tool | The `image_generation` tool takes `input_image_mask` (by `image_url` or `file_id`), and the same alpha requirement is stated there. An `action` parameter (`auto` / `generate` / `edit`) can force an edit; "If you force `edit` without providing an image in context, the call will return an error." |
| Is the whole image regenerated? | Not stated in those words. The docs describe edits as modifying images "either partially or entirely" and say masking is prompt-based, which implies the model re-renders the frame and the mask only steers it. Community reports below confirm the whole frame changes. |

Community evidence that the mask is soft in practice **[community]**:

- "Gpt-image-1 problems with mask edits": users report the model redraws the whole photo and changes
  pixels outside the mask; one points to OpenAI's own inpainting example, where trees and windows
  change outside the masked area. An OpenAI engineer only asked for a request ID; no fix or design
  statement was posted. <https://community.openai.com/t/gpt-image-1-problems-with-mask-edits/1240639>
- "Mask is completely ignored in the image edit api" (Aug 2025): a reply quotes the docs ("entirely
  prompt-based") and says the model "still regenerates the whole image"; another notes DALL-E 2 was
  "pixel-perfect, unable to draw anywhere not marked by the mask", which GPT image models are not.
  <https://community.openai.com/t/mask-is-completely-ignored-in-the-image-edit-api/1350867>

**Consequence:** if protecting the untouched plate matters (it does for comp work), the hard
guarantee has to come from compositing the original back over the output outside the mask. The
model also tends to shift colour and grain across the whole frame (reports of gradual yellowing over
repeated edits in the first thread above, **[community]**), so the comp edge needs a feather and
probably a colour match. Whether `gpt-image-2.5-sunburst` ("for workflows where editing precision
matters most", <https://developers.openai.com/api/docs/models/gpt-image-2.5-sunburst>) respects masks
more tightly is **[unverified]**: neither its model page nor the guide says anything about masks for
it specifically.

## 2. Multiple input images and input fidelity

- **Up to 16 input images** on the edits endpoint: "For GPT image models, you can provide up to 16
  images." Prompt up to 32,000 characters.
  <https://developers.openai.com/api/reference/resources/images/methods/edit>
- **The first image is special twice over.** The mask applies to it (guide, above), and with
  `input_fidelity="high"` "only the first one you provide is preserved with extra richness in
  texture" (cookbook, Jul 2025, archived,
  <https://developers.openai.com/cookbook/examples/generate_images_with_high_input_fidelity>). So the
  clean frame should be image 1 and the sketch image 2.
- **`input_fidelity` by model** (API reference, same URL as above): "Supports `high` and `low` on
  `gpt-image-1` and `gpt-image-1.5`; `gpt-image-1-mini` supports only `low`." "For `gpt-image-2`,
  omit this parameter. Defaults to `low` on supported models." The prompting guide (Apr 2026) says
  for `gpt-image-2`: "input_fidelity does not work for this model because output is already high
  fidelity by default" — though its own later examples still pass `input_fidelity="high"` to
  `gpt-image-2`, so treat the guide as slightly inconsistent.
  <https://developers.openai.com/cookbook/examples/multimodal/image-gen-models-prompting-guide>
  Whether the 2.5 models accept it is **[unverified]**; the 2.5 model page does not mention it.
- **High fidelity costs more:** "High input fidelity consumes more image input tokens than the
  default." (input-fidelity cookbook, above)
- **Referencing images in the prompt:** "Reference each input by index and description" (e.g. "Image
  1: product photo… Image 2: style reference…"), and "When compositing, be explicit about which
  elements move where." Image 1 is the base scene in the cookbook's compositing example ("Place the
  dog from the second image into the setting of image 1"). (prompting guide, above)
- **Current models** listed for the edits endpoint: `gpt-image-1`, `gpt-image-1-mini`,
  `gpt-image-1.5`, `gpt-image-2`, `gpt-image-2.5-sunburst`, `gpt-image-2.5-flare`,
  `chatgpt-image-latest`; default `gpt-image-2.5-sunburst`. Newer models take arbitrary sizes:
  "Width and height must both be divisible by 16 and the requested aspect ratio must be between 1:3
  and 3:1." (API reference, above). Relevant to the concept doc's resolution worry, but the ceiling
  on size was not checked here **[unverified]**.

## 3. What OpenAI says that bears on sketches and edits

From the prompting guide (<https://developers.openai.com/cookbook/examples/multimodal/image-gen-models-prompting-guide>):

- Edits: use "change only X" + "keep everything else the same", "repeat the preserve list on each
  iteration to reduce drift", and say not to alter "saturation, contrast, layout, arrows, labels,
  camera angle, or surrounding objects".
- Sketch-to-render (section 5.3) is a documented use: "Turn this drawing into a photorealistic
  image." "Preserve the exact layout, proportions, and perspective." "Do not add new elements or
  text." That example sends the drawing as the single input. It is a drawing-*becomes*-image case,
  not a marks-over-a-photo case; OpenAI has **no first-party example** of annotation marks drawn over
  a photo as edit instructions (none found in the guide, the prompting guide or the input-fidelity
  cookbook).
- Known limitation, from the guide: "the model may have difficulty placing elements precisely in
  structured or layout-sensitive compositions." Precise "move this exactly here" arrows are therefore
  the weakest part of a sketch vocabulary.

## 4. Prior art: annotation and sketch guided editing

- **VIBE: Visual Instruction Benchmark for Image Editing** **[paper]**, Zhang et al., arXiv
  2602.01851 (v2 May 2026). <https://arxiv.org/html/2602.01851>
  - Visual instructions are mostly **drawn directly on the input image** (red boxes and arrows,
    overlaid sketches); the formulation allows them "either as a separate image or as overlaid
    annotations". Pose control uses a separate reference image.
  - Grading treats "residual visual instruction marks such as arrows, boxes, strokes, or masks" as a
    coherence defect, i.e. the burned-in approach must also erase its own marks.
  - 17 models. Nano Banana Pro best (65.15/100); **GPT-image-1 lowest proprietary (44.21)**;
    best open model FLUX2-dev 30.95. GPT-image-1 is weak on arrow-driven tasks (Translation 47.00,
    Reorientation 11.09) and drops from 58.56 to 42.85 when two or more instructions are combined.
    Removal is the easiest task. Newer GPT image models were not tested, so their standing is
    **[unverified]**.
- **Exploring Visual Prompts: Refining Images with Scribbles and Annotations** **[paper]**, Park,
  Eiband, Luckow, Sedlmair, CHI '25 Extended Abstracts, arXiv 2503.03398.
  <https://arxiv.org/html/2503.03398v2> Seven professional designers; simulated outputs. Annotations
  (arrows, circles, numbering, labels) were preferred for spatial changes and for referring to
  elements; scribbles for shape/size/position, usually with a label; text for global or
  multi-attribute changes. Concerns: the model misreading handwriting, arrows or custom symbols, and
  clutter from overlapping marks. Design implication: help the model tell annotation types apart,
  "possibly via color coding or numbering". This directly supports the concept doc's colour legend.
- **Visual Instruction Inversion** **[paper]**, Nguyen et al., arXiv 2307.14331 (2023).
  <https://arxiv.org/abs/2307.14331> Uses a before/after image pair as the visual prompt; argues
  language is ambiguous for specific edits and visual prompts are "more informative and intuitive".
  Older diffusion-era method; relevant as evidence for the two-image framing, not as a recipe.
- **SpotEdit** **[paper]**, Ghazanfari et al. (Adobe/NYU), arXiv 2508.18159.
  <https://arxiv.org/html/2508.18159v2> Edits conditioned on a reference image plus text; finds GPT-4o
  often hallucinates objects not present in the reference (failure rates 81–92% on its robustness
  test). A caution for two-image setups: the model may invent content from the reference image.
- **SketchVLM** **[paper]**, Collins, Bolton, Nguyen et al., arXiv 2604.22875.
  <https://arxiv.org/html/2604.22875> The reverse direction (models *drawing* annotations), but notes
  model-dependent spatial precision; a coordinate grid helped GPT-5 and hurt Gemini-3-Pro.
- **Google Gemini app markup** (first-party, but a different vendor): the December 2025 Gemini Drop
  says "Use your finger or cursor to circle, draw, or annotate directly on an image to tell Gemini
  where to make changes." <https://blog.google/products/gemini/gemini-drop-december-2025> How it
  encodes the marks for the model (burned in vs separate) is not documented **[unverified]**.

## 5. The three options weighed

| Option | For | Against |
|---|---|---|
| **1. Clean frame + sketch-over-frame (two images)** | Model sees unmarked pixels as the base (image 1 gets the mask and extra fidelity); marks never need erasing; maps to OpenAI's documented multi-image indexing | Model must register image 2 onto image 1; possible hallucination from the reference (SpotEdit); more input tokens |
| **2. Sketch burned into one image** | One image, simplest; how VIBE and Gemini markup frame the task | Model must obey *and* erase marks; leftover marks are a known failure mode (VIBE); marked pixels are lost as the base for the edit |
| **3. Sketch → mask** | Tells the model where; reusable downstream as a hard comp matte | Soft on the model side; a mask says *where* not *what*, so colour legend, arrows and labels are lost unless sent another way |
| **1 + 3 + composite back** | Where (mask), what (sketch image + prompt), and a real guarantee outside the edit (comp) | Most moving parts; moves need the mask to cover both source and destination |

## Recommendation for v1

1. Send **image 1 = clean frame**, **image 2 = sketch over the frame** (same size), with a prompt
   template that names them ("Image 1 is the frame to edit. Image 2 is the same frame with
   instructions drawn on it: red = remove, green = add or replace, blue arrow = move to. Apply the
   instructions to Image 1. Change only the marked areas; keep everything else the same. Do not draw
   any of the marks, arrows or labels.") plus the artist's free text.
2. Derive a **mask from the sketch's coverage**, dilated (and for blue arrows, covering both ends),
   sent as the edit mask with alpha = 0 where editing is allowed.
3. **Composite the original back** outside a feathered version of the same mask in Nuke or ComfyUI.
   This, not the API mask, is what protects the plate.
4. Use `input_fidelity="high"` only on models that accept it (`gpt-image-1` / `1.5`); omit it for
   `gpt-image-2` and later until confirmed.
5. Keep the vocabulary small: three colours, arrows, one-or-two-word labels.

## What the first experiment should compare (once ComfyUI access exists)

Fixed set of ~8 frames x 4 edit types (remove, add, replace, move), same sketch per case, same
model (current default `gpt-image-2.5-sunburst`, plus `gpt-image-1.5` with `input_fidelity` high/low
if the ComfyUI node exposes it):

| Variant | Inputs |
|---|---|
| A | one image, sketch burned in, "apply then remove the marks" |
| B | two images (clean, sketch-over-frame), no mask |
| C | B + sketch-derived mask |
| D | clean frame + mask only, legend described in text |
| A–D + comp | each of the above with the original composited back outside a feathered mask |

Score each output on: (1) intent followed (did the right thing happen in the right place),
(2) residual marks visible, (3) drift outside the edit (pixel diff vs original outside the dilated
mask, before comp), (4) colour/grain shift across the frame, (5) cost/latency per call. Run each
case 2–3 times, since outputs vary. Also note whether the ComfyUI node accepts multiple images and a
mask at all (that is ticket 2's question; if it does not, variants B–D need a different node or a
direct API call).
