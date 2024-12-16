# indri-benchmark

Making small indri as fast as possible

This is an effort at making small models (<1B params) run as fast as possible!
(Running models at the highest throughput possible and lowest time to fist token for the batch size of 1)

- Baseline numbers on 4090 and A100
- Why is it so slow?
  - What is the expected?
  - Why is it lower than expected?
  - How do verify your hunch?
- Special case with smaller models vs larger models
  - Why?
- How to go about speeding it up?
  - Remove dead code
  - CUDA graphs
  - Sampling with custom kernels
  - Logit processor with custom kernels
- Final results


## Baseline numbers

### 4090

Let's first evaluate what speed do we get for small models on consumer grade GPUs. For this test, I am selecting Nvidia RTX 4090 as the GPU and gpt2 (smallest size) as the model. For simplicity, I am selecting batchsize as 1. We can extend the approach for different batch sizes. We will examine two frameworks - vLLM and HuggingFace. The model run via HuggingFace is first compiled using `torch.compile`. We will examine two metrics:

1. TTFT (time to first token): We will keep the output tokens fixed and vary the number of input tokens given to the model. Lower the better.
2. Throughput (Tokens/s): We will keep the input tokens fixed and vary output tokens generated by the model. Higher the better.


![benchmark1](./profiling/performance_plots_hf_vllm.png)
Figure 1: (Left) TTFT vs # of input tokens, (Right) Decoding throughput vs # of output tokens

A couple of observations from the benchmakr run:

1. The TTFT for huggingface is unusually lower than vLLM for small number of input tokens, but a general observation is that vLLM's TTFT is much more stable and predictable than HuggingFace.
2. The decoding throughput of vLLM is ~2x of HuggingFace. This clearly shows the effor that vLLM team has put into optimizing inference for language models.

## Why is it slow?

Let's first figure out if the numbers we got are the best numbers we can get. While running the decoder only model, we are doing 2 things:

1. Computations e.g. matrix multiplacations, exponentiation, etc.. The latency of these computations are bound by FLOPS of the GPU
2. Transferring model weights to and fro from the global memory of the GPU (HBM) to L1 cache where the cores can access them. The latency of this movement is bound by the memory bandwidth of the GPU.

The end to end latency is bound by whichever is slower from the two ($Latency_{compute}$, $Latency_{memory}$)

There are two phases in autoregressive decoding:

1. Prefill phase: Where we process the prompt and generate the first token. This phase is responsible for time to first token.
2. Decoding phase: Where we generate the remaining tokens auto regressively. This phase is reponsible for throughput (output tokens/s).

Let's calculate theoretical latencies on Nvidia 4090 for GPT2 for both of these phases separately.

### Prefill phase

For prefill phase, GPT2 would consumer around: $\approx 22.4 GFlOPSs$ for a context length of 128 tokens [^A1].

Nvidia 4090 has 84 TFlops of compute for FP16. Assuming 100% utilization, we would get

$$Latency_{compute} = 22.4 GFLOPS/84 TFLOPS = 2.6 * 10^{-4}s$$

> You can use `torch.flop_counter` to get the exact FLOPs of GPT2. The one we estimate here is close enough to the actual number.

Similarly, assuming FP16 weights, the model size that the GPU would need to transfer would be $2*137*10^6 \approx 274 MB$. The memory bandwidth of Nvidia 4090 is $1008 GB/s$. We would get:

$$Latency_{memory} = 274 MB/1008 (GB/s) = 2.7 * 10^{-4}s$$

The prefill phase of the model will be bound by $min(Latency_{compute}, Latency_{memory})$ which for this case comes to be memory bound. Theoretically, we should get the latency of 0.27 ms for prefill phase (i.e. TTFT). In the above graphs, we see a latency of approximately 10x.

### Decoding phase

For decoding phase, considering a single token decoding, GPT2 would consume $\approx 0.24 GFLOPS$ [^A2]. Again, assuming 100% utilization of FP16 cores on Nvidia 4090, we would get

$$Latency_{compute} = 0.24 GFLOPS/84 TFLOPS = 2.9 * 10^{-6}s$$

The $Latency_{memory}$ would be same as prefill phase, as we would require the same model weights to decode a single token. The decode phase of the model will again be memory bound. Theoretically, we should get the latency of 0.27 ms per token. This comes to a throughput of ~3700 tok/s (~4x of vLLM)

The theoretical limits are extremely high, but we fail to see those numbers during actual runs. What can be the reason for that?

## Special case for smaller models

We can see that according to our calculations, the model is memory bound. But we have not yet taken into account the overhead of running these models. The overhead of Python and PyTroch. For larger models, the overhead of the framework is mostly hidden because of the time it takes to transfer the weights or do to the actual computations is higher than the overhead time. But for smaller models, this overhead becomes a bottleneck in the latency.

For example, here is the profile gpt2 from HuggingFace:


Figure 2: GPT2 (HuggingFace) on torch profiler

Even after using `torch.compile`, we can see clear gaps in the GPU profile, where the GPU is waiting for instructions from CPU. This is the idle time on GPU where it is neither perfoming any computation nor transferring weights. This happens because the earlier computations and transfers of the model were so fast that they got completed before the CPU had the time to issue a new instruction to the GPU. This indicates that we are in the overhead bound region.

So, how do we go about solving overhead bound? There are a few possibilities:

1. Use a language or framework which has lower overhead (e.g. c++ rather than Python). This comes with a lot of challenges. You have to learn a new language and framework and reproduce the same model in another language.
2. Trace the computation and figure out how you can send instructions from CPU at a faster rate. This is possible by using `torch.compile` and using CUDA graphs. Let's discuss this approach in the next section.

## Overhead bound

CUDA graphs is a way to record all the operations of any series of computations ahead of time, and send it as one big computational graph to the GPU. And everytime we get a new request, we can just ask the GPU to replay the same instruction set (graph). This avoids the CPU from sending these instructions again and again to the GPU and saves on the overhead time. This is a perfect usecase for the decoding phase, where the transformer model is essentially performing the same set of operations on different tokens multiple times.

However, there are a few constraints[^3] that can prevent us from building this computational graph ahead of time:

1. Dynamic sizes of intermediate tensors
2. Any synchronization between CPU and GPU

Both of these constraints apply if we are using directly using HuggingFace transformers. How?

1. KV cache
2. Sampling

## References

[^1]: [torch.compile](..)
[^2]: [CUDA graphs]
[^3]: [CUDA graph constraints](https://pytorch.org/docs/main/notes/cuda.html#constraints)


## Appendix

[^A1]: Prefill phase FLOPs
[^A2]: Decode phase FLOPs
