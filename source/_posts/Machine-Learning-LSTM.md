---
title: 'Machine Learning: LSTM'
date: 2022-01-07 09:19:33
tags:
---
# LSTM: Long Short Term Memory networks
<!-- more -->

cite [Understanding-LSTMs](http://colah.github.io/posts/2015-08-Understanding-LSTMs/) [Learning-Memory-Access-Patterns](https://www.semanticscholar.org/paper/Learning-Memory-Access-Patterns-Hashemi-Swersky/7d89abfe87ed7d1b40391d37364560656d208117)


![Core Structure of LSTM](LSTM3-chain.png)

An LSTM is composed of a hidden state $h$ and a cell state $c$, along with input $i$, forget $f$ , and output gates $o$ that dictate what information gets stored and propagated to the next timestep.

注:[ ] is for concatenate 连接的意思

1. forgot or inherit: decide what cell state information get from before.
$$f_t=\sigma\left(W_f\cdot\left[h_{t-1},x_t\right]+b_f\right)$$
2. input new: decide what new information influence the cell state.
$$i_t=\sigma\left(W_i\cdot\left[h_{t-1},x_t\right]+b_i\right)$$
$$\widetilde{C_t}=\tanh\left(W_C\cdot\left[h_{t-1},x_t\right]+b_C\right)$$
3. calculate new cell state
$$C_t=f_t*C_{t-1}+i_t * \widetilde{C_t}$$
4. decide the output/hidden state which depends on the current cell state and current input.
$$o_t=\sigma\left(W_o\cdot\left[h_{t-1},x_t\right]+b_o\right)$$
$$h_t=o_t*\tanh(C_t)$$
