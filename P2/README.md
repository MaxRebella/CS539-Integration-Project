# P2 — SAM Leaf Localization

Yu Chen (P2)

**Stage 1 of the pipeline:** a frozen SAM 2.1 Hiera Tiny model segments the tomato leaf. The leaf crop (224 × 224) is then passed to the MobileNetV2 classifier (P3).

## Files
| File | What it is |
|---|---|
| `P2_SAM_Crop_Check.ipynb` | 50-image crop check (25 PlantVillage + 25 PlantDoc) |
| `crop_check.png` | Grid of original \| SAM mask \| crop for all 50 images |

## How to run
- **Colab:** open the notebook and choose a GPU runtime.
- PlantVillage is downloaded from Hugging Face (`mohanty/PlantVillage`, about 2.2 GB). PlantDoc is cloned from `pratikkayal/PlantDoc-Dataset`, and only its test split is used.

## Method
1. Load SAM 2.1 Hiera Tiny (`facebook/sam2.1-hiera-tiny`). It stays frozen, with no training.
2. Prompt SAM with one positive point at the image center and keep the highest-scoring of its 3 masks.
3. Crop a square box around the mask center with 10% padding, kept inside the image, and resize it to 224 × 224. The square box means the crop is never stretched.
4. Check each crop by eye: does it contain the whole leaf?

Proposal rule: ≥ 40/50 pass → keep SAM, otherwise fall back to a center crop.

## Results (by Nov 21, 50-image check)
| Source | Pass | Rate |
|---|---|---|
| PlantVillage (lab photos) | 24 / 25 | 96% |
| PlantDoc (real-world photos) | 18 / 25 | 72% |
| **Total** | **42 / 50** | **84%** |

**Decision: keep SAM.** It passes the rule, but only just.

Failed images (index in `crop_check.png`): 22, 29, 30, 31, 34, 38, 43, 45.

The first run used a rectangular crop box and scored 41/50. #26 was stretched into a blur and failed. The square box fixed that one.

## Problems found and Next Steps
- **SAM picks a lesion or part of the leaf instead of the whole leaf** (#22, #29, #30, #31, #34). The center point often lands on a diseased spot, and SAM segments only that spot. This happens most with blight, which makes it the most serious problem: the crop shows the disease up close but loses the leaf context.
- **Cluttered real-world photos** (#43 whole plant, #45 pile of leaves): there is no single leaf at the center, so the crop is meaningless.
- **Stretched crops** (#26, #38, fixed). The first version resized a thin rectangular box to a square, which distorted the image. It now uses a square box.
- **Bad data in PlantDoc:** #38 is a text-and-photo collage, not a leaf photo. Some PlantDoc images also carry watermarks or copyright text.
- **Lab-vs-real gap:** segmentation is near-perfect on PlantVillage and much weaker on PlantDoc. This matches the proposal's concern and will be reported as a finding.
- **Overlapping leaves** sometimes end up in the crop with the target leaf (#32, #35). These still pass because the target leaf is complete.

## Fixes
- [x] **Square crop box:** no more stretching. It raised the result from 41 to 42 out of 50.
- [ ] **Small-mask fallback:** if the mask covers less than about 5% of the image, use the whole image instead. This should fix most lesion-only crops.
- [ ] Possibly prompt with several points, or choose the largest mask instead of the highest-scoring one.

