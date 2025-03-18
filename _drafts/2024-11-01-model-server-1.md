---
layout: post
title: Serving Vision models at scale [Part 1]
subtitle: understanding batching and its performance implications
tags: [ai, python, model-server, pytorch, computer-vision, torchserve]
---


Working at [qure.ai](https://www.qure.ai), I've been working with Vision models for a long time. At X-Ray team, we serve ~30 pytorch models which detect various diseases.  
I've been working on improving the performance of our models, until now I've not been able to change the model definitions itself, our RnD team gives us models which are already exported to [torchscript](https://pytorch.org/docs/stable/jit.html)  

<img class="floating-right-picture" src="/assets/fast-duck.png">

In this series of articles, I'm going to talk about my journey of improving our model inference performance, increasing the throughput of our models from ~8 images per minute to ~25 images per minute [a **3x gain**]. Both statistics are per server, here I'm solely going to focus on increasing the throughput of a single server, its trivial to scale this to as many servers as you want  
I'm hoping that after we are through with this series, you would be able to develop an intuition of the GPU performance world, its an ongoing effort for me  


In this article, we will understand **model input batching** and its implications during inference

# Background

We are going to try to maximise the inference performance of a Resnet-101 model  
*Note: A pre-trained Resnet-101 from pytorch is a ~171MB file. Models at [qure.ai](https://www.qure.ai) are much larger.*  


The model that we are going to optimise is available publicly [here at my public S3 bucket](https://narang-public-s3.s3.us-east-1.amazonaws.com/narang99-pages/models-at-scale/bees_vs_ants.pth)  
> This model takes an input tensor of shape `[batch, channels=3, height=224, width=224]` (standard resnet dimensions) and outputs a tensor of shape `[1, 2]` with structure `[[<probability-ant>, <probability-bee>]]`, you classify the input with whatever gives the higher probability  
> The model is borrowed and trained from [official pytorch transfer learning tutorial](https://pytorch.org/tutorials/beginner/transfer_learning_tutorial.html), I tweaked the notebook a bit to train a resnet-101 instead of a resnet-18  
> You don't need to go through the training part for this exercise, you can simply download the model above



## Batching

Almost every model in the wild takes an extra dimension called `batch-size`. You can basically stack `n` tensors into a single tensor, creating a batch of size `n`  
You would notice a net increase in your throughput if you increase the batch-size during inference, this will become clear with examples in this experiment  

# Setup

First setup a working conda environment, you should use a jupyter notebook for this work.  

```bash
conda create -n model-server-1 python=3.12 -y
conda activate model-server-1
conda install anaconda::notebook conda-forge::matplotlib -y

# take a look at installing pytorch for your platform here: https://pytorch.org/get-started/locally/
# mine is:
# OS: Linux
# Cuda: 12.2
# Package manager: conda
conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia
```

Download the model
```bash
wget https://narang-public-s3.s3.us-east-1.amazonaws.com/narang99-pages/models-at-scale/bees_vs_ants.pth
```
# Experiment

This is my goto strategy and code for basic benchmarking, go through the comments to understand individual bits  
**Always benchmark the base performance of your model like this, before putting it behind some server, its important to set a baseline**  

Open a jupyter notebook and add the following snippets:

```python
import torch

# load the serialized model you downloaded previously
with open("bees_vs_ants.pth", "rb") as f:
    model = torch.jit.load(f, map_location="cuda")
# this is something of a convention, always call this when you are inferencing in production
model.eval()

# a resnet takes input of size [batch, channels=3, height=224, width=224]
# this would provide a single batch for input
def get_randn_input(batch_size=1):
    return torch.randn(batch_size, 3, 224, 224)


# try the model out
# torch.no_grad is another boilerplate needed while inferencing
with torch.no_grad():
    ip = get_randn_input(1)
    o = model(ip.to("cuda"))
```

Now you need to warmup the model (run it on some input before benchmarking it)

```python
def warmup(model_, cycles=10):
    ip = get_randn_input(1)
    with torch.no_grad():
        for _ in range(cycles):
            op = model_(ip.to("cuda"))

warmup(model, 10)
```


Running a single inference on a given input and returning the output
```python
def single_inference(model_, input_tensor_on_cpu):
    with torch.no_grad():
        on_cuda = input_tensor_on_cpu.to("cuda")
        op = model_(on_cuda)
        return op.cpu()
```

The above function is typically the hottest code path at [qure.ai](https://www.qure.ai)  
We always get a CPU tensor input, transfer it to GPU, run the model (already loaded on GPU), transfer the output back to CPU  
These are the **bare minimum steps** that your inference thread would need to run for any model  


Benchmarking this is really simple in jupyter notebook
```python
# first create the input, we dont want it to be the part of benchmark
ip = get_randn_input(batch_size)
%timeit single_inference(model, ip)
```

Change batch size to different numbers and collect your statistics, heres an example run:

```python
ip = get_randn_input(2)
%timeit single_inference(model, ip)

# output: 3.89 ms ± 1.28 μs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```

Note down these findings in a simple table


<img class="floating-right-picture" src="/assets/honking-goose.png">
# Findings


This is how the model runs look on my machine, I'm using an [NVIDIA GeForce RTX 3090 GPU](https://www.nvidia.com/en-in/geforce/graphics-cards/30-series/rtx-3090-3090ti/)

| batch-size | total time (ms) | time per tensor = total time / batch-size |
| ----- | ---------- | ---------- |
| 1 | 2.75 | 2.75 |
| 2 | 3.89| 1.945 |
| 4 | 5.88 | 1.47 |
| 8 | 10.7 | 1.33 |
| 16 | 19.4 | 1.21 |
| 32 | 37.3 | 1.16 |
| 64 | 70.5 | 1.10 |
| 128 | 136 | 1.06 |
| 256 | 262 | 1.02 |

- **Increasing batch size increases the latency of EVERY input tensor, but the latency does not increase linearly (it does not perform as if you ran it in a simple loop)**
  - A batch of size `1` gives us result in `2.75ms`, but a batch of size `32` gives all the results in `37.3ms`
  - Our overall latency increased for all the inputs increased
  - but consider how long it would take to get outputs of 32 inputs, one at a time: `32 * 2.75 = 88ms`
  - This is at least double the time taken by a batched 32 tensor batch
  - **Essentially, we have more than doubled the throughput of our inference function `single_inference`, but the latency of a single tensor has degraded a lot**
- **This tells us that Batch inferencing has an inherent trade-off between throughput and latency**, you need to find the soft spot (which batch size do you want?)
  - Always have this table with you, and **make sure you run this experiment on different GPUs (the difference can be massive on newer GPUs, its always best to run it on the GPU you will use on production)**
  - Your batch size would eventually depend on your environment and request load

If you are lucky, batching can 2x your throughput (as it has in this case)  
As a rule, I always do this exercise, playing around with the model I need to deploy, before even thinking of optimising it  

Now, how do you batch your inputs? The solution we chose lies in [Model Servers like TorchServe](https://github.com/pytorch/serve), I'll write about them in the next article
