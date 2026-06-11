# Graph-Structure-Only Recovery of a Conserved 161-Neuron GABAergic Circuit Across Three *Drosophila* Connectomes

---

## Introduction

The FlyWire Qualification Challenge requires identification of the largest weakly connected directed induced subgraph that is mutually isomorphic across at least three of five provided connectomic datasets. A valid solution must simultaneously satisfy one-to-one neuron correspondence, conserved directed connectivity, exact directed graph isomorphism, and weak connectivity of the recovered subgraph across all matched datasets.

The five provided connectomes — BANC, FAFB, MANC, MAOL, and MCNS — collectively contain **492,599 neurons** and **24,438,738 directed synaptic connections**. At this scale, brute-force graph matching or exhaustive isomorphism enumeration is computationally infeasible. The datasets further complicate direct comparison by exhibiting substantial structural heterogeneity: graph density varies by nearly 50-fold across datasets (FAFB density 0.000194 vs. MANC density 0.009493), and reciprocity differs by a factor of two between sparse and dense connectome families.

Standard approaches — full VF2 matching, SCC decomposition, or degree-histogram alignment — are either computationally intractable or insufficiently discriminative at this scale. Strongly connected component analysis confirmed this: every connectome consists of one enormous SCC plus thousands of tiny peripheral SCCs with virtually nothing in between, providing no meaningful reduction in search complexity.

We developed a ten-phase hierarchical pipeline that resolves these challenges through progressive structural filtering. The pipeline combines per-neuron topology fingerprinting, three-iteration directed Weisfeiler-Lehman (WL) refinement, robust mutual nearest-neighbor candidate generation, graph-consistent triplet seed formation, greedy conserved circuit expansion, and exact directed-graph isomorphism verification. **Biological annotations, cell-type labels, neurotransmitter identity, brain-region assignments, and morphological information were not used at any stage of the matching process.**

---

## Methods

### Data and Structural Characterization

Five connectomes were analyzed as directed graphs.

| Dataset | Nodes | Edges | Density | Avg Degree | Reciprocity |
|---------|------:|------:|--------:|-----------:|------------:|
| BANC | 112,885 | 2,676,592 | 0.000210 | 23.7 | 14.3% |
| FAFB | 138,584 | 3,732,460 | 0.000194 | 26.9 | 16.6% |
| MANC | 23,641 | 5,305,638 | 0.009493 | 224.4 | 30.4% |
| MAOL | 51,669 | 6,484,936 | 0.002429 | 125.5 | 28.2% |
| MCNS | 165,820 | 6,239,112 | 0.000227 | 37.6 | 15.7% |

All datasets were exceptionally clean: zero missing values, zero duplicate edges, zero isolated neurons across all five connectomes. Self-loops (MANC: 36, MAOL: 263, MCNS: 18) represent less than 0.01% of all edges.

Datasets naturally separate into two structural families. The **sparse family** (BANC, FAFB, MCNS) exhibits density ≈ 0.0002, average degree 24–38, reciprocity 14–17%, and smaller two-hop neighborhoods (average 2,943–6,875 neurons). The **dense family** (MANC, MAOL) exhibits density 0.002–0.009, average degree 125–224, reciprocity 28–30%, and two-hop neighborhoods reaching 15,998–25,464 neurons on average. This divergence ruled out raw degree-based cross-dataset comparison and motivated normalized, topology-aware fingerprinting.

Hub neuron analysis revealed extreme connectivity concentration. MAOL neuron **10009** has an in-degree of 11,496 — the most connected neuron in the dataset — making it an ideal structural anchor for graph alignment. All connectomes were dominated by a single giant weakly connected component (WCC coverage: BANC 99.99%, FAFB 98.85%, MANC 100%, MAOL 99.68%, MCNS 100%).

### Phase 4 — Feature Engineering

Each neuron was represented by five topology descriptors:

- **Total degree** — combined in-degree and out-degree
- **Reciprocal ratio** — fraction of neighbors with bidirectional connections
- **Hub-neighbor count** — number of structurally dominant neighbors; mean varies from 5.0 (BANC) to 23.3 (MANC), providing strong cross-dataset discrimination
- **Two-hop neighborhood size** — neurons reachable in two directed steps; MAOL neurons reach ≈25,464 nodes on average
- **PageRank** — global centrality; retained at low weight as it largely mirrors degree for hub neurons

### Phase 5 — Weisfeiler-Lehman Structural Encoding

WL₀ initialization discretized each Phase 4 feature into quantile bins (20 bins for degree and two-hop size; 10 bins for reciprocal ratio and hub count). Each neuron's bin-tuple became its initial color label.

Three iterations of directed WL refinement were applied. At each iteration, each neuron's label was replaced with a hash of its current label, the sorted multiset of predecessor labels, and the sorted multiset of successor labels. This incorporates increasingly large structural context: WL₁ captures one-hop neighborhoods, WL₂ two-hop, WL₃ approximately three-hop structure.

| Dataset | WL₀ Unique | WL₃ Unique | Nodes | Entropy WL₀→WL₃ |
|---------|----------:|----------:|------:|----------------:|
| MANC | 5,185 | 23,640 | 23,641 | 6.82 → 14.53 bits |
| MAOL | 5,471 | 51,660 | 51,669 | 3.56 → 15.66 bits |
| BANC | 1 | 112,574 | 112,885 | 0 → ~16 bits |
| FAFB | 1 | 135,313 | 138,584 | 0 → 16.99 bits |
| MCNS | 12,928 | 164,840 | 165,820 | 9.27 → 17.32 bits |

BANC and FAFB began with every neuron sharing a single color (WL₀ unique = 1) and reached near-complete discrimination by WL₃ — demonstrating that directed graph topology alone contains sufficient information to distinguish nearly every neuron. WL refinement converged by iteration 2–3; additional iterations provide negligible gain.

**Critical constraint:** Raw WL integer labels are generated independently within each dataset and are not cross-dataset comparable. Label rarity `(−log(frequency))` was used for matching; raw label integers were discarded.

### Phases 6–7 — Feature Construction and Candidate Matching

The final feature vector per neuron extended Phase 4 descriptors with:

- **WL rarity features:** `rarity_wl_k = −log(freq_k)` for k ∈ {0, 1, 2, 3}, downweighted ×0.5 to prevent WL dominance
- **Local directionality:** in-degree, out-degree, reciprocal edge count, successor ratio, predecessor ratio

This produced a **14-dimensional structural feature vector** per neuron. **RobustScaler** normalization (median/IQR) was applied per dataset to handle hub-induced outliers without distortion.

**Mutual cosine KNN matching** (K = 50) identified candidate correspondences for every dataset pair. Forward and reverse KNN indices were built independently. A pair was flagged as mutually matched if A appeared in B's top-50 *and* B appeared in A's top-50. Mutual pairs scored `forward_dist + reverse_dist`; non-mutual pairs penalized with `+1.0`.

**Triplet generation** merged pairwise matches through a pivot neuron, producing `(neuron_A, neuron_B, neuron_C)` spanning three datasets. A-C consistency validation checked whether a direct A↔C candidate existed; triplets with A-C support received a −0.25 bonus, those without were retained (soft penalty preserves recall):

```
triplet_score = dist(A,B) + dist(B,C) + 0.5×dist(A,C) − 0.25×(ac_supported)
```

Up to 10 candidates per anchor and 10 triplets per node were retained to prevent memory explosion.

### Phases 8–10 — Circuit Search, Expansion, and Exact Verification

**Phase 8** performed multi-start greedy circuit search across candidate triplet families (500 seeds per family). Tested families and initial best circuit sizes:

| Family | Best Circuit Size |
|--------|------------------:|
| MAOL–FAFB–MCNS | **126** |
| BANC–FAFB–MCNS | 44 |
| MANC–FAFB–MCNS | 6 |
| MANC–MAOL–MCNS | 9 |
| MAOL–BANC–MCNS | 5 |

The winning family was **MAOL–FAFB–MCNS**. Seeds were sorted by `triplet_score` (best first). Circuit growth followed three strict rules: (1) **strict bijection** — no neuron ID reused across any dataset in the current mapping; (2) **conserved connectivity** — for every accepted triplet `(a,b,c)` and candidate `(a′,b′,c′)`, edge presence/absence of `a→a′`, `a′→a`, `b→b′`, `b′→b`, `c→c′`, `c′→c` must be identical across all three graphs; (3) **minimum conserved-edge participation** — the candidate must share at least one conserved edge with the existing circuit.

**Phase 9** performed second-pass expansion: all remaining unused triplets were scanned and valid additions incorporated, adding **35 neurons** to the initial 126. Full O(N²) pairwise re-verification checked every mapped pair in both directions to guarantee correctness.

**Phase 10** applied the strongest validation: NetworkX `DiGraphMatcher` exact directed-graph isomorphism on induced subgraphs from all three datasets, plus `nx.is_weakly_connected()` confirmation.

---

## Results

The largest conserved circuit was recovered across **MAOL, FAFB, and MCNS**.

| Metric | Value |
|---|---|
| Conserved neurons | **161** |
| Conserved edges | **301** |
| Datasets | MAOL, FAFB, MCNS |
| Weakly connected | Yes |
| Connected components | 1 |
| Circuit density | 0.011685 |
| Average degree (circuit) | 1.87 |
| Maximum in-degree (circuit) | 153 |
| Maximum out-degree (circuit) | 148 |
| Exact isomorphism MAOL ↔ FAFB | **Verified** |
| Exact isomorphism MAOL ↔ MCNS | **Verified** |

Induced subgraph edge counts are identical across all three datasets (MAOL = FAFB = MCNS = **301 edges**), confirming perfect conservation of directed connectivity. The bijection check confirmed 161 unique neurons per dataset with no reuse. Both directed-graph isomorphism tests returned `True`.

![Figure 1](figures/networkdiagram.jpeg)

*Figure 1. Conserved 161-neuron circuit recovered across MAOL, FAFB, and MCNS.*

---

## Biological Interpretation

After circuit recovery, the most connected neuron within the induced subgraph was identified in each dataset and cross-referenced against published biological annotations. The recovered circuit exhibits a strongly hub-centered architecture: the majority of conserved connectivity is organized around a single dominant neuron, which functions as the structural anchor of the entire subgraph.

| Dataset | Neuron ID | Cell Type | Neurotransmitter |
|---|---|---|---|
| MAOL | 10009 | CT1 | GABA |
| FAFB | 720575940628908548 | CT1 | GABA |
| MCNS | 10157 | CT1 | GABA |

**The graph-matching algorithm independently recovered CT1 neurons across three separately reconstructed connectomes.** This correspondence emerged entirely from graph structure: no cell-type information, neurotransmitter identity, morphology, spatial coordinates, or brain-region annotations were used during matching.

CT1 is classified as an **optic-lobe intrinsic neuron** of the lobula-medulla amacrine class, associated with inhibitory GABAergic signaling. CT1 is believed to participate in inhibitory visual-information processing within the lobula–medulla pathway, where widespread GABAergic connectivity may contribute to modulation and integration of visual signals. In MAOL, CT1 (neuron 10009) exhibits approximately **11,496 incoming** and **9,619 outgoing** synaptic connections — one of the most structurally distinctive neurons in the entire dataset, with a maximum in-degree exceeded by no other neuron. Its connectivity signature is sufficiently unique that WL refinement assigns it a near-singleton structural label, making it a highly reliable anchor for cross-dataset alignment.

The independent recovery of CT1 across three separately reconstructed connectomes — MAOL, FAFB, and MCNS — provides strong external validation that the recovered mapping reflects homologous neuronal identities rather than arbitrary structural coincidences. **The recovered correspondence is biologically consistent** with established CT1 identity.

![Figure 2](figures/3d_diagram.jpeg)

*Figure 2. Three-dimensional morphology of CT1 (MAOL neuron 10009), the central hub neuron of the conserved circuit.*

### Hypothesis

The recovered circuit may represent a structurally conserved inhibitory visual-processing motif organized around CT1 neurons. Because CT1 was independently recovered across MAOL, FAFB, and MCNS using graph topology alone, we hypothesize that its large-scale connectivity pattern represents one of the most structurally conserved motifs within the Drosophila optic lobe.

---

## Discussion

This study demonstrates that exact graph-matching techniques can identify biologically meaningful conserved circuits across independently reconstructed connectomes. Although the underlying datasets differ substantially in scale (23,641–165,820 nodes), density (50-fold variation), reciprocity (14–30%), and local topology, the hierarchical pipeline successfully recovered a conserved 161-neuron circuit.

WL refinement on directed graph topology alone provides near-unique neuron identification without any biological annotation, confirming that structural connectivity signatures carry sufficient discriminative information for cross-dataset alignment. The result is consistent with the theoretical properties of WL refinement: in graphs with sufficient local structural diversity, three iterations are enough to nearly perfectly distinguish all nodes.

The recovered circuit is dominated by a CT1-centered hub architecture, suggesting that maximizing induced subgraph size under strict isomorphism constraints naturally favors circuits organized around highly conserved, structurally distinctive hub neurons. The biological consistency of the CT1 correspondence across MAOL, FAFB, and MCNS further supports the validity of the discovered circuit.

### Limitations

The reported 161-neuron circuit should be interpreted as a lower bound on the size of the true conserved circuit. The search procedure relies on candidate pruning and greedy multi-start expansion and therefore is not guaranteed to recover the globally optimal maximum common induced subgraph.

No claim of neuronal homology is made beyond biological consistency. No claim of novel biological discovery is made. The results demonstrate that the recovered structural correspondence is exact and that it aligns with established biological annotations. More broadly, these results demonstrate that graph topology alone can recover biologically meaningful neuron correspondences across independently reconstructed connectomes without requiring cell-type annotations, morphology, anatomical information, or neurotransmitter metadata.

---

## References

1. Takemura S.Y. et al. (2017). A connectome of a learning and memory center in the adult *Drosophila* brain. *eLife.*
2. FlyWire Codex Database. https://codex.flywire.ai
3. Weisfeiler, B. and Lehman, A. (1968). A reduction of a graph to a canonical form and an algebra arising during this reduction. *Nauchno-Technicheskaya Informatsia.*
4. NetworkX Developers. NetworkX: Network Analysis in Python. https://networkx.org
