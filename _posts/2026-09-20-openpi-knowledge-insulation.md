---
published: true
layout: post
title: Implementing Knowledge Insulation in OpenPI
date: 2026-09-20 16:32:00 +0800
description: How Knowledge Insulation separates FAST and flow-matching gradients in OpenPI, and why sharing the VLM forward changes the training objective.
tags: 
categories: tech
---

# Why Sharing the VLM Forward Can Change the Objective

Knowledge Insulation (KI) is a training strategy introduced by Physical Intelligence to combine the semantic capabilities of a pretrained vision-language model (VLM) with a continuous action expert, while preventing continuous control optimization from directly modifying the VLM representation.

The basic idea is simple: a discrete autoregressive action objective trains the VLM, while the continuous flow-matching objective trains the action expert with gradients blocked from flowing back into the VLM.

Physical Intelligence describes the method in:

[VLAs that Train Fast, Run Fast, and Generalize Better](https://www.pi.website/research/knowledge_insulation)

I implemented and experimented with KI for π0.5 in my OpenPI fork:

[openpi05-post-train](https://github.com/yang-si-hang/openpi05-post-train)

This post focuses on one implementation detail that turned out to be important:

> Sharing VLM **parameters** between the FAST and flow-matching branches is natural. Sharing the same VLM **forward pass or KV cache** is an additional assumption, and may require changing the training objective.

In the implementation studied here, forcing the two branches to share a single VLM prefix changed the FAST autoregressive sequence itself. Experiments show that this change significantly alters the VLM optimization direction. However, the results do **not** establish the exact serialization used during π0.5 pretraining.

## 1. Knowledge Insulation in the OpenPI implementation

The implementation optimizes two objectives:

$$
L = \lambda_{\mathrm{FAST}}L_{\mathrm{FAST}} + \lambda_{\mathrm{FM}}L_{\mathrm{FM}}.
$$

The FAST branch predicts discretized action tokens autoregressively. Its gradients update the vision encoder and VLM.

The flow-matching branch predicts continuous robot actions using a separate action expert. The VLM prefix is evaluated first, but its KV cache is detached before being consumed by the action expert:

```python
_, kv_cache = self.PaliGemma.llm(
    [prefix_tokens, None],
    mask=prefix_attn_mask,
    positions=prefix_positions,
)

kv_cache = jax.tree.map(
    jax.lax.stop_gradient,
    kv_cache,
)
```

The resulting gradient topology is:

```text
FAST CE
   ├──> SigLIP
   └──> VLM

Flow Matching
   ├──X SigLIP / VLM
   └──> Action Expert + action projections
```

Equivalently,

$$
\nabla_{\theta_{\mathrm{VLM}}}L_{\mathrm{FM}}=0.
$$

This is the essential role of Knowledge Insulation: the continuous controller can consume VLM features, but the continuous action loss cannot directly rewrite the VLM.

The implementation also avoids an unnecessary second image-encoder pass. Images are encoded once by SigLIP, and the resulting image tokens are reused by both branches:

```text
                    ┌── FAST VLM forward ──> FAST CE
Images ──> SigLIP ──┤
                    └── FM VLM forward ──> detached KV
                                             │
                                             ▼
                                       Action Expert
                                             │
                                             ▼
                                         Flow loss
```

The image representation is shared, but the two VLM forwards remain separate.

That distinction matters.

## 2. Why not also share the VLM forward?

At first, sharing the VLM prefix seems like a natural optimization.

Both branches process the same image, language instruction, and robot state. If the VLM could be executed once and its KV cache reused, KI training would require substantially less computation.

The difficulty is that, in the legacy FAST formulation used in this implementation, FAST and flow matching do not use the same autoregressive sequence.

The FAST tokenizer constructs approximately:

```text
Task: <lowercase instruction>, State: <discretized state>;
Action: <FAST action tokens> | <EOS>
```

with:

```text
Task + State       : bidirectional prefix
Action: + FAST ... : autoregressive postfix
```

The CE loss is applied to the postfix, including `Action:`.

The FAST objective can therefore be written conceptually as:

$$
p(\mathrm{Action:}, z_1,\ldots,z_N \mid I,\mathrm{Task},\mathrm{State}).
$$

The standard π0.5 flow-matching prefix is different:

```text
Task: <instruction>, State: <state>;
Action:
```

Here, `Action:` is already part of the conditioning context.

The action expert therefore receives features corresponding approximately to:

$$
p(a \mid I,\mathrm{Task},\mathrm{State},\mathrm{Action:}).
$$

The two branches therefore do not have identical sequence boundaries.

## 3. What changes when the VLM prefix is shared?

An earlier implementation attempted to share VLM computation by constructing one canonical π0.5 prefix:

```text
Image + Task + State + Action:
```

and using it for both FAST and flow matching.

FAST then predicted only:

```text
FAST_1 FAST_2 ... FAST_N | EOS
```

This changes the FAST factorization from:

$$
p(\mathrm{Action:}, z_{1:N} \mid C)
$$

to:

$$
p(z_{1:N} \mid C,\mathrm{Action:}).
$$

where the context is:

$$
C=(I,\mathrm{Task},\mathrm{State}).
$$

This is not merely a computational refactor. It changes what is treated as conditioning information and what is treated as a prediction target.

There is also a smaller serialization difference: the legacy FAST tokenizer lowercases the natural-language instruction, whereas the standard π0.5 tokenizer preserves its case.

The important point is not that one serialization is universally correct. It is that **sharing the VLM forward is only semantics-preserving when both objectives are defined using the same prefix and autoregressive boundaries**.

## 4. Why the sequence difference may matter

π0.5 pretraining is not simply continuous-action regression. Its VLM is trained using multiple types of autoregressive supervision, including robot-related and semantic outputs.

In such a multi-task autoregressive model, tokens such as:

```text
Action:
```

can naturally act as output-type markers:

```text
context → output type → output contents
```

Conceptually, the model might learn structures such as:

```text
Task / State → Action:  → action tokens
Task / State → Subtask: → language tokens
Task / State → object-related outputs
```

Under such a formulation, moving `Action:` from the predicted suffix into the conditioning prefix changes the autoregressive task presented during post-training.

However, this argument is only a **plausibility argument**.

The exact internal serialization used during π0.5 pretraining is not publicly available, and the released checkpoint alone cannot uniquely reveal the original training sequence.

Therefore, the strongest theoretical statement is:

> If FAST and flow matching require different autoregressive sequence semantics, forcing them to share the same VLM prefix changes at least one of the objectives.

This statement does not require knowing the exact π0.5 pretraining format.

## 5. Frozen-checkpoint likelihood probe

I also compared the two serialization choices using the frozen released π0.5-base checkpoint.

For every example:

- the image was identical;
- the state was identical;
- the continuous action trajectory was identical;
- the resulting FAST action tokens were identical;
- only the context serialization changed.

The two variants were:

```text
Legacy:
Task(lowercase) + State
    → Action: + FAST tokens

Canonical:
Task + State + Action:
    → FAST tokens
```

The main metric compared NLL only on the **same FAST action-code targets**, excluding the `Action:` marker itself.

On 100 samples from our downstream UR manipulation dataset:

```text
Mean action-token CE

Legacy:     16.0106 nats/token
Canonical:  16.1231 nats/token

Canonical - Legacy:
+0.1125 nats/token

Bootstrap 95% CI:
[+0.0430, +0.1870]

Per-sample wins:
Legacy     62 / 100
Canonical  38 / 100
```

Pooling all 2,857 matched FAST action tokens gave:

```text
Legacy:     15.7656
Canonical:  15.8369
```

The difference is small but statistically consistent on this dataset.

### What this experiment actually tells us

This experiment does **not** show that π0.5 was pretrained with the legacy serialization.

The evaluation data come from our own downstream UR manipulation dataset rather than the original π0.5 pretraining distribution. The robot embodiment, observations, action distribution, prompts, and task distribution may all differ substantially from those seen during pretraining.

Therefore the appropriate interpretation is:

> On our downstream UR data distribution, the frozen released π0.5-base checkpoint assigns slightly higher likelihood to the same FAST action targets when they are evaluated under the legacy context.

This is evidence about **downstream compatibility with the released checkpoint**, not evidence that uniquely identifies the original pretraining recipe.

The distinction is important.

A stronger claim about pretraining serialization would require similar behavior across multiple datasets and embodiments that better cover the model's original training distribution. Even then, checkpoint behavior alone would not uniquely recover the original data serialization.

## 6. The stronger result: the optimization direction changes

The likelihood probe is useful but not the strongest evidence in this study.

More importantly, changing the FAST serialization substantially changes the fine-tuning gradient.

Several FAST implementations were compared while keeping the model parameters, observations, actions, and random inputs fixed.

When comparing the legacy FAST objective against the canonical shared-prefix objective, the gradient cosine similarity was approximately:

```text
Image encoder: 0.811
VLM LoRA:      0.855
```

The cosine similarity between two gradient vectors is:

$$
\mathrm{cos}(g_1,g_2) = \frac{g_1^\top g_2}{\lVert g_1\rVert \lVert g_2\rVert}.
$$

A value near one means that the two implementations produce nearly the same optimization direction.

Values around 0.81–0.86 therefore represent a substantial change in the VLM update direction. The VLM gradient norm also increased by approximately 45% under the canonical formulation.

In contrast, when the **objective was kept the same** and only the computational path changed from a one-shot forward to an incremental KV-cache forward, the gradient similarities remained around:

```text
Image encoder: ~0.990
VLM LoRA:      ~0.993
```

There were measurable BF16/XLA numerical differences, but they were much smaller.

The comparison therefore looks like:

```text
Same objective
one-shot → incremental
        ≈ relatively small numerical change

Legacy FAST objective
→ canonical shared-prefix FAST objective
        ≈ substantially different gradient direction
```

This is the most robust conclusion of the experiment:

> The shared-prefix refactor was not a numerically transparent implementation optimization. It materially changed the FAST optimization objective.

This conclusion does not depend on knowing the original π0.5 pretraining serialization.

## 7. Interpreting the higher flow-matching loss under KI

Another observation was that the KI run reached a noticeably higher continuous flow-matching loss than conventional LoRA training.

For example:

```text
KI flow loss:
~0.0078

Conventional LoRA flow loss:
~0.0018
```

At first glance, this might appear to indicate that KI is not working correctly.

That conclusion would be too strong.

The two training problems have different optimization constraints.

Without KI, the flow-matching loss can update both the action expert and the VLM representation:

$$
L_{\mathrm{FM}} \rightarrow \{\theta_{\mathrm{VLM}}, \theta_{\mathrm{AE}}\}.
$$

Under KI, the same loss is constrained to update only the continuous-action path:

$$
L_{\mathrm{FM}} \rightarrow \theta_{\mathrm{AE}},
$$

while:

$$
\nabla_{\theta_{\mathrm{VLM}}} L_{\mathrm{FM}} = 0.
$$

The KI optimization problem therefore has fewer degrees of freedom for reducing the continuous-action training loss.

A higher flow-matching loss is consequently possible even when KI is implemented correctly.

### A moving representation problem

There is an additional asymmetry.

FAST continues to update the VLM:

```text
FAST CE
   ↓
VLM representation changes
```

while the action expert consumes that representation:

```text
VLM representation
        ↓
   Action Expert
        ↓
     FM loss
```

but the FM objective cannot modify the VLM:

```text
FM loss ──X──> VLM
```

The action expert is therefore learning on top of a representation that may continue to move during training.

This creates a possible **representation-drift / moving-target problem**:

$$
h_t=f_{\theta_t}(x),
$$

while:

$$
\theta_{t+1} = \theta_t - \eta_{\mathrm{VLM}} \nabla L_{\mathrm{FAST}}.
$$

The action expert must continuously adapt to the changing representation:

$$
a_t = g_{\phi_t}(h_t).
$$

This does not imply that KI is flawed. It means that optimization hyperparameters that work for ordinary joint LoRA fine-tuning may not automatically be optimal after the gradient paths are separated.

## 8. KI may require different optimization hyperparameters

In the current implementation, the VLM LoRA parameters and action-expert parameters are trained under the same optimizer schedule.

There is no theoretical reason that the optimal learning rate should be identical for both parameter groups.

The VLM is already pretrained and only needs controlled adaptation, while the action expert must fit the continuous downstream control problem.

A reasonable KI optimization regime may therefore satisfy:

$$
\eta_{\mathrm{VLM}} < \eta_{\mathrm{AE}}.
$$

This would allow the VLM representation to move more conservatively while allowing the action expert to adapt more quickly.

For example, a controlled experiment could compare:

```text
VLM LR : Action Expert LR

1.00 : 1
0.50 : 1
0.25 : 1
0.10 : 1
```

while keeping dataset, initialization, batch size, FAST formulation, and training duration fixed.

Useful diagnostics would include:

```text
FAST CE
Flow-matching loss
VLM gradient norm
Action-expert gradient norm
xyz / z prediction error
rotation error
gripper prediction error
real-robot success
```

If reducing the VLM learning rate lowers the FM loss while preserving useful FAST learning, that would support the hypothesis that representation drift contributes to the optimization gap.

If the FM loss remains much higher regardless of the relative learning rates, other explanations become more likely, including action-expert capacity, representation quality, training duration, or limitations of using KI on this downstream dataset.

### Why simply changing the loss weights may not be sufficient

The KI objective contains explicit weights:

$$
L = \lambda_{\mathrm{FAST}}L_{\mathrm{FAST}} + \lambda_{\mathrm{FM}}L_{\mathrm{FM}}.
$$

However, because the two losses largely update disjoint parameter groups, increasing one loss weight is not necessarily equivalent to giving that branch a proportionally larger effective optimization step.

With Adam-like optimizers, multiplying a gradient by a constant affects both its first- and second-moment estimates, so much of the scale change can cancel in the normalized update.

Therefore, if the goal is specifically to make the VLM adapt more slowly than the action expert, **separate parameter-group learning rates are a more direct control mechanism** than simply changing the two loss weights.

## 9. The current implementation choice

Given the evidence above, the implementation I currently prefer keeps the computation that can be safely shared while preserving the two autoregressive formulations:

```text
                           ┌─ Legacy FAST VLM
                           │  Task(lowercase) + State
Images ──> shared SigLIP ──┤       ↓
                           │  Action: + FAST tokens
                           │       ↓
                           │     FAST CE
                           │
                           └─ Canonical π0.5 VLM
                              Task + State + Action:
                                      ↓
                                  KV cache
                                      ↓
                                stop_gradient
                                      ↓
                                Action Expert
                                      ↓
                                  Flow loss
```

The VLM parameters remain shared.

The SigLIP image representation is computed once and reused.

The VLM activations and KV caches are computed separately because the two branches currently use different sequence semantics.

The distinction is:

> **Parameter sharing is part of KI. Forward-pass sharing is an additional implementation assumption.**

The latter is safe only when it preserves the objective.

## 10. Limitations

Several limitations are important when interpreting these experiments.

First, the frozen-checkpoint NLL comparison was performed on our downstream UR manipulation dataset rather than data sampled from the original π0.5 pretraining distribution.

Therefore it cannot establish which serialization Physical Intelligence used during pretraining.

The appropriate conclusion is only that the legacy serialization was slightly more compatible with the released checkpoint **on this downstream dataset**.

Second, the likelihood difference was small and not universal. The canonical formulation produced lower per-example loss on 38 of 100 samples.

Third, the experiments were performed on one robot embodiment and a limited set of manipulation tasks. The result may not generalize to other embodiments, prompts, action spaces, or datasets.

Fourth, a higher KI flow-matching loss does not by itself indicate an implementation error. KI changes the optimization problem by preventing FM gradients from updating the VLM. The current KI run also reused hyperparameters designed for a different optimization topology. The effect of parameter-group learning rates and other KI-specific optimization choices has not yet been systematically studied.

Fifth, the downstream implementation is not equivalent to Physical Intelligence's full training recipe. The VLM is mainly adapted using FAST supervision from the same downstream robot dataset, whereas the original KI recipe involves a substantially broader training mixture. This difference may affect representation stability and optimization dynamics.

Sixth, real-robot behavior cannot isolate the effect of FAST serialization. Checkpoint selection, data coverage, LoRA configuration, action-expert capacity, optimization hyperparameters, and deployment details can all affect policy performance.

Finally, the conclusion is **not** that a shared VLM forward is impossible.

A shared forward would be perfectly reasonable if the FAST and continuous-action objectives were designed around the same conditioning sequence.

The narrower result is:

> In this OpenPI KI implementation, sharing the VLM prefix required changing the legacy FAST autoregressive sequence. That change materially altered the VLM gradient direction. Therefore the shared-prefix implementation should be treated as a different training objective rather than a purely computational optimization.

## Takeaway

Knowledge Insulation is primarily about **gradient routing**.

It does not require the FAST and continuous-action branches to share identical intermediate activations.

For the OpenPI implementation studied here, the conservative design is:

```text
shared SigLIP computation
+
shared VLM parameters
+
separate FAST and FM VLM forwards
+
stop-gradient from FM into the VLM
```

The current experiments provide strong evidence that replacing the two VLM forwards with one shared canonical prefix changes the FAST optimization objective.

They provide only weak, downstream-specific evidence about which serialization is more compatible with the released π0.5-base checkpoint, and they do not reveal the original π0.5 pretraining serialization.

Finally, the higher flow-matching loss observed under KI should not immediately be interpreted as a failure of KI. Since KI changes both the gradient topology and the optimization constraints, it likely deserves its own hyperparameter study—especially the relative learning rates of the VLM and action expert.
