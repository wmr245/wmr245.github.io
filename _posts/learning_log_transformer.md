# Learning Log

**Date:** Today  
**Topic:** Transformer, Decoder training mechanism, normalization, FFN, loss functions, and backpropagation

---

## 1. What I studied today

### 1.1 Batch Normalization and Layer Normalization

Today I focused on understanding two common normalization methods:

#### Batch Normalization (BN)
- Normalizes the same feature dimension across samples in a batch.
- Depends on the mean and variance of the batch.
- Commonly used in CNN-like settings.

#### Layer Normalization (LN)
- Normalizes all features inside a single sample.
- Does not depend on batch size.
- More suitable for sequence models such as RNNs and Transformers.

#### Key takeaway
- **BN** can be thought of as **column-wise normalization**.
- **LN** can be thought of as **row-wise normalization**.
- Transformer is more suitable for **Layer Normalization** because:
  - It does not rely on batch statistics.
  - It is more stable for small batch sizes.
  - It works naturally with variable-length sequences.
  - Training and inference behavior are more consistent.

---

### 1.2 FFN (Feed-Forward Network)

I learned what FFN means in Transformer.

#### Core idea
FFN stands for **Feed-Forward Network**. In Transformer, it is usually a two-layer MLP with a nonlinear activation in the middle:

```text
FFN(x) = W2 * σ(W1 * x + b1) + b2
```

#### Role in Transformer
- **Attention** lets tokens exchange information with other tokens.
- **FFN** further processes each token representation independently.

#### Key takeaway
- The same FFN parameters are shared across all positions.
- But each position is computed independently.
- Token-to-token interaction happens in **Attention**, not in **FFN**.

---

### 1.3 Linear transformation, nonlinear transformation, and expressive power

I developed a deeper understanding of why neural networks need nonlinearity.

#### Linear transformation
In deep learning, this usually refers to an affine map:

```text
y = Wx + b
```

It can perform operations like:
- scaling
- rotation
- translation
- linear combination

But it cannot represent complex curved relationships.

#### Nonlinear transformation
Examples include:
- ReLU
- GELU
- Sigmoid
- Tanh

These allow the network to model more complex functions.

#### Most important insight
If a network contains only linear layers, then stacking many layers is still equivalent to a single linear transformation.

That means:

- no real gain in expressive power
- no ability to represent complex decision boundaries

So **nonlinearity is the key reason deep neural networks can model complex patterns**.

---

### 1.4 ReLU activation function

I learned one of the most common activation functions: **ReLU**.

```text
ReLU(x) = max(0, x)
```

#### Properties
- Keeps positive values unchanged
- Sets negative values to 0
- Easy to compute
- Helps gradient propagation in deep networks

#### Main role
- Introduces **nonlinearity**
- Makes training efficient because computation is simple

---

### 1.5 Transformer Encoder and Decoder

I systematically reviewed the two core parts of Transformer.

#### Encoder
The encoder is responsible for:
- reading the input sequence
- building contextual relationships with self-attention
- producing context-aware representations for each token

##### Features
- bidirectional context modeling
- better at **understanding**
- outputs representations, not final generated text

#### Decoder
The decoder is responsible for:
- generating the output sequence step by step
- predicting the next token based on previous tokens
- using encoder outputs in encoder-decoder architectures

##### Features
- autoregressive generation
- only looks at current and previous positions
- better at **generation**

#### One-sentence summary
- **Encoder = understand**
- **Decoder = generate**

---

### 1.6 How decoder-only models take input

I also understood how decoder-only models (such as GPT-style models) receive input.

#### Input path
```text
text -> tokenizer -> token ids -> embedding vectors
```

#### Relation to encoder input
At the input construction stage, decoder-only models are very similar to encoder input:
- tokenize text
- map token ids to embeddings
- add positional information

#### Main difference
- **Encoder** uses bidirectional self-attention.
- **Decoder-only** uses masked causal self-attention.

So the embedding step is similar, but the later processing is different.

---

### 1.7 Decoder input, Teacher Forcing, and "labels shifted by one position"

This was one of the most important parts I learned today.

#### Decoder input
During training, the decoder is not directly fed the full target sequence as both input and output.

Instead, the target sequence is shifted right by one position.

Example:

Target sequence:

```text
[我, 爱, 猫]
```

Decoder input:

```text
[<BOS>, 我, 爱]
```

Target labels:

```text
[我, 爱, 猫]
```

#### Teacher Forcing
Teacher forcing means:

- during training, the decoder receives the **true previous token**
- instead of the token predicted by the model itself

#### Why this helps
- makes training more stable
- avoids early prediction errors propagating immediately
- allows parallel computation during training

#### "Shifted by one position"
This means:
- input at position `t` contains the previous token
- label at position `t` is the current token to predict

In short:

> Use the correct previous token to predict the current correct token.

---

### 1.8 Why the Decoder can be parallelized during training

This was another major breakthrough in understanding.

#### The confusion
A decoder is autoregressive, so it seems like it should only work step by step.

#### Why training can still be parallel
Because during training:
- the whole target sequence is already known
- the shifted target sequence can be fed in all at once
- a **causal mask** ensures each position only attends to earlier positions

So:
- the logical dependency is still autoregressive
- but the computation can be parallelized with matrix operations

#### Why inference is different
At inference time:
- future tokens are unknown
- the model must generate one token at a time

So:
- **training is parallel**
- **inference is usually sequential**

---

### 1.9 The essential difference between parallel training and autoregressive inference

I clarified this important point:

#### During training
The model assumes previous tokens are correct ground-truth tokens.

This is exactly what **Teacher Forcing** does.

#### During inference
There is no ground-truth next token available.

So the model must use its own previously generated outputs as input.

This leads to a known issue:

#### Exposure Bias
- During training, the model always sees correct prefixes.
- During inference, it sees its own generated prefixes.
- If an early prediction is wrong, later steps may continue from the wrong context.

---

### 1.10 Loss functions and backpropagation

I also studied the most fundamental learning pipeline in neural networks.

#### Loss function
A loss function measures how far the model prediction is from the true answer.

Examples:
- **MSE** for regression
- **Cross-Entropy Loss** for classification and language modeling

#### One-sentence understanding
> The loss function tells the model how wrong it is.

#### Backpropagation
Backpropagation computes how much each parameter contributed to the loss.

In other words, it computes gradients for model parameters.

#### One-sentence understanding
> Backpropagation tells the model how each parameter should change.

#### Parameter update
Then the optimizer updates the parameters, for example:

```text
w <- w - η * (dL/dw)
```

That is how what is learned in one training step gets stored in the model parameters.

---

## 2. Most important gains from today

### 2.1 I now understand the division of labor inside Transformer more clearly
- **Attention** handles information exchange across positions.
- **FFN** handles nonlinear processing within each position.
- **Encoder** focuses on understanding input.
- **Decoder** focuses on generating output.

### 2.2 I now understand why parallel training and autoregressive inference are not contradictory
- Training can be parallelized because the correct target prefix is known.
- Inference must be sequential because future tokens are unknown.

### 2.3 I now understand the essence of Teacher Forcing
- It does not directly carry over output text into the next training step.
- Instead, it uses correct tokens to construct training conditions.
- Then it updates model parameters through loss and backpropagation.

### 2.4 I now understand why nonlinearity is essential
- Without nonlinearity, a deep network is still just a linear model.
- Nonlinearity is what gives neural networks stronger expressive power.

---

## 3. Points I still need to strengthen

1. The detailed computation of Q, K, and V  
2. The difference between self-attention and cross-attention  
3. Why multi-head attention works so well  
4. The exact calculation of cross-entropy loss in language models  
5. How backpropagation applies the chain rule layer by layer  

---

## 4. Suggested next-step learning direction

I should continue with:

- where Q, K, and V come from
- the full matrix computation of self-attention
- how cross-attention works in encoder-decoder architecture
- a complete toy example from:
  - Embedding
  - Attention
  - FFN
  - Loss
  - Backpropagation

---

## 5. Final summary

Today I systematically reviewed many core Transformer concepts and moved from simply knowing the names to understanding how the modules cooperate.

### Main breakthroughs today
- the difference between BN and LN
- the role of FFN
- why nonlinearity matters
- the division of labor between Encoder and Decoder
- decoder input and teacher forcing
- why training can be parallel but inference is sequential
- the basic mechanism of loss functions and backpropagation

### Overall reflection
Today’s learning moved beyond memorizing concepts and into understanding mechanisms. I now have a more connected picture of how Transformer training and inference actually work.
