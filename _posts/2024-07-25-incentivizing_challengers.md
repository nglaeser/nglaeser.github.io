---
layout: single
title:  "Incentivizing challengers in optimistic settings"
header: 
  teaser: 
categories: 
  - Notes
tags:
  - naysayer
  - xpost
  - hackmd
share: false
---

*Read this note on HackMD [here](https://hackmd.io/@nglaeser/BkMvYaJKC).*

Challengers earn a reward (some portion of the prover's collateral) if they successfully challenge an optimistic proof. Let's look at the costs and rewards of the parties involved (ignoring network fees).

## Prover (rollup)
### Costs
- **Computation costs** ($P_{\sf comp}$) - The cost of doing the required computation, which includes computing the actual "rolled up" state transition (new state $y = f(x)$, where $x$ are the input transactions).
    - In naysayer proofs, this also includes computing a SNARK $\pi$ that $y = f(x)$.
- **Collateral** ($P_{\sf coll}$) - To disincentivize malicious/lazy provers. Will be returned if no successful challenge happens during the challenge period.

### Rewards
- **Rollup reward** (orthogonal to this note, so I'm not going to talk about it) - This is similar to miner fees on the L1, and it's paid by rollup users to the prover(s) for their services

## Challenger
### Costs
- **Observation costs** ($C_{\sf obs}$) - The cost to monitor whether the prover's reported results $y$ (or $\pi$) are correct.
    - In naysayer proofs, the observation cost is lower since it does not require re-computing $f$ (computationally expensive), but instead only running SNARK verification on $\pi$. This expands the potential challenger pool to other, resource-constrained parties, e.g., light clients.
- **Collateral** ($C_{\sf coll}$) - A challenger must put down some collateral in order to challenge a result $y$, since resolving the dispute will require the involvement of the prover (bisection game).
    -  In naysayer proofs, the dispute resolution process is non-interactive and does not require any action by the original prover. Therefore, the challenger does not need to cover any prover costs, only the on-chain costs of settling the dispute (the cost of the `VerifyNay` algorithm). This lowers the challenger collateral.
- **Computation costs** ($C_{\sf comp}$) - The cost of the computation involved in creating a challenge.
    - In naysayer proofs, this is the cost of the `Naysay` algorithm.
- **Submission costs** ($C_{\sf sub}$) - The cost of submitting the challenge on-chain.
    - In naysayer proofs, this is the cost of posting the naysayer proof $\pi_{\sf nay}$ in calldata.

### Rewards
- **Prover collateral** ($P_{\sf coll}$) - If the challenge is deemed correct, the challenger receives the prover collateral (minus the amount spent to verify the challenge)

# Why be a challenger?

Whether or not it's "worth it" to be a challenger (assuming rational, risk-neutral parties) depends on the above parameters. Roughly speaking, the rewards outweigh the costs if

$$
P_{\sf coll} - V > n_{\sf obs} \cdot C_{\sf obs} + n_{\sf sub} \cdot (C_{\sf comp} + C_{\sf sub})
$$

where 
- $V$: the cost of verifying the challenge on-chain
- $n_{\sf obs} = \mathbb{E}[\text{# observations until success}]$: the expected number of observations before the challenger notices an incorrect result
- $n_{\sf sub} = \mathbb{E}[\text{# submissions until success}]$: the expected number of submitted challenges until a challenge is submitted successfully. (This is because even if a party spots a faulty update, someone else may notice it too and also submit a challenge. So it may be that a challenger wastes work not only by observing, but by computing and submitting a challenge only to be beaten to the punch.)

A rational challenger should only submit correct challenges, so it should always expect to have its collateral returned.

Purely theoretically speaking, since $n_{\sf obs}, n_{\sf sub}$ increase as more challengers enter the system, the prover collateral will have to be high to incentivize challengers to expend resources on observation and challenging ($C_{\sf obs}$, $C_{\sf comp}$, and $C_{\sf sub}$).

In practice, challengers have some vested interest in the proper functioning of the system as a whole. That is, there is an additional positive term $I$ on the left side of the inequality which allows $P_{\sf coll}$ to be lower.

# Advantages of naysayer proofs

Naysayer proofs offer lower $C_{\sf obs}$ (and probably also $V$, $C_{\sf comp}$ and $C_{\sf sub}$, but more precise analysis is needed), lowering the necessary magnitude of $P_{\sf coll}$ compared to "vanilla" optimistic proofs. This also has the effect of lowering the barrier of entry for challengers:

- **Lower challenger collateral.** Because dispute-resolution is non-interactive, challengers are not required to put up any collateral to challenge.
- **Lower observation cost.** The passive cost of monitoring the prover(s) for cheating is lower, since it requires only a SNARK verification (efficient) rather than recomputing the rollup's work $f$ (slow). This "democratizes" challenging by making it accessible to, e.g., end users with light clients.

There's no such thing as a free lunch, though, and these advantages come at the cost of increasing the burden on rollup services (provers), who must now additionally compute a SNARK (which is quite computationally expensive) and post it in blob data.

# Other considerations

This note only discusses the collateral/rewards aspects of vanilla optimistic rollups vs. naysayer proofs. Another direction to explore is what effect the naysayer paradigm has on the *length* of the optimistic challenge period, which in vanilla rollups is generally set to [7 days](https://kelvinfichter.com/pages/thoughts/challenge-periods/) and causes a significant usability burden (this is one of the main criticisms of optimistic rollups).