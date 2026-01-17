
# Radius-based Spatial Thinning & Merge Scheme

![lod.webp](lod.webp)

## 1. Technical Overview

To address the severe spatial redundancy of 3D Gaussian Splatting (3DGS) data in high-density regions and the grid artifacts and structural aliasing commonly introduced by traditional voxel-based thinning methods, we propose a radius-based spatial thinning and merge scheme.

This method uses a spatial distance constraint as its core criterion. By defining a minimum spatial separation (radius threshold), Gaussian primitives are processed through point-wise selection and merging. The algorithm prioritizes Gaussians with higher visual contribution and suppresses redundant points within their spatial neighborhoods, thereby achieving a more uniform spatial distribution.

To ensure computational efficiency for large-scale datasets, a hash grid is introduced as a spatial acceleration structure. The 3D space is partitioned into regular cells whose edge length equals the radius threshold. Neighbor searches are restricted to the current cell and its adjacent cells only, reducing the computational complexity to near-linear while guaranteeing completeness.

If a representative Gaussian already exists within the radius neighborhood of a given Gaussian, its geometric and attribute parameters are merged into that representative using weighted aggregation. Otherwise, the Gaussian is retained as a new representative. After merging, an upper bound is enforced on the Gaussian scale to prevent abnormal inflation caused by excessive fusion, ensuring rendering stability.

This scheme does not rely on regular voxel boundaries, multi-level LOD hierarchies, or information-theoretic distance metrics. It is well suited for offline thinning and engineering-level optimization of large-scale 3DGS datasets.


## 2. Algorithm Workflow

**Input:**
- Gaussian set
- Radius threshold (r)

**Output：**
- Thinned and merged Gaussian set

**Steps：**

1. **Importance Sorting**  
   Gaussians are sorted according to opacity and local scale to prioritize primitives with higher visual contribution.

2. **Spatial Hash Construction**  
   The 3D space is divided into grid cells with edge length (r), and a hash mapping is built as a spatial index.

3. **Radius Neighborhood Query**  
   For each Gaussian, neighbor search is performed only within its own grid cell and the surrounding 27 neighboring cells.。

4. **Thinning and Merging**
    - If no representative exists within radius(r), the Gaussian is retained as a new representative.
    - If a representative exists, the Gaussian is merged into it using weighted aggregation.

5. **Scale Constraint and Parameter Update**  
   A scale upper bound is applied to merged Gaussians, and the final Gaussian set is output.

## 3. Mathematical Formulation

### 3.1 Gaussian Representation

Let the input Gaussian set be defined as:

$$[\mathcal{G} = \{ g_i \mid g_i = (\mathbf{x}_i, {\Sigma}_i, \mathbf{f}_i, \alpha_i) \}]$$

where:
- $(\mathbf{x}_i \in \mathbb{R}^3 )$ is the Gaussian center
- $({\Sigma}_i)$represents scale or covariance approximation
- $( \mathbf{f}_i)$denotes color or spherical harmonics (SH) features
- $(\alpha_i)$ is the opacity

### 3.2 Radius Constraint

For a Gaussian $(g_i)$，if there exists a representative $(g_j )$ such that:

$$[\|\mathbf{x}_i - \mathbf{x}_j\|_2 \le r]$$

then the two Gaussians are considered spatially redundant and should be merged. Otherwise, $(g_i)$ is preserved as a new representative.

### 3.3 Weight Definition

The merge weight is defined as:

$$
[
w_i = \alpha_i
]
$$

（In practice, this can be extended to $( w_i = \alpha_i \cdot s_i )，where (s_i)$ is a local scale factor.）

---

### 3.4 Position and Feature Merging

**Position merging:**

$$
[
\mathbf{x}_{\text{new}} =
\frac{\sum_i w_i \mathbf{x}_i}{\sum_i w_i}
]
$$

**Feature merging (color / SH):**

$$
[
\mathbf{f}_{\text{new}} =
\frac{\sum_i w_i \mathbf{f}_i}{\sum_i w_i}
]
$$

---

### 3.5 Opacity Fusion

Opacity is merged using a multiplicative model:

$$
[
\alpha_{\text{new}} = 1 - \prod_i (1 - \alpha_i)
]
$$

This formulation preserves consistent occlusion behavior.

---

### 3.6 Scale Merging and Constraint

Position variance estimation:

$$
[
{\sigma}^2_{\text{pos}} =
\mathbb{E}[\mathbf{x}^2] - \mathbf{x}_{\text{new}}^2
]
$$

Scale fusion:

$$
[
{\sigma}^2_{\text{new}} =
{\sigma}^2_{\text{pos}} + \mathbb{E}[{\Sigma}_i^2]
]
$$

An upper bound constraint is applied:

$$
[
{\Sigma}_{\text{new}} \le k \cdot \mathbb{E}[{\Sigma}_i]
]
$$

where $(k)$ is an empirical coefficient used to prevent excessive scale inflation.

---

### 3.7 Hash Grid and 27-Cell Neighborhood

When the grid cell edge length is set to $(r)$, any two points within distance $(r)$ must differ by at most $([-1, 1])$ grid units along each axis.
Therefore, only:

$$
[
3 \times 3 \times 3 = 27
]
$$

neighboring grid cells need to be examined to ensure no valid neighbors are missed.

## 4. Computational Complexity

Let (N) be the number of input Gaussians and (M) the number of representatives after thinning（typically $( M \ll N ) $）。

- **Sorting:**

$$[O(N \log N)] $$

- **Radius-based thinning and merging (with hash grid acceleration):**  
  Each Gaussian checks only a constant number of candidates on average:

$$[O(N)]$$

- **Overall time complexity:**

$$[O(N \log N)]$$

- **Space complexity:**

$$[ O(N)]$$

---

## 5. Advantages and Effects

### 5.1 Visual Quality
- Eliminates grid artifacts introduced by voxel-based thinning
- Suppresses Gaussian stacking and ghosting in high-density regions
- Preserves structural continuity and boundary stability

### 5.2 Engineering and Performance
- Near-linear complexity, suitable for million- to ten-million-scale datasets
- Simple hash grid structure, easy to implement on CPU, WASM, or GPU
- No dependence on covariance decomposition or KL-divergence computation, ensuring numerical stability

### 5.3 General Applicability
- No multi-level LOD hierarchy required; suitable for single-layer model optimization
- Can serve as a preprocessing, compression, or loading optimization module for 3DGS data
- Easily decoupled and integrated with rendering schedulers and streaming systems

## 6. Conclusion

The **radius-based spatial thinning and merge scheme** suppresses redundant Gaussian primitives through spatial distance constraints. Without relying on regular voxel partitioning or information-theoretic metrics, it achieves a high-quality and scalable thinning approach for 3D Gaussian data, balancing visual fidelity with engineering performance and making it well suited for deployment in large-scale 3DGS systems.

Moreover, this scheme naturally serves as a solid foundation for **Level of Detail (LOD)** construction. By progressively increasing the radius threshold \( r \), a hierarchy of Gaussian representations from fine to coarse can be generated while preserving consistent merging rules and stable spatial distributions. Each radius corresponds to a distinct spatial resolution, ensuring a minimum inter-Gaussian distance at every level and avoiding the abrupt transitions and structural discontinuities commonly observed in voxel-based LOD methods.

Because all levels follow the same radius constraint and merging strategy, strong geometric, density, and visual coherence is maintained across LODs. This allows seamless integration with runtime scheduling mechanisms based on view distance, screen-space error (SSE), or other adaptive criteria, enabling smooth, stable, and engineering-friendly multi-level rendering and streaming of 3D Gaussian Splatting data.
