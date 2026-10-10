# GPT image edit in ComfyUI, driven over the API

Research for issue #3 ("GPT image edit in ComfyUI, driven over the API"). Researched 2026-10-10.

Source snapshot: ComfyUI `master` at commit
[`0df64eb2`](https://github.com/comfyanonymous/ComfyUI/tree/0df64eb242b7c5759c3e86afd5d1846d923b1033)
(committed 2026-10-09). All source links below are pinned to that commit; `B` stands for
`https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033`.

## Short answer

- **There is no separate "edit" node.** GPT image edit is the image-generation node used with one
  or more reference images connected. If any image is connected, the node calls OpenAI's
  `/images/edits` (through Comfy's proxy). With no image, it calls `/images/generations`
  ([B/comfy_api_nodes/nodes_openai.py#L698-L773](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L698-L773)).
- **There are two node classes, and the names are confusing:**
  - `OpenAIGPTImageNodeV2`, display name **"OpenAI GPT Image 2.5"**, is the current node. Its
    `model` choices are `gpt-image-2.5-flare`, `gpt-image-2.5-sunburst`, `gpt-image-2`,
    `gpt-image-1.5` and `gpt-image-1`. It takes up to 16 reference images and an optional mask.
  - `OpenAIGPTImage1`, display name **"OpenAI GPT Image 2"**, is marked **deprecated**. Its
    models are `gpt-image-1`, `gpt-image-1.5` and `gpt-image-2` (default `gpt-image-2`). Its
    single `image` input accepts a batch.

  "GPT2 edit" most likely means one of these with `model = gpt-image-2`. Use the V2 node for new
  work.
- **Inputs:** `prompt`, `model`, `size`, `custom_width`/`custom_height`, `background`,
  `quality`, `images` (`image_1`…`image_16`), `mask`, `n` (1–8) and `seed`. `seed` is not sent to
  OpenAI. `input_fidelity`, `output_format`, `output_compression` and `moderation` are **not
  exposed**; `moderation` is hard-coded to `"low"`.
- **Mask:** a ComfyUI `MASK` in which **white (1.0) means "replace"**. It requires exactly one
  reference image of the same size. The node converts it into the RGBA PNG that OpenAI expects,
  where fully transparent pixels are the ones edited.
- **Over HTTP:** POST the API-format graph to `/prompt` and put a Comfy account API key in
  `extra_data.api_key_comfy_org`. Create the key at https://platform.comfy.org/login. The node
  sends it to `api.comfy.org` as `X-API-KEY`. You cannot use your own OpenAI key. Costs come out
  of prepaid Comfy credits, at **211 credits = $1**.
- **Cost:** `gpt-image-2` at 1024x1024 costs about $0.007 (low), $0.063 (medium) or $0.253 (high)
  per output image. Each reference image adds about $0.010–0.015. These are the node's own price
  estimates.

## 1. Node classes

| `class_type` (what goes in API JSON) | Display name | Status | Models |
|---|---|---|---|
| `OpenAIGPTImageNodeV2` | OpenAI GPT Image 2.5 | current | `gpt-image-2.5-flare`, `gpt-image-2.5-sunburst`, `gpt-image-2`, `gpt-image-1.5`, `gpt-image-1` |
| `OpenAIGPTImage1` | OpenAI GPT Image 2 | `is_deprecated=True` | `gpt-image-1`, `gpt-image-1.5`, `gpt-image-2` (default `gpt-image-2`) |

Sources: the V1 schema is at
[B/comfy_api_nodes/nodes_openai.py#L141-L253](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L141-L253),
with `is_deprecated=True` at L150 and the model combo at L221-L225. The V2 schema is at
[B/comfy_api_nodes/nodes_openai.py#L507-L566](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L507-L566),
and the model list is at L413. Both nodes are in category `partner/image/OpenAI` and have
`is_api_node=True`.

The DALL·E nodes "have been retired and removed from ComfyUI", according to
https://docs.comfy.org/tutorials/partner-nodes/pricing.

OpenAI's edit endpoint also accepts `gpt-image-1-mini`, dated snapshots (`gpt-image-2-2026-04-21`,
`gpt-image-2.5-*-2026-09-08`) and `chatgpt-image-latest`. The ComfyUI nodes do **not** offer
these. The V2 node rejects any model that is not in `GPT_IMAGE_MODELS`
([B/comfy_api_nodes/nodes_openai.py#L695-L697](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L695-L697)).
OpenAI's model list is at
https://developers.openai.com/api/reference/python/resources/images/methods/edit.

On the 2.5 variants, OpenAI recommends Sunburst "for workflows where editing precision matters
most" and Flare for "fast, high-quality everyday image generation"
(https://developers.openai.com/api/docs/guides/image-generation). OpenAI's dated snapshot ID for
2.5 is `-2026-09-08`
(https://developers.openai.com/api/docs/models/gpt-image-2.5-sunburst).

## 2. Inputs of `OpenAIGPTImageNodeV2`

`model` is a **DynamicCombo**: the widgets under it change depending on which model is selected
([B/comfy_api_nodes/nodes_openai.py#L415-L504](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L415-L504)).

| Input | Values | Notes |
|---|---|---|
| `prompt` | string | Required. It is validated as a non-empty string. |
| `model` | see above | The sub-inputs below are keyed under it (see §5). |
| `model.size` | 2.x: `auto`, `1024x1024`, `1024x1536`, `1536x1024`, `2048x2048`, `2048x1152`, `1152x2048`, `3840x2160`, `2160x3840`, `Custom`. 1.x: `auto` and the three 1024-based sizes only. | Default `auto`. |
| `model.custom_width` / `model.custom_height` | 480–3840, step 16 (V1 node: min 1024) | Used only when `size = Custom`. The node checks: both multiples of 16, max edge ≤ 3840, aspect ratio ≤ 3:1, and total pixels between 655,360 and 8,294,400 (L674-L693). |
| `model.background` | 2.5: `auto`/`opaque`/`transparent`. `gpt-image-2`: `auto`/`opaque` only. 1.x: `auto`/`opaque`/`transparent`. | |
| `model.quality` | 2.5: `low`/`medium`/`high`/`xhigh`/`max`. Others: `low`/`medium`/`high`. | Default `low`. |
| `model.images.image_1` … `image_16` | IMAGE (Autogrow, min 0) | Tooltip: "Up to 16 images" (L425-L433). A batched tensor on one slot is split into separate images (L699-L705). |
| `model.mask` | MASK, optional | See §3. |
| `n` | 1–8 | OpenAI allows 1–10; the node caps it at 8. |
| `seed` | 0–2147483647 | Tooltip: "not implemented yet in backend". V2 does not send it (L746-L754). |

The deprecated `OpenAIGPTImage1` has the same inputs as flat widgets (`prompt`, `seed`,
`quality`, `background`, `size`, `n`, `image`, `mask`, `model`, `custom_width`,
`custom_height`). It has one `image` input: a batch of more than one image is sent as `image[]`
(L338-L353). Custom sizes there are only allowed for `gpt-image-2` (L312-L314), and sizes above
1536 are rejected for 1.x models (L330-L332).

**Not exposed by either node:** `input_fidelity`, `output_format` (so OpenAI's default PNG is
used), `output_compression`, `partial_images`/`stream` and `user`. `moderation` is always
`"low"` (L752, L770). The request models do not even have an `input_fidelity` field
([B/comfy_api_nodes/apis/openai.py#L27-L36](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/apis/openai.py#L27-L36)).
That only matters for 1.x models: OpenAI says "For `gpt-image-2`, omit this parameter" and
documents `high`/`low` on `gpt-image-1` and `gpt-image-1.5`
(https://developers.openai.com/api/reference/python/resources/images/methods/edit).

## 3. How the node prepares reference images and masks

From
[B/comfy_api_nodes/nodes_openai.py#L698-L740](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L698-L740):

- Every reference image is downscaled to at most **2048×2048 total pixels** (about 4.19 MP),
  keeping its aspect ratio. Images already within that limit are left alone
  ([`downscale_image_tensor`, B/comfy_api_nodes/util/conversions.py#L165-L176](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/util/conversions.py#L165-L176)).
- Each image is then **re-encoded as PNG** and sent as multipart form data. The field is `image`
  when there is one image and `image[]` when there are several. Because ComfyUI does this
  conversion, OpenAI's own format rule never reaches the caller. That rule is "png, webp, or jpg
  file less than 50MB. You can provide up to 16 images"
  (https://developers.openai.com/api/reference/python/resources/images/methods/edit). Whatever
  you load into ComfyUI is already a tensor.
- **Mask semantics:** a ComfyUI MASK is 1.0 for "replace" (tooltip: "white areas will be
  replaced"). The node builds an RGBA PNG whose alpha channel is `1 - mask` (L729). OpenAI
  defines the mask as "An additional image whose fully transparent areas (e.g. where alpha is
  zero) indicate where `image` should be edited … Must be a valid PNG file, less than 4MB, and
  have the same dimensions as `image`" (same OpenAI page). OpenAI also says: "If there are
  multiple images provided, the mask will be applied on the first image."
- The node is stricter than OpenAI. A mask needs **exactly one** reference image
  ("Cannot use a mask with multiple image"), the mask must have the same H×W as that image, and
  a mask with no image is an error (L672-L673, L722-L726).
- **Practical consequence for API callers:** `LoadImage` takes its MASK output from the PNG's
  alpha channel as `1 - alpha`. If the image has no alpha, it returns a **64×64 zero mask**
  ([B/nodes.py#L1795-L1799](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/nodes.py#L1795-L1799)).
  Uploading a PNG whose transparent pixels mark the edit region and wiring `LoadImage` output 1
  to `model.mask` therefore works. Wiring that output from an image without alpha fails the
  same-size check.
- Whether OpenAI applies the mask exactly or only as guidance for gpt-image-2/2.5 is
  **unverified**. OpenAI's reference page does not say.
- Output: the node decodes every returned image (b64 or URL) to RGBA and stacks them into one
  IMAGE batch. If an `auto` size returns slightly different dimensions, those images are
  resized to match the first (L95-L138).

## 4. Supported sizes and limits

| Constraint | ComfyUI node | OpenAI API |
|---|---|---|
| Preset sizes | listed in §2 | "The standard sizes `1024x1024`, `1536x1024`, and `1024x1536` are supported by the GPT image models" |
| Custom size | 2.x models only. Multiples of 16, edge ≤ 3840, ratio ≤ 3:1, 655,360–8,294,400 px | Arbitrary `WIDTHxHEIGHT`: "divisible by 16 … aspect ratio … between 1:3 and 3:1. Resolutions above `2560x1440` are experimental, and the maximum supported resolution is `3840x2160`" |
| Reference images | ≤ 16 (V2), downscaled to ≤ 4.19 MP, sent as PNG | ≤ 16. png/webp/jpg, each < 50 MB |
| Mask | exactly 1 reference image, same size | PNG < 4 MB, same dimensions, applied to the first image |
| `n` | 1–8 | 1–10 |
| Transparent background | not offered for `gpt-image-2` | "For `gpt-image-2` … this support is in preview". Requires png/webp output |

OpenAI quotes are from
https://developers.openai.com/api/reference/python/resources/images/methods/edit. ComfyUI rules
are from the line ranges in §2–3.

The two sides disagree in one place. ComfyUI's preset list and custom check allow 2160×3840
(portrait) and 8,294,400 px. OpenAI states the maximum as `3840x2160`. Whether OpenAI accepts the
portrait 4K preset is **unverified**.

## 5. Driving it over the ComfyUI HTTP API

### 5.1 Authentication

- **Server side:** `POST /prompt` reads `json_data["extra_data"]`. Before queueing, it moves
  `auth_token_comfy_org` and `api_key_comfy_org` into a separate `sensitive` dict, so they are
  never stored in queue or history
  ([B/server.py#L1179-L1204](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/server.py#L1179-L1204);
  key list in
  [B/execution.py#L162](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/execution.py#L162)).
  At execution time those values become the hidden inputs that both GPT image nodes declare
  ([B/execution.py#L207-L210](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/execution.py#L207-L210);
  node `hidden=[...]` at nodes_openai.py L249-L253 and L563-L567).
- **Outbound:** if `auth_token_comfy_org` is set, the node sends
  `Authorization: Bearer <token>`. Otherwise, if `api_key_comfy_org` is set, it sends
  `X-API-KEY: <key>`
  ([B/comfy_api_nodes/util/_helpers.py#L40-L45](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/util/_helpers.py#L40-L45)).
  The target is `https://api.comfy.org` plus `/proxy/openai/images/edits`. The host can be
  changed with `--comfy-api-base`
  ([B/comfy/cli_args.py#L267-L272](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy/cli_args.py#L267-L272)).
  The browser frontend uses the bearer token from the logged-in session. A script should use
  `api_key_comfy_org`.
- **Repo example:**
  [B/script_examples/basic_api_example.py#L100-L106](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/script_examples/basic_api_example.py#L100-L106)
  shows `p["extra_data"] = {"api_key_comfy_org": "comfyui-87d01e28d…"}`. That suggests keys start
  with `comfyui-`, but the documentation does not formally specify a key format.
- **Docs:** https://docs.comfy.org/development/comfyui-server/api-key-integration says "The API
  key should be added to the `extra_data` field of the payload".
- **Getting a key** (https://docs.comfy.org/development/api-development/getting-an-api-key): log
  in at https://platform.comfy.org/login, then use API Keys → `+ New` → name → `Generate`. "API
  Keys are only shown once upon creation." This key is **not** the Registry publishing key
  (https://docs.comfy.org/tutorials/partner-nodes/overview).
- **No bring-your-own OpenAI key:** "Currently, the Partner Nodes are still in the testing phase
  and do not support this feature yet"
  (https://docs.comfy.org/tutorials/partner-nodes/overview).
- **Errors:** the API returns 401 for a missing or bad key ("Unauthorized: Please login first to
  use this node.") and 402 for too few credits ("Payment Required: Please add credits…")
  ([B/comfy_api_nodes/util/client.py#L742-L745](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/util/client.py#L742-L745)).
  The actual charge comes back in response header `X-Comfy-Credits-Used` (L110).
- **The localhost / `--listen` caveat in the docs:** "Only allow access from `127.0.0.1` or
  `localhost`" and "only API Key login is supported in a LAN environment"
  (https://docs.comfy.org/interface/user). As I read it, this restriction covers **browser
  login** to the frontend, which is whitelisted by origin. A key passed in `extra_data` takes the
  server-side path above, which has no origin check. **Unverified** by a live run.
- `--disable-api-nodes` (deprecated alias of `--offline`), or `--disable-partner-nodes`, removes
  these nodes entirely
  ([B/comfy/cli_args.py#L218-L220](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy/cli_args.py#L218-L220)).

### 5.2 API-format JSON for the V2 node

Inputs nested under a DynamicCombo or Autogrow are flattened into **dot-joined keys**
(`finalize_prefix` joins with `.`,
[B/comfy_api/latest/_io.py#L1085-L1091](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api/latest/_io.py#L1085-L1091)).
The unit tests use the same form, for example `{"mode": "off", "mode.rows.summary": 0.5}`
([B/tests-unit/comfy_api_test/io_dynamic_group_test.py#L113](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/tests-unit/comfy_api_test/io_dynamic_group_test.py#L113)).
The safest way to get exact keys is to export a working graph with "Export (API)". Inferred
shape, **not run live**:

```json
{
  "prompt": {
    "1": {"class_type": "LoadImage", "inputs": {"image": "plate.png"}},
    "2": {"class_type": "OpenAIGPTImageNodeV2", "inputs": {
      "prompt": "Replace the sky with a stormy dusk sky",
      "model": "gpt-image-2",
      "model.size": "auto",
      "model.custom_width": 1024,
      "model.custom_height": 1024,
      "model.background": "auto",
      "model.quality": "medium",
      "model.images.image_1": ["1", 0],
      "model.mask": ["1", 1],
      "n": 1,
      "seed": 0
    }},
    "3": {"class_type": "SaveImage", "inputs": {"images": ["2", 0], "filename_prefix": "gptedit"}}
  },
  "extra_data": {"api_key_comfy_org": "comfyui-…"}
}
```

Upload `plate.png` first with `POST /upload/image`
([B/server.py#L525](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/server.py#L525)).
Leave out `model.mask` for a plain reference-guided edit.

For the deprecated `OpenAIGPTImage1`, the keys are flat: `image`, `mask`, `model`, `quality`
and so on.

## 6. Cost

### 6.1 Comfy credits

Conversion: "211 credits = $1 USD"
(https://docs.comfy.org/development/comfy-router/pricing).

Token rates from https://docs.comfy.org/tutorials/partner-nodes/pricing (credits per 1M tokens):

| Model | Input | Output (image) |
|---|---|---|
| gpt-image-1 | 2110 (≈ $10) | 8440 (≈ $40) |
| gpt-image-1.5 | 1688 (≈ $8) | 6752 (≈ $32) |
| gpt-image-2 | 2025.6 (≈ $9.60) | 7600 (≈ $36.02) |
| gpt-image-2.5-flare / -sunburst | image 2413.84 (≈ $11.44), text 1508.65 (≈ $7.15) | 9051.9 (≈ $42.90) |

The Comfy Router pricing page lists a different, GPU-second rate for `openai/gpt-image-2`
(0.266 credits per GPU-second). That is the Router product, not Partner Nodes; the two pages
disagree on how gpt-image-2 is metered.

The node's own **price badge** (V2,
[B/comfy_api_nodes/nodes_openai.py#L569-L639](https://github.com/comfyanonymous/ComfyUI/blob/0df64eb242b7c5759c3e86afd5d1846d923b1033/comfy_api_nodes/nodes_openai.py#L569-L639))
estimates USD per output image as follows.

| Size | gpt-image-2 low / med / high | gpt-image-2.5 low / med / high / xhigh / max |
|---|---|---|
| 1024x1024 | 0.0071 / 0.0632 / 0.2529 | 0.0084 / 0.0188 / 0.0753 / 0.1339 / 0.3013 |
| 1536x1024, 1024x1536 | 0.0057 / 0.0494 / 0.1976 | 0.0068 / 0.0147 / 0.0589 / 0.1055 / 0.2354 |
| 2048x2048 | 0.0143 / 0.1284 / 0.5138 | 0.0170 / 0.0383 / 0.1531 / 0.2721 / 0.6123 |
| 2048x1152, 1152x2048 | 0.0057 / 0.0509 / 0.2034 | 0.0067 / 0.0157 / 0.0606 / 0.1077 / 0.2424 |
| 3840x2160, 2160x3840 | 0.0134 / 0.1201 / 0.4803 | 0.0159 / 0.0371 / 0.1431 / 0.2544 / 0.5724 |

Each connected reference image adds about $0.0098–0.0147 for gpt-image-2 and $0.0117–0.0176 for
2.5. The total is multiplied by `n`.

For gpt-image-1 at 1.5 the badge shows only ranges per quality, for example gpt-image-1 high
$0.167–0.25. These are estimates. The real charge is metered by tokens on Comfy's side and
reported in `X-Comfy-Credits-Used`.

### 6.2 OpenAI list price (for comparison only; you cannot pay OpenAI directly through these nodes)

From https://developers.openai.com/api/docs/models/gpt-image-2.5-sunburst: gpt-image-2.5-sunburst
Standard pricing is $5.00 text input, $8.00 image input ($2.00 cached) and $30.00 output per 1M
tokens. "Image output costs $30 per million tokens with Standard processing or $15 with Batch
processing."

https://developers.openai.com/api/docs/pricing lists Flare at the same rates. It also lists
`gpt-image-2` at $4.00 image input / $15.00 output, but in a table whose tier label is ambiguous
("All models"). Since 2.5 shows $15 there and $15 is its Batch price, that table is probably
Batch, and gpt-image-2 Standard would then be $8 / $30. **The gpt-image-2 Standard list price is
unverified.**

The gpt-image-2 model page gives no price; it points to the image generation calculator
(https://developers.openai.com/api/docs/models/gpt-image-2). gpt-image-1 and 1.5 are no longer on
the pricing page.

Comparing the two: Comfy's gpt-image-1 and 1.5 token rates equal OpenAI's historic list prices.
For gpt-image-2 and 2.5, Comfy appears to charge about 1.2× and 1.43× respectively over $8 / $30.
That is inferred from the numbers above, not stated anywhere.

## 7. Other notes

- OpenAI rate limits for gpt-image-2 run from Build tier 20 IPM up to Grow tier 250 IPM
  (https://developers.openai.com/api/docs/models/gpt-image-2). Those are OpenAI's limits on
  Comfy's own org. **Comfy-side rate limits are unverified.**
- OpenAI may require organization verification for GPT Image models
  (https://developers.openai.com/api/docs/guides/image-generation). Comfy's proxy handles this,
  so a ComfyUI user does not need to do it.
- Credits are prepaid and non-refundable. Partner Nodes have no free tier. A negative balance
  blocks runs (https://docs.comfy.org/tutorials/partner-nodes/overview).
- This research ran against current `master`. A pinned install, such as Kodiak's ComfyUI
  v0.28.3, may have an older `nodes_openai.py` without the V2 node or the 2.5 models. Check
  `GET /object_info/OpenAIGPTImageNodeV2` on the target server. **Unverified** which release
  first shipped each node.
