---
layout: single
title:  "Prover, Verifier, and Challenger costs in rollups"
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

*Read this note on HackMD [here](https://hackmd.io/@nglaeser/Bkb8SZ6oR).*

Let's look at the costs (in terms of computation) and rewards (in tokens) of the main parties involved in a rollup (both zk and optimistic).

## Prover/Proposer

The prover $P$ (or as I will refer to it in the case of optimistic rollups, proposer) posts some updated state $y = f(x)$ to the blockchain (L1) after applying some update $x$. In a zk-rollup, $P$ also posts a SNARK $\pi$ that $y = f(x)$.

### Costs
1. Computing $y = f(x)$
1. `[zk only]` Computing $\pi \gets {\sf SNARK}_f.{\sf Prove}(y, x)$
1. `[optimistic only]` Collateral $c_P$ to back the claim that $y$ is correct
1. `[optimistic only]` Engaging with any interactive challengers (i.e., bisection game)

### Rewards
1. Pre-established reward $r$ if $y$ is accepted

## Verifier

The verifier $V$ runs on the L1 and determines whether or not to accept $y$. In an optimistic rollup, this means arbitrating the dispute if a challenge is submitted; in a zk-rollup, this means running the SNARK verifier.

### Costs
1. `[zk only]` Computing ${\sf SNARK}_f.{\sf Vrfy}(y, \pi)$
1. `[optimistic only]` Resolving dispute (e.g., a step of the evaluation of $f(x)$ as determined by the bisection game)

### Rewards
N/A

## Challenger

The challenger $C$ monitors the rollup for any faulty submissions $y$ to challenge. In the case of a zk-rollup, **there is no challenger role.**

### Costs
1. `[optimistic only]` Collateral $c_C$ to back the claim that $y$ is *in*correct (i.e., to back the challenge)
1. `[optimistic only]` Engaging in an interactive challenge with $P$ (i.e., bisection game)

### Rewards
1. `[optimistic only]` $P$'s collateral if $y$ is rejected

## Who pays for what costs

- $P$'s cost \#1 (P1) will be covered by the rollup reward, so $r$ should be set so that $r$ > P1.
- P2, if relevant, should also be covered by the rollup reward, so in zk-rollups, $r$ should be set so that **$r$ > P1 + P2**.
- If $P$ is honest, P3 should always be returned.
- If $P$ is honest, P4 will be reimbursed from $C$'s collateral, so $c_C$ should be set so that $c_C$ > P4.
- V1 should be covered by users of the rollup service.
- V2 should be reimbursed by the collateral of loser of the dispute, so in optimistic rollups, $c_C, c_P$ should be set so that **$c_C$ > P4 + V2** and **$c_P$ > C2 + V2** (see below).
- If $C$ is honest, C1 should always be returned.
- If $C$ is honest, C2 will be reimbursed from $P$'s collateral, so $c_P$ should be set so that $c_P$ > C2 (and in fact higher, see above).

These constraints determine lower bounds for the values of $r$, $c_P$, and $c_C$. The final lower bounds are bolded. The funding for $r$ (and V1) comes from transaction fees paid by users of the rollup service. This accounts for all the costs/rewards in the system.