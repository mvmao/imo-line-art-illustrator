---
name: imo-line-art-illustrator
description: "Create IMO black-line, white-background illustrations from a city or scene brief, using finished-artwork images as actual style references and qweapi gpt-image-2. Invoke only for explicit IMO illustration requests."
metadata:
  short-description: "生成 IMO 黑白手绘线稿与城市文化场景主题插画作品"
---

# IMO 黑色线稿插画生成器

This skill is explicit-only. Run it when the user invokes `$imo-line-art-illustrator` or clearly asks to use this skill by name.

## Modes

- **Analyze references**: use only when the user provides a new archive or explicitly asks to update the visual knowledge base. Run `scripts/ingest_sources.py` and `scripts/compile_knowledge.py`, then inspect representative images before marking the guides ready.
- **Generate illustration**: the normal mode. Read the compiled guides and create one or more original illustrations from a city or scene brief. If any guide has `status: "awaiting-source-assets"` or `"pending-source-analysis"`, stop and ask for the reference archive instead of inventing IMO or logo details.

Read these references for generation:

- [references/visual-style-guide.json](references/visual-style-guide.json)
- [references/imo-character-guide.json](references/imo-character-guide.json)
- [references/strawberry-logo-guide.json](references/strawberry-logo-guide.json)
- [references/prompt-constraints.json](references/prompt-constraints.json)
- [references/creative-transformation-guide.json](references/creative-transformation-guide.json)

## Fixed visual language

Every prompt must preserve these invariants:

- White background, black contour line, and only tiny solid-black fills where needed.
- Childlike simple drawing and low-detail cartoon geometry; silhouette and immediate recognition matter more than realistic structure.
- Medium-to-heavy rounded hand-drawn lines with mild wobble, slight width variation, and a few natural openings. Keep the drawing clean: no construction lines, duplicate tracing, fuzz, ink wash, gradients, gray shading, colored blocks, or glossy volumetric light.
- Use circles, ovals, rounded rectangles, trapezoids, triangles, and soft irregular shapes with slight asymmetry. Avoid sterile vector-icon geometry.
- The four stylized identity options are REZ (male astronaut), YUKO (female astronaut), GANS (double-headed goose), and KULI (spherical pufferfish). Their finished-artwork transformations are the primary identity reference. For REZ/YUKO, preserve the finished works' round helmet, broad soft torso, short thick limbs, and oversized gloves/boots; for GANS, preserve one rounded body with two distinct thin goose necks/heads; for KULI, preserve one ball-like pufferfish silhouette with tiny face and sparse fin/protrusion marks. Do not redraw the original 3D/model-sheet forms.
- When explicitly requested, the Strawberry Planet is a strawberry body plus leafy crown plus its characteristic orbital ring. It is a symbol, not a realistic fruit. Otherwise omit it entirely, even if a reference contains one.
- The scene should feel friendly, absurd, casually artistic, fashion-aware, and adult-humorous, with low seriousness and clear readability. Default to a character-led half-body or full-body vignette: the character occupies most of the frame, one to three related props create the joke, and the environment is only a few atmospheric lines.

Do not turn the style into a polished mascot sheet, realistic illustration, manga rendering, 3D render, generic astronaut, or random messy sketch.

## Request workflow

1. Classify the brief as a named city, a concrete scene, or both.
2. For a city, derive 3–6 grounded cultural signals from everyday behavior, local objects, food or work rhythms, public space, and social relationships. Use landmarks only as secondary context; avoid stereotypes and one-landmark shorthand.
3. Choose the primary stylized identity (REZ, YUKO, GANS, or KULI) before inventing the scene. If the user does not name one, choose the identity that makes the theme's prop relationship clearest; do not invent a generic astronaut or animal.
4. Read `references/creative-transformation-guide.json` and choose one transformation mode. Identify the subject, action, one to three related props, spatial relationship, and one comic contradiction or anthropomorphic beat. Prefer body-as-object, role swap, prop-as-character, ritual inversion, or material transformation over a literal background scene. Keep the character at roughly 55-80% of the frame and reduce the environment to a few lines.
5. Read the relevant compiled guides. Keep the style block and finished-artwork-derived identity block stable; add the Strawberry Planet block only for an explicit strawberry request. Vary only the action, props, compact atmospheric marks, and information-bearing elements.
6. Run [scripts/select_reference_artworks.py](scripts/select_reference_artworks.py) with the complete brief. It selects six finished-artwork references by default (choose between four and eight when the theme needs fewer or more), preferring character-led finished works. Never use raw character sheets or logos as substitutes for finished-artwork style evidence. Use the returned `style_features_to_transfer`, `content_features_to_exclude`, and `new_scene_features` as separate prompt sections.
7. Build a structured prompt with: selected stylized identity, half/full-body proportion, character-led composition, chosen transformation mode, action, one to three related props, minimal atmospheric marks, fixed black-line style, required IMO mark for visible astronauts, IP fidelity constraints, and explicit negative constraints. Treat uploaded finished artworks as authoritative for the transformed character appearance, contour behavior, simplification, spacing, density, pose language, visual gag structure, and hand-drawn finish. Do not copy unrelated objects, landmarks, wording, logos, brands, scene arrangements, or character actions from any reference; all such content must be replaced by the new brief. Do not ask the image model to typeset long Chinese copy. Preserve only short, necessary marks such as the small integrated IMO mark or a simple English label.
8. Apply the Strawberry rule before prompting: do not include Strawberry Logo, Strawberry Planet, or strawberry motifs unless the user explicitly requests strawberry/logo/planet integration. A strawberry visible in a reference is not permission to copy it. When explicitly requested, include the local logo reference image if the endpoint accepts multiple inputs; if exact fidelity is required and the endpoint cannot preserve it, use `scripts/compose_logo.py` after generation and disclose the deterministic composite.
9. Generate through [scripts/qweapi_image_gen.py](scripts/qweapi_image_gen.py), passing the selected finished-artwork paths via repeated `--reference-image` flags. The finished artworks, not raw character sheets, carry the stylized identity. The script uses qweapi `POST /images/edits` with `gpt-image-2`, so the references are sent to the model rather than merely named in text. If credentials, reference files, image-edit support, or the API request fail, report the blocker and do not use `/images/generations`, another model, or a local placeholder.
10. Save outputs under the current workspace at `outputs/imo-line-art-illustrator/<slug>/`. Pass each selected local reference through `--reference-asset-id` and keep the selected identity, half/full-body composition, reference count, style/content separation, Strawberry decision, output paths, and asset IDs in the `--metadata` sidecar JSON, never the key.
11. Run `scripts/inspect_line_art.py` on every generated PNG. A `revise` result requires visual inspection and regeneration or correction; automated metrics do not replace checking the selected stylized identity, body proportion, humor, logo structure when requested, or semantic accuracy.

## Backend and credentials

The only raster backend is:

```text
POST https://qweapi.com/v1/images/edits
model: gpt-image-2
```

`images/edits` is mandatory for ordinary illustration generation because it carries the finished-artwork references. A request without `--reference-image` is invalid by design. Never fall back to `images/generations` when reference-conditioned editing is unavailable.

The script reads `IMO_IMAGE_API_KEY` first, then the macOS Keychain item named by `IMO_IMAGE_KEYCHAIN_SERVICE` (default `imo-line-art-qweapi`). `IMO_IMAGE_BASE_URL` may override the base URL for testing, but the default remains qweapi. Never put an API key in this skill, prompts, JSON, logs, or generated deliverables. The key previously pasted in chat is treated as exposed and should be rotated before configuration.

Defaults are PNG, white background, `1024x1024`, high quality, one image. Accept only valid positive dimensions that are multiples of 16 and within the gpt-image-2 limits; allow portrait or landscape sizes through the script flags.

## Reference archive contract

The archive may use arbitrary folder names, but files are classified by the first matching path or filename token:

- finished artwork tokens: `finished`, `artwork`, `master`, `poster`, `lineart`, `完成`, `成品`, `插画`
- IMO tokens: `imo`, `character`, `astronaut`, `运营`, `联名`
- logo tokens: `logo`, `strawberry`, `草莓`

Ambiguous files remain `needs-review`; do not silently classify them as style or IP evidence. `scripts/ingest_sources.py` rejects unsafe archive paths, records SHA-256 hashes, and never overwrites an existing reference file without `--force`.

## Validation

Run `scripts/validate_knowledge.py` after reference compilation and `/Users/yt/.codex/skills/.system/skill-creator/scripts/quick_validate.py /Users/yt/.codex/skills/imo-line-art-illustrator` after edits. Use `--dry-run` on the image script with at least one real `--reference-image` to verify the `/images/edits` endpoint, fixed model, multipart reference payload, and output path without reading credentials. Visual acceptance still requires comparing the output directly against the selected finished artworks for line weight, simplification, composition, whitespace, density, pose language, gag construction, white background, IMO recognizability, requested Strawberry Planet structure (if any), and semantic accuracy.
