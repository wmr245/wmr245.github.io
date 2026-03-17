---
layout: post
title: "Learning-LLaMA-Decoder-Only"
date: 2026-03-16 10:00:00 +0800
categories: [til]
tags: [LLaMA]
---

# Transformer & LLaMA Learning Log

> Date: 2026-03-16  
> Topic: Transformer fundamentals, masking, attention, Decoder-Only architecture, and LLaMA embedding  
> Language: Chinese

---

## 1. Learning Goals

Today I focused on building a bottom-up understanding of Transformer, especially these core questions:

- What are masks and masked attention?
- When are masks applied: before or after softmax?
- How are padding mask and look-ahead mask implemented?
- What do the rows and columns of the attention score matrix mean?
- What is encoder-decoder attention?
- How does a Decoder-Only model work?
- What are LLaMA and the embedding layer in LLaMA?
- What is the relationship between autoregressive modeling and Decoder-Only architecture?
- What is MLP in Transformer?

---

## 2. Key Concepts Learned

### 2.1 Mask and Masked Attention

**Mask** is a rule matrix used to tell attention which positions are visible and which are forbidden.

**Masked Attention** is standard attention with a mask added to the attention score matrix.

Core formula:

```math
Attention(Q, K, V) = softmax\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V
```

Where:

- allowed positions: `M = 0`
- blocked positions: `M = -inf` (or a very small negative number in implementation)

**Important conclusion: mask is applied before softmax, not after softmax.**

Why:

- If a blocked score becomes `-inf` before softmax, its attention weight becomes `0`
- If masking is done after softmax, the probability distribution is broken and extra re-normalization is needed

---

### 2.2 Two Common Masks

#### (1) Padding Mask

Used to block `<pad>` positions.

Purpose:

- sentence lengths in a batch are often padded to the same length
- `<pad>` has no semantic meaning
- the model should not attend to these positions

Typical implementation:

```python
padding_mask = (input_ids == pad_id)              # [B, L]
padding_mask = padding_mask[:, None, None, :]     # [B, 1, 1, L]
scores = scores.masked_fill(padding_mask, float('-inf'))
```

Key idea:

- attention score shape is usually `[B, heads, Lq, Lk]`
- padding mask only describes which **key** positions are invalid
- so it is reshaped to `[B, 1, 1, Lk]`
- then it is broadcast to all heads and all query positions

---

#### (2) Look-ahead Mask / Causal Mask

Used to prevent a position from seeing future tokens.

Purpose:

- in autoregressive generation, the current token cannot see future tokens

Typical implementation:

```python
mask = torch.triu(torch.ones(seq_len, seq_len, dtype=torch.bool), diagonal=1)
mask = mask.unsqueeze(0).unsqueeze(0)   # [1, 1, L, L]
scores = scores.masked_fill(mask, float('-inf'))
```

Key intuition:

- upper triangle = future positions
- future positions must be blocked

---

### 2.3 Why Reshape the Padding Mask

This was one of the main confusing points today.

Attention scores usually have shape:

```python
scores.shape = [B, heads, Lq, Lk]
```

But the original padding mask is only:

```python
padding_mask.shape = [B, Lk]
```

This mask only says:

> for each sample, which key positions are `<pad>`

It does **not** yet specify:

- which head
- which query position

So we reshape it to:

```python
padding_mask = padding_mask[:, None, None, :]
```

Which becomes:

```python
[B, 1, 1, Lk]
```

Then broadcasting expands it to:

```python
[B, heads, Lq, Lk]
```

Meaning:

> for every head and every query position in the same sample, the same invalid key columns are masked out.

---

### 2.4 Meaning of Score Matrix Rows and Columns

For a score matrix:

```math
score = QK^T
```

If shape is:

```python
[Lq, Lk]
```

Then:

- **row = query position**
- **column = key position**

So:

```math
score[i, j]
```

means:

> how strongly query position `i` wants to attend to key position `j`

Best memory sentence:

- **rows = who is looking**
- **columns = who is being looked at**

And softmax is applied row-wise:

> each query distributes its attention over all keys

---

### 2.5 Encoder-Decoder Attention (Cross-Attention)

This is the core attention layer used in the decoder of the original Encoder-Decoder Transformer.

Sources of Q / K / V:

- `Q` comes from decoder hidden states
- `K` comes from encoder outputs
- `V` comes from encoder outputs

So it is **not** self-attention.

It means:

> the decoder uses its current state as a query to retrieve useful information from the encoder outputs

Score shape:

```python
[B, heads, Ltgt, Lsrc]
```

Where:

- rows = decoder positions
- columns = encoder positions

Typical mask used here:

- source-side padding mask
- no causal mask

Reason:

- source sentence is already fully known
- only encoder-side `<pad>` must be blocked

---

### 2.6 Why “apples” Is Not Directly Mapped to “苹果”

Important correction learned today:

Cross-attention does **not** directly perform dictionary lookup.

What actually happens:

1. decoder attends strongly to the encoder position representing `apples`
2. encoder provides a semantic vector for that source position
3. decoder fuses that information with its own current hidden state
4. final hidden state is projected onto the target vocabulary
5. the target token with highest probability may become `苹果`

So:

- attention decides **where to look**
- output projection decides **what to generate**

Best memory sentence:

> Attention decides who to look at; the output layer decides what to say.

---

### 2.7 Where Masks Are Used in the Whole Transformer

#### Encoder Self-Attention

Usually:

- no causal mask
- optional source padding mask

Reason:

- encoder is bidirectional
- but `<pad>` should still be blocked if present

#### Decoder Self-Attention

Usually:

- causal mask
- target padding mask (if padded)

Reason:

- decoder must not see future tokens
- padded positions should not participate

#### Encoder-Decoder Attention

Usually:

- source padding mask only

Reason:

- decoder is attending to encoder outputs
- source side is fully known, so no future blocking is needed

---

### 2.8 Decoder-Only Architecture

Core definition:

> A Decoder-Only model stacks Transformer decoder-style blocks and uses autoregressive next-token prediction.

Typical representatives:

- GPT
- LLaMA

Key structural features:

- no encoder
- no encoder-decoder attention
- only masked self-attention + MLP blocks
- uses causal mask

Training target:

```math
P(x_t | x_{<t})
```

That is:

> predict the next token based on previous tokens

Training vs inference:

- **training**: full sequence is fed in parallel, but causal mask blocks future tokens
- **inference**: tokens are generated one by one autoregressively

---

### 2.9 Autoregressive vs Decoder-Only

A very important distinction:

- **autoregressive** = a modeling / generation strategy
- **decoder-only** = a model architecture

Relationship:

- most Decoder-Only LLMs are autoregressive
- but autoregressive models are not always Decoder-Only

Examples:

- GPT / LLaMA: autoregressive + Decoder-Only
- RNN language model: autoregressive but not Decoder-Only Transformer
- Encoder-Decoder translation model: target-side generation is autoregressive, but the overall model is not Decoder-Only

---

### 2.10 MLP in Transformer

MLP = **Multi-Layer Perceptron**

In Transformer, MLP is usually the **FFN (Feed-Forward Network)** inside each block.

Typical form:

```math
FFN(x) = W_2 \sigma(W_1 x + b_1) + b_2
```

Role:

- attention mixes information across tokens
- MLP transforms features **within each token position**

Best memory sentence:

> Attention is for communication across tokens; MLP is for internal processing inside each token.

---

### 2.11 Bottom-Up Knowledge Structure of Transformer

Today I also organized Transformer as a layered knowledge tree:

1. numbers / vectors / matrices
2. linear algebra basics
3. neural network basics: linear layer, activation, MLP
4. sequence representation: token, token id, embedding, position
5. attention mechanism
6. self-attention and multi-head attention
7. mask mechanism
8. Transformer block
9. encoder / decoder / cross-attention
10. three architectures:
    - encoder-only
    - encoder-decoder
    - decoder-only
11. training objectives:
    - next-token prediction
    - masked language modeling
    - seq2seq generation
12. modern model families:
    - BERT
    - T5
    - GPT
    - LLaMA

This helped form a clearer roadmap instead of learning concepts in isolation.

---

### 2.12 What Is LLaMA

LLaMA is a family of large language models from Meta.

At the architecture level, LLaMA is essentially:

- a **Decoder-Only Transformer**
- trained with **autoregressive next-token prediction**

So when studying LLaMA, it can be seen as:

> the Decoder-Only Transformer route implemented at large scale

---

### 2.13 LLaMA Embedding Layer

LLaMA embedding layer mainly does one job:

> convert discrete token IDs into continuous vectors

If:

- vocabulary size = `V`
- hidden size = `H`

Then embedding matrix shape is:

```python
[V, H]
```

For each input token id, the model looks up one row of that matrix.

Important distinction learned today:

- **token embedding**: converts token id to vector
- **position information** in LLaMA is not handled by a standard absolute position embedding table
- LLaMA mainly uses **RoPE** to inject positional information in attention computation

So the embedding layer itself mainly handles **token representation**, not full contextual understanding.

Best memory sentence:

> Embedding gives each token an initial vector; context understanding happens later in attention layers.

---

## 3. One Mini Numeric Example of Encoder-Decoder Attention

To make cross-attention concrete, here is the simplified example studied today.

Assume:

```math
Q=
\begin{bmatrix}
1 & 0 \\
1 & 2
\end{bmatrix}
```

```math
K = V =
\begin{bmatrix}
1 & 0 \\
0 & 1 \\
1 & 1 \\
0 & 0
\end{bmatrix}
```

The 4 source positions correspond to:

1. I
2. love
3. apples
4. `<pad>`

### Step 1: compute raw scores

```math
QK^T =
\begin{bmatrix}
1 & 0 & 1 & 0 \\
1 & 2 & 3 & 0
\end{bmatrix}
```

Because `d_k = 2`, divide by `sqrt(2)`:

```math
scores \approx
\begin{bmatrix}
0.71 & 0 & 0.71 & 0 \\
0.71 & 1.41 & 2.12 & 0
\end{bmatrix}
```

### Step 2: apply source padding mask

The 4th source position is `<pad>`, so it must be masked:

```math
scores' =
\begin{bmatrix}
0.71 & 0 & 0.71 & -\infty \\
0.71 & 1.41 & 2.12 & -\infty
\end{bmatrix}
```

### Step 3: softmax

Approximate attention weights:

```math
attn \approx
\begin{bmatrix}
0.40 & 0.20 & 0.40 & 0 \\
0.14 & 0.28 & 0.58 & 0
\end{bmatrix}
```

Interpretation:

- decoder position 2 attends most strongly to source position 3 (`apples`)

### Step 4: weighted sum over V

If:

```math
V=
\begin{bmatrix}
1 & 0 \\
0 & 2 \\
3 & 1 \\
0 & 0
\end{bmatrix}
```

Then outputs are approximately:

```math
o_1 = [1.60, 0.80]
```

```math
o_2 = [1.88, 1.14]
```

Meaning:

- each decoder position retrieves a weighted summary from encoder outputs
- position 2 is more influenced by the `apples` source representation

---

## 4. Most Important Takeaways of the Day

### A. Mask timing

Mask is applied **before softmax**, on the attention scores.

### B. Score matrix meaning

- rows = query positions
- columns = key positions

### C. Padding mask shape

Padding mask is reshaped to `[B, 1, 1, Lk]` so it can broadcast to `[B, heads, Lq, Lk]`.

### D. Cross-attention

- Q from decoder
- K/V from encoder
- usually only source padding mask is used

### E. Decoder-Only

Decoder-Only models are not defined by “generation” alone, but by the fact that they keep only the decoder-side masked self-attention stack.

### F. Autoregressive != Decoder-Only

Autoregression is a prediction style; Decoder-Only is an architecture.

### G. LLaMA embedding

LLaMA embedding mainly maps token ids to vectors; positional information is handled through RoPE in attention rather than classic absolute position embeddings.

---

## 5. Personal Understanding Summary

Today’s biggest progress was not just learning isolated definitions, but forming a more complete mental model:

- attention is the core interaction mechanism
- mask is a visibility rule added before softmax
- rows and columns in the score matrix have very specific meanings
- cross-attention is decoder querying encoder outputs
- Decoder-Only models are self-contained autoregressive generators
- LLaMA is one important large-scale implementation of the Decoder-Only path
- embedding is only the entry point; semantic interaction mainly happens later in attention layers

The main mindset shift today was:

> Transformer should be learned as a layered system, not as a list of disconnected terms.

---

## 6. Suggested Next Steps

- Draw the shape flow of a full self-attention computation
- Compare encoder-only, decoder-only, and encoder-decoder block structures side by side
- Trace how token embedding becomes Q/K/V in the first Transformer layer
- Study RoPE in more detail
- Study why MLP usually expands dimension first and then projects back

---

## 7. Quick Memory Cheatsheet

```text
Embedding: turn token ids into vectors
Attention: let tokens interact
Mask: control who can see whom
Padding Mask: block invalid pad positions
Causal Mask: block future positions
MLP: process each token independently after attention
Encoder: bidirectional contextual encoding
Decoder: autoregressive generation
Cross-Attention: decoder queries encoder outputs
Decoder-Only: stacked masked self-attention blocks
LLaMA: a large-scale Decoder-Only Transformer family
```

