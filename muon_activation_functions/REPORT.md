# What's the best activation function to use with Muon for (small) LM?

Date: 2026-06-11

## Summary

I've ran a controlled activation-function sweep for a small GPT-style language model trained with Muon on `shakespeare_char` (using the beloved [NanoGPT](https://github.com/karpathy/nanogpt)). The strongest validation results so far came from smooth, non-gated activations: `tanh`, `mish`, `celu`, `silu`, and `elu`.

The best observed validation loss was `1.4679` from `tanh` at step `3750`. The next best results were close: `mish` at `1.4706`, `celu` at `1.4733`, `silu` at `1.4735`, and `elu` at `1.4740`. This group is tight enough that we should not treat the exact rank as final without multiple seeds*.

The clearest negative result is the gated MLP group (`swiglu`, `geglu`, `reglu`) in the current configuration. These runs learned quickly early, but overfit hard and ended with the worst final validation losses among the serious contenders.

## My Setup

Common setup across all the 19 completed runs:

- Dataset: `shakespeare_char`
- Model: GPT-style character model
- Layers: `6`
- Heads: `6`
- Embedding size: `384`
- Block size: `256`
- Batch size: `64`
- Dropout: `0.2`
- Optimizer: Muon with auxiliary AdamW
- Muon learning rate: `0.02`
- Muon momentum: `0.95`
- AdamW learning rate for embeddings/scalars: `0.001`
- Weight decay: `0.1`
- Max iterations: `5000`
- Eval interval: `250`
- Eval iterations: `200`
- Device: CUDA, bfloat16
- W&B project: `shakespeare-char-muon`

All 19 runs exited successfully with exit code `0`.

## Results

Sorted by best validation loss:


| Rank | Activation     | Best Val Loss | Best Step | Final Val Loss | Final Train Loss |
| ---- | -------------- | ------------- | --------- | -------------- | ---------------- |
| 1    | `tanh`         | 1.4679        | 3750      | 1.4762         | 1.0787           |
| 2    | `mish`         | 1.4706        | 3000      | 1.4963         | 0.9751           |
| 3    | `celu`         | 1.4733        | 4750      | 1.4790         | 1.0925           |
| 4    | `silu`         | 1.4735        | 3500      | 1.4934         | 0.9913           |
| 5    | `elu`          | 1.4740        | 3750      | 1.4797         | 1.0925           |
| 6    | `hardswish`    | 1.4784        | 3500      | 1.4909         | 1.0509           |
| 7    | `leaky_relu`   | 1.4784        | 3000      | 1.5196         | 0.9266           |
| 8    | `relu`         | 1.4798        | 3000      | 1.5221         | 0.9242           |
| 9    | `gelu_tanh`    | 1.4819        | 3000      | 1.5384         | 0.9051           |
| 10   | `quick_gelu`   | 1.4820        | 3000      | 1.5368         | 0.9052           |
| 11   | `gelu`         | 1.4823        | 3000      | 1.5385         | 0.9051           |
| 12   | `softplus`     | 1.4871        | 4250      | 1.4928         | 1.1352           |
| 13   | `prelu`        | 1.4898        | 3000      | 1.5474         | 0.8623           |
| 14   | `selu`         | 1.5004        | 4750      | 1.5070         | 1.1650           |
| 15   | `squared_relu` | 1.5102        | 2500      | 1.6025         | 0.8327           |
| 16   | `reglu`        | 1.5332        | 2000      | 1.6893         | 0.7294           |
| 17   | `geglu`        | 1.5353        | 2000      | 1.7393         | 0.6834           |
| 18   | `swiglu`       | 1.5435        | 2000      | 1.7454         | 0.6806           |
| 19   | `sigmoid`      | 1.5934        | 4750      | 1.5971         | 1.3280           |


## Key takeaways

### 1. Smooth Non-Gated Activations Are Leading

The best validation losses came from `tanh`, `mish`, `celu`, `silu`, and `elu`. These activations did not always minimize training loss the fastest, but they generalized better in this small-data regime.

This is the most useful practical takeaway so far: for this small Shakespeare character model with Muon, the best activations are not necessarily the standard GPT defaults. Take this with a grain of salt.

### 2. GELU Was Solid But Not Best

The GELU family was consistent:

- `gelu`: best val `1.4823`
- `gelu_tanh`: best val `1.4819`
- `quick_gelu`: best val `1.4820`

These are very close to each other and only modestly behind the top cluster. GELU is still a good baseline, but in this sweep it was not the best choice.

### 3. ReLU-Like Activations Learned Fast But Overfit

`relu`, `leaky_relu`, `prelu`, and `squared_relu` drove training loss lower than the top validation activations. However, their final validation losses degraded more strongly.

Examples:

- `relu`: best val `1.4798`, final val `1.5221`
- `leaky_relu`: best val `1.4784`, final val `1.5196`
- `prelu`: best val `1.4898`, final val `1.5474`
- `squared_relu`: best val `1.5102`, final val `1.6025`

This suggests the sharper / more aggressively fitting activations may be too high-capacity or too easy to overfit under the current regularization and training length.

### 4. Gated Activations Are Not Parameter-Matched Yet

The gated activations are currently not directly comparable to the standard MLP activations.

In our model definition our activations use:

```python
fc_dim = 2 * hidden_dim if is_gated_activation(config.activation) else hidden_dim
```

With `mlp_hidden_mult = 4.0`, this means `swiglu`, `geglu`, and `reglu` have a larger first MLP projection than the non-gated activations.

Approximate parameter counts:

- Non-gated activations: `10.75M`
- Gated activations: `14.28M`

Despite having more parameters, the gated runs overfit badly:

- `reglu`: best val `1.5332`, final val `1.6893`
- `geglu`: best val `1.5353`, final val `1.7393`
- `swiglu`: best val `1.5435`, final val `1.7454`

This does not prove that gated activations are bad. It proves that the current gated configuration is not working well for this tiny dataset and training recipe.

### 5. Final Loss Is Misleading Here

Most strong runs peaked before step `5000`. If we rank by final validation loss, we penalize activations that learned quickly and then overfit. The correct comparison for this sweep is **best validation loss**.

That said, final validation loss is still useful because it exposes overfitting dynamics:

- `tanh`, `celu`, `elu`, and `softplus` degraded only mildly after their best checkpoints.
- `gelu`, `relu`, `prelu`, `squared_relu`, and the gated activations degraded much more.

## Step-Level Behavior

At step `1000`, the fastest early learners were mostly gated or sharp activations:

- `geglu`: val `1.6341`
- `swiglu`: val `1.6396`
- `reglu`: val `1.6451`
- `squared_relu`: val `1.6636`
- `prelu`: val `1.6738`

By step `3000`, the ranking shifted:

- `mish`: val `1.4706`
- `silu`: val `1.4760`
- `leaky_relu`: val `1.4784`
- `relu`: val `1.4798`
- `gelu_tanh`: val `1.4819`

By step `4000` and later, the smoother activations (`tanh`, `celu`, `elu`, `silu`, `mish`) were clearly stronger:

The sweep therefore shows a tradeoff between early optimization speed and later validation stability.

## Current Interpretation

My perspective is that Muon is making several activations viable, but the small-data regime strongly rewards activations that do not overfit too aggressively.

The top cluster is not dominated by the usual Transformer defaults. `tanh` winning is surprising but plausible here because this is a small character-level dataset, not a large-token pretraining run. Its bounded output may be acting as useful implicit regularization.

`mish` and `silu` look strong because they combine smoothness with <span data-glossary="non-monotonic">non-monotonic</span> or gated-like behavior without adding the extra MLP projection used by the explicit GLU variants.

`celu` and `elu` also look interesting because they keep stable final validation performance. They may be less exciting from an optimization-speed perspective, but they generalize well in this experiment.

## Caveats

1. Single-seed results are not enough.

The top activations differ by roughly `0.002` to `0.006` validation loss in several places. That is small enough that seed variance could reorder the top five.

2. Tiny Shakespeare is a narrow benchmark.

The result may not transfer to larger SLM pretraining, token-level datasets, or longer-context setups.

3. Gated activations need a fair rerun.

The current gated runs are larger than the non-gated runs. They should be rerun with `mlp_hidden_mult = 2.6667` or another parameter-matched setting.

4. Training length interacts with activation choice.

Some activations look much better at their best checkpoint than at step `5000`. Early stopping or stronger regularization could change the ranking.

5. Throughput varied.

Some activations were much faster per iteration than others. `relu` and `squared_relu` were fastest; gated variants were slowest. For real SLM work, quality per unit compute matters, not just best validation loss.

## Recommendations

### Keep for Multi-Seed Follow-Up

Run multiple seeds for:

- `tanh`
- `mish`
- `celu`
- `silu`
- `elu`
- `hardswish`
- `gelu`

These cover the current winner, the top smooth activations, a mobile-style swish variant, and the canonical Transformer baseline.

### Drop or Deprioritize

Deprioritize:

- `sigmoid`
- `squared_relu`
- current `swiglu`
- current `geglu`
- current `reglu`

The gated activations should not be abandoned permanently, but the current configuration is not worth extending without parameter matching.

### Next Experiments

1. Multi-seed top-7 sweep.

Run 3 to 5 seeds for `tanh`, `mish`, `celu`, `silu`, `elu`, `hardswish`, and `gelu`.

2. Parameter-matched gated sweep.

Rerun:

- `swiglu`
- `geglu`
- `reglu`

with `mlp_hidden_mult = 2.6667`.

3. Early-stopping comparison.

Compare activations by best checkpoint and by fixed compute budget. Report:

- best validation loss
- best step
- final validation loss
- train-validation gap at best step
- train-validation gap at final step

4. Regularization check.

For fast-overfitting activations such as `relu`, `prelu`, `gelu`, and gated variants, test whether higher dropout or lower MLP width improves validation.

5. Larger dataset sanity check.

Repeat a smaller activation shortlist on a token-level dataset before generalizing to SLM training claims.

## Provisional Ranking

Current best practical choices:

1. `tanh`
2. `mish`
3. `celu`
4. `silu`
5. `elu`
6. `hardswish`
7. `gelu`

Current baseline:

- `gelu`

Current surprise:

- `tanh`

Current warning:

- Gated activations overfit badly in the present, non-parameter-matched setup.

