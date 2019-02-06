---
layout: post
name: Fair Resource Distribution Algorithm
tags: [algorithm, distribution, fair, resource, sharing]
math: true
permalink: /:title/
hidden: true
---

The following is a rendered version of the *Fairdist Algorithm*, a way to share
and distribute a dynamic set of resources with a dynamic number of peers.

## Prerequisites

 * Let $$P \subset \mathbb{N}$$ be a finite set of peers.
 * Let
   $$\forall p \in P: C_p \in \{ s \in \mathbb{R} \mid s \geq 0 \}$$
   be a total amount of consumed resources of each individual peer.
 * Let $$R \in \mathbb{R}$$ be a total amount of remaining resources.

Based on these, we define:

 * Let $$C := \{ C_p > 0 \mid p \in P \}$$ be the set of consumed resources.
 * Let $$C^t := \sum_{c \in C} c$$ be the total amount of consumed resources.
 * Let $$n := \lvert C \rvert$$ be the number of active peers.
 * Let $$f: (\mathbb{N}, \mathbb{R}) \to \mathbb{R}$$ be a function that
   computes how many resources a new peer is allowed to allocate. It takes the
   number of peers $n$ and the remaining resources $R$ as input.

For this version of the algorithm, we use:

$$f(n, R) \hookrightarrow \frac{R}{n + 2}$$

## Theorem #1

$$R \geq \frac{C^t}{n} \implies f(n, R) \geq \frac{R + C^t}{(n + 2)^2}$$

That is, if the reserve is bigger than an *n-th* of the consumed resources, the
allocation scheme $$f$$ guarantees that a peer can allocate at least an
*n-plus-2-squared-th* amount of the total resources.

## Proof

$$
\begin{align}
f(n, R) &\geq \frac{R + C^t}{(n + 2)^2}\\
\frac{R}{n + 2} &\geq \frac{R + C^t}{(n + 2)^2}\\
R &\geq \frac{R + C^t}{n + 2}\\
Rn + 2R &\geq R + C^t\\
Rn + R &\geq C^t\\
R (n + 1) &\geq C^t\\
R &\geq \frac{C^t}{n + 1}\\
R \geq_{req} \frac{C^t}{n} &\geq \frac{C^t}{n + 1}\\
\end{align}
$$

## Theorem #2

$$R - f(n, R) \geq \frac{C^t + f(n, R)}{n + 1}$$

Alternatively written as:

$$R_2 \geq \frac{C^t_2}{n_2}$$

That is, in the environment resulting when a single peer allocates its
resources, the requisites for Theorem #1 still hold true.

## Proof

$$
\begin{align}
R - f(n, R) &\geq \frac{C^t + f(n, R)}{n + 1}\\
R - \frac{R}{n + 2} &\geq \frac{C^t + \frac{R}{n + 2}}{n + 1}\\
R(n + 1) - \frac{R(n + 1)}{n + 2} &\geq C^t + \frac{R}{n + 2}\\
R(n + 1) - \frac{R(n + 1) + R}{n + 2} &\geq C^t\\
R(n + 1) - \frac{R(n + 2)}{n + 2} &\geq C^t\\
R(n + 1) - R &\geq C^t\\
nR &\geq C^t\\
R &\geq \frac{C^t}{n}\\
\end{align}
$$
