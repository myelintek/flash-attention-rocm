# FlashAttention

> MyelinTek Info:
>
> The MyelinTek fork adds fixes to the [ROCm official repo](https://github.com/ROCm/flash-attention/tree/flash_attention_for_rocm2)
> whose [latest update](https://github.com/ROCm/flash-attention/commit/63ce40f08451dae239b267edce38571e50bde560) has been 8 month ago.
> The applied fix is for [PyTorch 2.1+](https://github.com/ROCm/flash-attention/issues/20#issuecomment-1832147180) that no longer needs the hipify patch.
>
> This fix works for `rocm/pytorch:rocm6.0.2_ubuntu22.04_py3.10_pytorch_2.1.2`.

This repository provides the official implementation of FlashAttention and
FlashAttention-2 from the
following papers.

**FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness**  
Tri Dao, Daniel Y. Fu, Stefano Ermon, Atri Rudra, Christopher Ré  
Paper: https://arxiv.org/abs/2205.14135  
IEEE Spectrum [article](https://spectrum.ieee.org/mlperf-rankings-2022) about our submission to the MLPerf 2.0 benchmark using FlashAttention.
![FlashAttention](assets/flashattn_banner.jpg)

**FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning**  
Tri Dao

Paper: https://tridao.me/publications/flash2/flash2.pdf

![FlashAttention-2](assets/flashattention_logo.png)


## Usage

We've been very happy to see FlashAttention being widely adopted in such a short
time after its release. This [page](https://github.com/Dao-AILab/flash-attention/blob/main/usage.md)
contains a partial list of places where FlashAttention is being used.

FlashAttention and FlashAttention-2 are free to use and modify (see LICENSE).
Please cite and credit FlashAttention if you use it.

## Installation and features

Requirements:
- CUDA 11.4 and above.
- PyTorch 1.12 and above.

We recommend the
[Pytorch](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch)
container from Nvidia, which has all the required tools to install FlashAttention.

To install:
1. Make sure that PyTorch is installed.
2. Make sure that `packaging` is installed (`pip install packaging`)
3. Make sure that `ninja` is installed and that it works correctly (e.g. `ninja
--version` then `echo $?` should return exit code 0). If not (sometimes `ninja
--version` then `echo $?` returns a nonzero exit code), uninstall then reinstall
`ninja` (`pip uninstall -y ninja && pip install ninja`). Without `ninja`,
compiling can take a very long time (2h) since it does not use multiple CPU
cores. With `ninja` compiling takes 3-5 minutes on a 64-core machine.
4. Then:
```sh
pip install flash-attn --no-build-isolation
```
Alternatively you can compile from source:
```sh
python setup.py install
```

If your machine has less than 96GB of RAM and lots of CPU cores, `ninja` might
run too many parallel compilation jobs that could exhaust the amount of RAM. To
limit the number of parallel compilation jobs, you can set the environment
variable `MAX_JOBS`:
```sh
MAX_JOBS=4 pip install flash-attn --no-build-isolation
```

Interface: `src/flash_attention_interface.py`

FlashAttention-2 currently supports:
1. Ampere, Ada, or Hopper GPUs (e.g., A100, RTX 3090, RTX 4090, H100). Support for Turing
   GPUs (T4, RTX 2080) is coming soon, please use FlashAttention 1.x for Turing
   GPUs for now.
2. Datatype fp16 and bf16 (bf16 requires Ampere, Ada, or Hopper GPUs).
3. All head dimensions up to 256. Head dim > 192 backward requires A100/A800 or H100/H800.


## How to use FlashAttention

The main functions implement scaled dot product attention (softmax(Q @ K^T *
softmax_scale) @ V):
```python
from flash_attn import flash_attn_qkvpacked_func, flash_attn_func
```

```python
flash_attn_qkvpacked_func(qkv, dropout_p=0.0, softmax_scale=None, causal=False):
"""dropout_p should be set to 0.0 during evaluation
If Q, K, V are already stacked into 1 tensor, this function will be faster than
calling flash_attn_func on Q, K, V since the backward pass avoids explicit concatenation
of the gradients of Q, K, V.
Arguments:
    qkv: (batch_size, seqlen, 3, nheads, headdim)
    dropout_p: float. Dropout probability.
    softmax_scale: float. The scaling of QK^T before applying softmax.
        Default to 1 / sqrt(headdim).
    causal: bool. Whether to apply causal attention mask (e.g., for auto-regressive modeling).
Return:
    out: (batch_size, seqlen, nheads, headdim).
"""
```

```python
flash_attn_func(q, k, v, dropout_p=0.0, softmax_scale=None, causal=False):
"""dropout_p should be set to 0.0 during evaluation
Supports multi-query and grouped-query attention (MQA/GQA) by passing in KV with fewer heads
than Q. Note that the number of heads in Q must be divisible by the number of heads in KV.
For example, if Q has 6 heads and K, V have 2 heads, head 0, 1, 2 of Q will attention to head
0 of K, V, and head 3, 4, 5 of Q will attention to head 1 of K, V.

Arguments:
    q: (batch_size, seqlen, nheads, headdim)
    k: (batch_size, seqlen, nheads_k, headdim)
    v: (batch_size, seqlen, nheads_k, headdim)
    dropout_p: float. Dropout probability.
    softmax_scale: float. The scaling of QK^T before applying softmax.
        Default to 1 / sqrt(headdim).
    causal: bool. Whether to apply causal attention mask (e.g., for auto-regressive modeling).
Return:
    out: (batch_size, seqlen, nheads, headdim).
"""
```

To see how these functions are used in a multi-head attention layer (which
includes QKV projection, output projection), see the MHA [implementation](https://github.com/Dao-AILab/flash-attention/blob/main/flash_attn/modules/mha.py).

## Upgrading from FlashAttention (1.x) to FlashAttention-2

These functions have been renamed:
- `flash_attn_unpadded_func` -> `flash_attn_varlen_func`
- `flash_attn_unpadded_qkvpacked_func` -> `flash_attn_varlen_qkvpacked_func`
- `flash_attn_unpadded_kvpacked_func` -> `flash_attn_varlen_kvpacked_func`

If the inputs have the same sequence lengths in the same batch, it is simpler
and faster to use these functions:
```python
flash_attn_qkvpacked_func(qkv, dropout_p=0.0, softmax_scale=None, causal=False)
```
```python
flash_attn_func(q, k, v, dropout_p=0.0, softmax_scale=None, causal=False)
```

## Performance

We present expected speedup (combined forward + backward pass) and memory savings from using FlashAttention against PyTorch standard attention, depending on sequence length, on different GPUs (speedup depends on memory bandwidth - we see more speedup on slower GPU memory).

We currently have benchmarks for these GPUs:
* [A100](#a100)
* [H100](#h100)
<!-- * [RTX 3090](#rtx-3090) -->
<!-- * [T4](#t4) -->

### A100

We display FlashAttention speedup using these parameters:
* Head dimension 64 or 128, hidden dimension 2048 (i.e. either 32 or 16 heads).
* Sequence length 512, 1k, 2k, 4k, 8k, 16k.
* Batch size set to 16k / seqlen.

#### Speedup

![FlashAttention speedup on A100 80GB SXM5 with FP16/BF16](assets/flash2_a100_fwd_bwd_benchmark.png)

#### Memory

![FlashAttention memory](assets/flashattn_memory.jpg)

We show memory savings in this graph (note that memory footprint is the same no matter if you use dropout or masking).
Memory savings are proportional to sequence length -- since standard attention has memory quadratic in sequence length, whereas FlashAttention has memory linear in sequence length.
We see 10X memory savings at sequence length 2K, and 20X at 4K.
As a result, FlashAttention can scale to much longer sequence lengths.

### H100

![FlashAttention speedup on H100 SXM5 with FP16/BF16](assets/flash2_h100_fwd_bwd_benchmark.png)

## Full model code and training script

We have released the full GPT model
[implementation](https://github.com/Dao-AILab/flash-attention/blob/main/flash_attn/models/gpt.py).
We also provide optimized implementations of other layers (e.g., MLP, LayerNorm,
cross-entropy loss, rotary embedding). Overall this speeds up training by 3-5x
compared to the baseline implementation from Huggingface, reaching up to 225
TFLOPs/sec per A100, equivalent to 72% model FLOPs utilization (we don't need
any activation checkpointing).

We also include a training
[script](https://github.com/Dao-AILab/flash-attention/tree/main/training) to
train GPT2 on Openwebtext and GPT3 on The Pile.

## Triton implementation of FlashAttention

Phil Tillet (OpenAI) has an experimental implementation of FlashAttention in Triton:
https://github.com/openai/triton/blob/master/python/tutorials/06-fused-attention.py

As Triton is a higher-level language than CUDA, it might be easier to understand
and experiment with. The notations in the Triton implementation are also closer
to what's used in our paper.

We also have an experimental implementation in Triton that support attention
bias (e.g. ALiBi):
https://github.com/Dao-AILab/flash-attention/blob/main/flash_attn/flash_attn_triton.py


## Tests
We test that FlashAttention produces the same output and gradient as a reference
implementation, up to some numerical tolerance. In particular, we check that the
maximum numerical error of FlashAttention is at most twice the numerical error
of a baseline implementation in Pytorch (for different head dimensions, input
dtype, sequence length, causal / non-causal).

To run the tests:
```sh
pytest -q -s tests/test_flash_attn.py
```

# AMD GPU/ROCm Support
## Prerequisite
- ROCm 5.4+
- PyTorch 1.12.1+
- MI200 & MI300 GPUs
## Method 1: Build from Source
### I. Launch a ROCm PyTorch docker (recommended): E.g. 
```bash
docker run -it --device /dev/dri --device /dev/kfd --network host --ipc host --privileged --cap-add SYS_PTRACE --group-add video --security-opt seccomp=unconfined rocm/pytorch:rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1
```
### II. Clone the repo with submodules
```bash
git clone --recursive https://github.com/ROCmSoftwarePlatform/flash-attention.git
```
### III. (optional): Build for the desired GPU architecture(s) by setting the enviroment variable (semicolon seperated). We currently only support the following options. If you do not specify, defaultly it will build for your native device architecture:
To manually target for MI200 series:
```bash
export GPU_ARCHS="gfx90a"
```
To manually target for MI300 series:
```bash
export GPU_ARCHS="gfx940;gfx941;gfx942"
```
### IV. Build from source
```bash
$ cd flash-attention
$ export PYTHON_SITE_PACKAGES=$(python -c 'import site; print(site.getsitepackages()[0])')
$ patch "${PYTHON_SITE_PACKAGES}/torch/utils/hipify/hipify_python.py" hipify_patch.patch
$ pip install .
```

## Method 2: Use the Docker building script to build the Flash-Attention in one shot:
### Build and Run the Container with Flash-Attention
This command will build Flash-Attention based on rocm/pytorch:latest for the AMD GPUs detected on your machine. 
```bash
bash ./build_and_run.sh
```
### Optional Arguments:
By default, the **rocm/pytorch:latest** image will be the base image, but you can override this with any valid tags from DockerHub. For example:
```bash
tag="rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1"
```
If you want to use the nightly PyTorch from ROCm, use the version argument which will look for tags from the rocm/pytorch-nightly:
```bash
version="-nightly"
```
The script will detect your native GPU architecture for the Flash-Attention, but if you need to select a different one, pass the arguments to the script. For example:
```bash
gpu-archs="gfx90a;gfx940;gfx941;gfx942"
```
If you encountered RAM issues, you can lower the MAX_JOBS environment for ninja by:
```bash
max-jobs=4
```
Additionally, you can build the Flash-Attention in unit test mode by setting:
```bash
unit-test=true
```

### Example Command with specified configs:
The following command will build the Flash-Attention in non-unit-test mode for MI200s and MI300X with the base docker rocm/pytorch:rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1 with max-jobs=128 for ninja:
```bash
bash ./build_and_run.sh tag="rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1" gpu-archs="gfx90a;gfx941" max-jobs=128
```


By default, Flash-attention is built with optimized performance.
To run the benchmark against PyTorch standard attention: 
```bash
PYTHONPATH=$PWD python benchmarks/benchmark_flash_attention.py
```

Benchmark results(MI250, deterministic off, unit test mode off, RTZ):
```bash
PYTHONPATH=$PWD python benchmarks/benchmark_flash_attention.py 
### causal=False, headdim=64, batch_size=32, seqlen=512 ###
Flash2 fwd: 66.38 TFLOPs/s, bwd: 36.46 TFLOPs/s, fwd + bwd: 41.85 TFLOPs/s
Pytorch fwd: 15.87 TFLOPs/s, bwd: 19.43 TFLOPs/s, fwd + bwd: 18.26 TFLOPs/s
### causal=False, headdim=64, batch_size=8, seqlen=2048 ###
Flash2 fwd: 73.72 TFLOPs/s, bwd: 43.80 TFLOPs/s, fwd + bwd: 49.55 TFLOPs/s
Pytorch fwd: 18.92 TFLOPs/s, bwd: 24.36 TFLOPs/s, fwd + bwd: 22.51 TFLOPs/s
### causal=False, headdim=64, batch_size=2, seqlen=8192 ###
Flash2 fwd: 75.70 TFLOPs/s, bwd: 44.86 TFLOPs/s, fwd + bwd: 50.77 TFLOPs/s
Pytorch fwd: 19.40 TFLOPs/s, bwd: 29.32 TFLOPs/s, fwd + bwd: 25.58 TFLOPs/s
### causal=False, headdim=128, batch_size=32, seqlen=512 ###
Flash2 fwd: 61.47 TFLOPs/s, bwd: 44.31 TFLOPs/s, fwd + bwd: 48.15 TFLOPs/s
Pytorch fwd: 22.65 TFLOPs/s, bwd: 19.92 TFLOPs/s, fwd + bwd: 20.63 TFLOPs/s
### causal=False, headdim=128, batch_size=8, seqlen=2048 ###
Flash2 fwd: 74.88 TFLOPs/s, bwd: 52.49 TFLOPs/s, fwd + bwd: 57.39 TFLOPs/s
Pytorch fwd: 28.89 TFLOPs/s, bwd: 34.99 TFLOPs/s, fwd + bwd: 33.00 TFLOPs/s
### causal=False, headdim=128, batch_size=2, seqlen=8192 ###
Flash2 fwd: 78.00 TFLOPs/s, bwd: 53.70 TFLOPs/s, fwd + bwd: 58.94 TFLOPs/s
Pytorch fwd: 33.54 TFLOPs/s, bwd: 53.66 TFLOPs/s, fwd + bwd: 45.81 TFLOPs/s
### causal=True, headdim=64, batch_size=32, seqlen=512 ###
Flash2 fwd: 30.61 TFLOPs/s, bwd: 18.59 TFLOPs/s, fwd + bwd: 20.94 TFLOPs/s
Pytorch fwd: 5.85 TFLOPs/s, bwd: 9.68 TFLOPs/s, fwd + bwd: 8.16 TFLOPs/s
### causal=True, headdim=64, batch_size=8, seqlen=2048 ###
Flash2 fwd: 44.62 TFLOPs/s, bwd: 29.61 TFLOPs/s, fwd + bwd: 32.76 TFLOPs/s
Pytorch fwd: 6.28 TFLOPs/s, bwd: 12.33 TFLOPs/s, fwd + bwd: 9.67 TFLOPs/s
### causal=True, headdim=64, batch_size=2, seqlen=8192 ###
Flash2 fwd: 58.47 TFLOPs/s, bwd: 42.12 TFLOPs/s, fwd + bwd: 45.77 TFLOPs/s
Pytorch fwd: 6.38 TFLOPs/s, bwd: 14.89 TFLOPs/s, fwd + bwd: 10.78 TFLOPs/s
### causal=True, headdim=128, batch_size=32, seqlen=512 ###
Flash2 fwd: 29.54 TFLOPs/s, bwd: 23.47 TFLOPs/s, fwd + bwd: 24.93 TFLOPs/s
Pytorch fwd: 9.02 TFLOPs/s, bwd: 9.96 TFLOPs/s, fwd + bwd: 9.67 TFLOPs/s
### causal=True, headdim=128, batch_size=8, seqlen=2048 ###
Flash2 fwd: 46.04 TFLOPs/s, bwd: 36.84 TFLOPs/s, fwd + bwd: 39.07 TFLOPs/s
Pytorch fwd: 10.40 TFLOPs/s, bwd: 17.59 TFLOPs/s, fwd + bwd: 14.69 TFLOPs/s
### causal=True, headdim=128, batch_size=2, seqlen=8192 ###
Flash2 fwd: 61.03 TFLOPs/s, bwd: 51.99 TFLOPs/s, fwd + bwd: 54.29 TFLOPs/s
Pytorch fwd: 11.48 TFLOPs/s, bwd: 27.39 TFLOPs/s, fwd + bwd: 19.62 TFLOPs/s
```

## Unit Test Mode
### How to build

For passing unit tests compile flash-attention from source which may take a while:
```bash
FLASH_ATTENTION_INTERNAL_USE_RTN=1 pip install .
```

Before running unit tests, the unit test mode and deterministic flags should be both turned on by setting the environment variables:
```bash
export FLASH_ATTENTION_INTERNAL_DETERMINISTIC=1
export FLASH_ATTENTION_INTERNAL_UNIT_TEST_MODE=1
```

Run the unit tests:
```bash
pytest tests/test_flash_attn.py
```

Unit tests results(MI250, deterministic on, unit test mode on, RTN):
```bash
9119 passed, 32 skipped in 395.27s (0:06:35)
```

FlashAttention currently supports:
1. MI200 & MI300 GPUs (MI210, MI250, MI300A, MI300X).
2. fp16 and bf16.
3. Head dimensions up to 128 (e.g., 32, 40, 59, ..., 128).

## When you encounter issues

This new release of FlashAttention-2 has been tested on several GPT-style
models, mostly on A100 GPUs.

If you encounter bugs, please open a GitHub Issue!

## Citation
If you use this codebase, or otherwise found our work valuable, please cite:
```
@inproceedings{dao2022flashattention,
  title={Flash{A}ttention: Fast and Memory-Efficient Exact Attention with {IO}-Awareness},
  author={Dao, Tri and Fu, Daniel Y. and Ermon, Stefano and Rudra, Atri and R{\'e}, Christopher},
  booktitle={Advances in Neural Information Processing Systems},
  year={2022}
}
@article{dao2023flashattention2,
  title={Flash{A}ttention-2: Faster Attention with Better Parallelism and Work Partitioning,
  author={Dao, Tri},
  year={2023}
}
```
