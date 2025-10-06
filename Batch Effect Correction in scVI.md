# Batch Effect Correction in scVI: Deep Dive

## What Are Batch Effects?

Batch effects are systematic technical differences between groups of cells that were processed separately. For example:
- Cells sequenced on Monday vs. Friday
- Different technicians handling samples
- Different reagent batches
- Different sequencing machines

These create artificial patterns that look like biological differences but aren't real.

## The Core Mathematical Strategy

scVI uses **conditional independence** to separate batch effects from biology. Here's the key insight:

**If you know a cell's true biological state, the batch it came from shouldn't matter for predicting its gene expression.**

Mathematically: `p(expression | biology, batch) ≈ p(expression | biology)`

But we don't know the true biological state - we have to learn it!

## How scVI Implements This

### 1. **The Latent Space Strategy**

scVI creates two types of variables:

**Biological representation (z)**: 
- A 10-dimensional vector that captures what the cell actually is
- Should be the same regardless of which batch the cell came from
- This is what you want to preserve

**Batch label (s)**:
- A categorical variable (Batch 1, Batch 2, etc.)
- Fed into the model but separated from z

### 2. **The Neural Network Architecture**

Here's where the magic happens. scVI has two neural networks that reconstruct gene expression:

```
Input → Neural Network → Gene Expression Parameters

The input is: [z (biology), s (batch)]
```

**The critical design choice**: The networks learn batch-specific transformations.

Let me break down what `f_w(z, s)` and `f_h(z, s)` do:

**f_w(z, s)**: Predicts the expected frequency of each gene
- Takes both biology (z) and batch (s) as input
- Can learn: "In batch 1, gene X is systematically detected 20% less often"
- Outputs corrected frequencies that represent what you'd see without batch effects

**f_h(z, s)**: Predicts dropout probability (technical zeros)
- Takes both biology (z) and batch (s) as input  
- Can learn: "Batch 2 has worse capture efficiency, so more false zeros"

### 3. **The Training Process**

During training, scVI learns:

**For the encoder (going from data to latent space)**:
- How to map observed gene counts → biological representation (z)
- This should extract biology while ignoring batch

**For the decoder (going from latent space to predictions)**:
- How each batch systematically distorts the measurements
- Batch-specific parameters for mean expression levels
- Batch-specific dropout rates

### 4. **Conditional Independence Assumption**

The key assumption is:

```
p(x | z, ℓ, s) = p(x | z, ℓ, s)
```

This means: once you know the biological state (z) and library size (ℓ), the batch label (s) only affects how the signal gets distorted during measurement, not what the true signal is.

## Detailed Mechanism: How Batch Information Flows

### During Encoding (Observation → Latent Space):

1. Cell from Batch 1 with gene counts enters
2. Encoder network receives: `[gene_counts, batch_1_label]`
3. Network learns: "This pattern + batch 1 effects = biological state z"
4. Output: z (batch information removed from this representation)

### During Decoding (Latent Space → Prediction):

1. Biological state z is combined with batch label
2. Decoder learns: "For this biology (z) in batch 1, expect these distortions"
3. Outputs batch-specific predictions

### The Correction Happens Because:

When you want batch-corrected data, you:
1. Take the latent representation z (which has no batch info)
2. Decode it with a "reference" batch or average across batches
3. Get expression values as if all cells came from the same batch

## Mathematical Detail: The ZINB Parameters

For each gene g in cell n from batch s:

**Mean expression**: `μ_ng = ℓ_n × f_w(z_n, s_n)^g`

- ℓ_n: library size (sequencing depth)
- f_w(z_n, s_n)^g: frequency of gene g, **batch-corrected**
- The neural network f_w has learned how batch s affects gene g's detection

**Dropout probability**: `π_ng = f_h(z_n, s_n)^g`

- Different batches have different technical failure rates
- f_h learns these batch-specific dropout patterns

**The batch-corrected expression** is: `ρ_ng = f_w(z_n, s_n)^g`

This ρ matrix is what they use for downstream analysis - it represents gene frequencies **after removing batch effects**.

## Why This Works Better Than Alternatives

### ComBat (Linear Correction):
- Assumes batch effects are additive/multiplicative shifts
- Can't handle complex, gene-specific batch patterns
- Applied after the fact

### scVI:
- Learns non-linear, gene-specific batch effects
- Integrated into the probability model
- Can handle dropout (zeros) differently per batch

### Mutual Nearest Neighbors (MNN):
- Finds similar cells across batches and aligns them
- Can fail if batches don't share all cell types
- Doesn't model the generative process

### scVI's advantage:
- Models the actual data generation process
- Batch effects are "baked into" the statistical model
- Can extrapolate to unseen batch combinations

## The RETINA Dataset Example

In the paper, they tested on retinal bipolar neurons from 2 batches:

**Without batch correction**:
- Cells clustered by batch first, biology second
- Low "entropy of batch mixing" (batches stayed separate)

**With scVI**:
- Cells from both batches mixed in latent space
- Clusters represented cell types, not batch origin
- High entropy of batch mixing (good!)

The metric they used: In each cell's 50 nearest neighbors, how mixed are the batches? Random mixing = high entropy = good correction.

## Limitations and Assumptions

**Assumes shared biology**: Both batches must contain similar cell types. If Batch 1 has cell type A and Batch 2 doesn't, scVI can't tell if that's batch effect or biology.

**Conditional independence**: Assumes batch only affects measurement, not true biology. This breaks if:
- Batch 1 is diseased tissue, Batch 2 is healthy (biological difference!)
- Different labs collected fundamentally different samples

**Sufficient overlap**: Needs enough similar cells across batches to learn the transformation.

## Practical Takeaway

scVI treats batch correction as a **statistical denoising problem**:

1. Learn what cells "really are" (z) - this should be batch-free
2. Learn how each batch distorts observations  
3. Use the biological representation (z) for analysis
4. When generating corrected data, apply a standard transformation instead of batch-specific ones

The neural networks provide the flexibility to learn complex, gene-specific batch effects that simpler methods miss.
