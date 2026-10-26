# Evidence-Coupled Multiclass Inference for Training-Free Remote Sensing Segmentation

**Prototype Evidence and Edge Relations (PEER)** is a training-free framework for few-shot remote-sensing semantic segmentation. It couples support-conditioned class evidence with relations between neighboring query patches through a shared label distribution: presence-gated prototype adaptation and reciprocal support–query correspondences form a fixed semantic prior that constrains graph propagation, while optional auxiliary observations (nDSM, SAR, NIR, relative depth, thermal) contribute support-estimated histogram likelihoods. The frozen DINOv3 encoder is never fine-tuned, and no task-specific training or annotations beyond the support masks are required.

## Authors

Yongqi Sun, Chenguang Dai, Yujun Quan, Yu Su, Yanfei Zhong (Senior Member, IEEE), Anzhu Yu\*

\*Corresponding author: Anzhu Yu (`anzhu_yu@126.com`).

## Abstract

Frozen vision foundation models can segment remote sensing imagery from a few annotated examples without task-specific training. However, acquisition and scale variations change land-cover appearance, limiting the transfer of fixed reference prototypes. Limited spatial resolution can merge adjacent land-cover regions, while viewing-angle differences can obscure building outlines. These ambiguities can lead to incorrect class assignments, including predictions of classes absent from the scene. To address these issues, we propose PEER, a training-free framework that couples query-adapted prototype evidence with relations between neighboring image patches. An image-level gate controls prototype updates from confident query regions, while reciprocal support–query correspondences retain local appearances that class averaging can discard. Optional auxiliary observations provide class-conditional histogram likelihoods, with distributions and fusion weights estimated from the same support annotations. The resulting semantic prior constrains graph propagation, encouraging agreement between appearance-compatible neighbors while retaining class evidence at each step. Evaluations across remote sensing benchmarks, including cross-domain transfer, demonstrate the effectiveness of the framework under a shared inference configuration. The approach supports optical-only and auxiliary-assisted land-cover mapping from limited annotations without task-specific model training.

## Method at a Glance

- **Dual descriptors from frozen DINOv3** (ViT-L/16, last four blocks, four-rotation aggregation): positionally cleaned descriptors `f^d` (support and query) drive all cross-image evidence; original descriptors `f^{r,q}` define eight-neighbor affinities within the query graph. The positional basis (rank 500) is estimated once from a zero-valued probe via SVD and shared by all images.
- **Presence-gated prototype adaptation** (Eq. 3): a class prototype is updated with confident query features only when its image-averaged probability passes a gate, keeping absent classes anchored to their support prototypes.
- **Semantic prior** (Eq. 7): prototype/auxiliary class probabilities `p_i(c)` are averaged with mutual-nearest-neighbor correspondence evidence `u_i(c)` into a fixed prior `R`.
- **Evidence-guided propagation** (Eqs. 8–10): standard label propagation on the query graph with `R` re-injected at every iteration, followed by upsampling and boundary-CRF refinement.
- **Auxiliary fusion without training** (Eq. 4): class-conditional histograms and the fusion weight `w_m` are both selected using support annotations alone.

## Main Results on Five Remote Sensing Settings

All support-based methods use the same five binary masks per class. `Aux.` lists the inference-time auxiliary field. Bold marks the best mIoU among training-free methods in each setting. Values are mIoU (%).

| Setting | Aux. | SegEarth-OV3 (zero-shot) | INSID3 (5-shot) | FSSDINO (ViT-L/16) | PEER w/o aux. | **PEER (full)** |
|---|---|---:|---:|---:|---:|---:|
| Potsdam | nDSM | 57.84 | 59.35 | 60.79 | 63.83 | **64.09** |
| Vaihingen | nDSM | 60.77 | 65.98 | 65.42 | 69.50 | **70.05** |
| SUSAN | Rel. depth | 26.78 | 27.99 | 20.80 | 28.50 | **28.54** |
| WHU OPT–SAR | SAR | 15.23 | 41.17 | 36.56 | 43.61 | **43.85** |
| WHU OPT–NIR | NIR | 15.23 | 41.17 | 36.56 | 43.61 | **43.86** |

PEER achieves the highest mIoU among the compared training-free methods in all five settings, exceeding matched five-shot INSID3 by 0.55–4.74 points and matched-weight FSSDINO (ViT-L/16) by 3.04–7.70 points. A fully supervised dual ConvNeXt + UPerNet reference remains stronger (e.g., 79.68 on Potsdam), while auxiliary observations add up to +0.55 points, with the largest gains from ground-referenced nDSM.

## Cross-Domain Transfer on LoveDA

Optical-only results on the official LoveDA evaluation server with five source-domain masks per class (U2R: urban support → rural queries; R2U: rural support → urban queries). AT/ST/DG/TF denote adversarial training, self-training, domain generalization, and training-free regimes. Values are mIoU (%).

| Method | Regime | U2R | R2U |
|---|---|---:|---:|
| DAFormer | ST | 46.25 | 48.99 |
| HRDA | ST | 49.08 | 47.31 |
| MIC | ST | 49.69 | 49.31 |
| DG-CroSeg (best trained) | DG | **50.53** | 50.95 |
| SegEarth-OV3 | TF | 44.22 | 40.97 |
| INSID3, five-shot | TF | 44.53 | 51.10 |
| **PEER, five-shot** | TF | 45.94 | **52.99** |

On Rural-to-Urban, PEER (52.99) outperforms all compared trained cross-domain methods without any task-specific training. On Urban-to-Rural it improves over matched INSID3 by +1.41 points but remains below DG-CroSeg, partly reflecting limited rural appearance coverage in the urban support set.

## Transfer to RGB–Thermal Benchmarks

Matched five-shot inference with the shared RS configuration on MFNet and FMB (thermal weight selected from support annotations alone: 0.5 on MFNet, 0 on FMB). Values are mIoU (%).

| Dataset | INSID3 (5-shot) | PEER | Gain |
|---|---:|---:|---:|
| MFNet (RGB + thermal) | 35.04 | **38.49** | +3.45 |
| FMB (optical-only, w_m = 0) | 49.61 | **52.18** | +2.57 |

## Ablations and Analyses

- **Component gains** (Vaihingen / WHU-SAR / WHU-NIR, mIoU points): rotation averaging +0.47/+0.55/+0.61; presence gate vs. ungated update +2.26/+0.42/+0.47; MNN evidence +0.75/+0.59/+0.61.
- **Positional cleaning**: removing the estimated positional subspace raises 1-NN class agreement of cross-image matches from 59.39% to 81.62% on held-out Vaihingen tiles.
- **Support budget** (Potsdam, nested sets): 64.09 → 71.33 → 73.67 mIoU as support grows from 5 to 10 to 50 masks per class, with the gain from MNN evidence and graph propagation rising from +0.23 to +1.86.
- **Resolution degradation** (Potsdam, joint RGB + nDSM): mIoU decreases from 64.09 at 5 cm GSD to 24.83 at 80 cm equivalent, identifying retained spatial detail as a limit of inference from frozen features.
- **Boundary refinement**: the boundary CRF improves mIoU (+0.44/+0.11 on held-out Vaihingen/WHU), whereas a joint mean-field CRF after graph propagation severely oversmooths (−13.94/−8.12).

## Code Availability

Code will be made publicly available upon publication.

## Citation

If you find this work useful, please cite:

```bibtex
@article{sun2026peer,
  title   = {Evidence-Coupled Multiclass Inference for Training-Free
             Remote Sensing Segmentation},
  author  = {Sun, Yongqi and Dai, Chenguang and Quan, Yujun and Su, Yu and
             Zhong, Yanfei and Yu, Anzhu},
  journal = {IEEE Transactions on Pattern Analysis and Machine Intelligence},
  year    = {2026}
}
```

The volume, number, page range, and DOI will be added once the paper is published.

