
# llama.cpp作为推理框架

[llama.cpp](https://github.com/ggerganov/llama.cpp)是一个纯c/c++构造的端侧llm推理框架,最早是为了能在macos上高效的跑llm而创建的,越做越好用的人越来越多支持的平台支持的硬件越来越多,现如今已经是最主流的端侧llm推理框架了.

它的主要特点有

+ 支持纯cpu推理(OpenBLAS),尤其如果cpu支持avx512效能不错
+ 支持macos(Metal)
+ 支持amd显卡(rocm/HIP),而且自己编译可以让核显启动HIP_UMA模式,使用全量内存作为显存,当然这样做会影响独显推理性能
+ 支持在windows使用Vulkan作为运算后台.这样只要是支持vulkan的显卡在window上都能顺利使用
+ 支持N卡(cuda),I卡(SYCL)
+ 支持升腾npu(CANN)和摩尔线程显卡(MUSA)
+ 支持混合推理,你可以将大模型按层分配给不同的gpu和cpu串行推理
+ 支持android端侧部署

安装也很简单,我们可以直接用`homebrew`安装

```bash
brew install llama.cpp
```

brew会下载预编译好的llama.cpp,你就可以直接用了,不过这样安装的纯cpu推理的版本.先不考虑性能,我们继续往下.

## 推理

llama.cpp使用[GGUF](https://github.com/ggerganov/ggml/blob/master/docs/gguf.md)格式的大模型,这种格式对低精度量化的模型非常友好,相对更加紧凑,而且可以包含大量元信息,非常适合在资源受限的端侧部署.这种格式的模型huggingface上有些是有现成的有些则需要自己转,我们就先用一个现成的[Qwen/Qwen2.5-Coder-0.5B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-Coder-0.5B-Instruct-GGUF/tree/main)来演示下

```bash
# 下载模型
huggingface-cli download --resume-download Qwen/Qwen2.5-Coder-0.5B-Instruct-GGUF --local-dir ~/WorkSpace/Models/Qwen/Qwen2.5-Coder-0.5B-Instruct-GGUF
# 命令行聊天
llama-cli -m ~/WorkSpace/models/Qwen/Qwen2.5-Coder-0.5B-Instruct-GGUF/qwen2.5-coder-0.5b-instruct-fp16.gguf -p "你是一个个人助理" -cnv
```

## 编译GPU版的llama.cpp

在机器上要让llama.cpp用上gpu你得编译安装.你可以像下面这样编译.

```bash
# 克隆源码到本地
git clone https://github.com/ggerganov/llama.cpp.git
# 切到最新的release分支
git checkout b4485  
# 开始编译
HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)" cmake -S . -B build -DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1100  -DCMAKE_BUILD_TYPE=Release  && cmake --build build --config Release -- -j 16
```

我是`780m`核显,因此上面是基于HIP编译的的,这里面主要要改的就是`AMDGPU_TARGETS`这个参数.
如果你打算就amd这个核显用到天荒地老也不加独显了可以考虑用下面这种方式自己编译llama.cpp.这样你就可以用全量内存当显存了.

```bash
HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)" cmake -S . -B build -DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1100 -DGGML_HIP_UMA=ON -DCMAKE_BUILD_TYPE=Release  && cmake --build build --config Release -- -j 16
```

个人还是推荐第一种编译方式,因为780m核显虽然挺不错的,但和独显还是没法比,还是预留下给独显的空间比较好.

亲测对于qwen2.5-coder-0.5b-instruct-fp16.gguf这个模型,对于8700g用核显比用cpu快约25倍

| 项目             | 8700gCPU                   | 780m                       |
| ---------------- | -------------------------- | -------------------------- |
| sampling time    | 44491.53 tokens per second | 55118.11 tokens per second |
| load time        | 265.35 ms                  | 479.18 ms                  |
| prompt eval time | 0.72 tokens per second     | 17.97 tokens per second    |
| eval time        | 53.52 tokens per second    | 53.41 tokens per second    |


+ `load time`: 模型的加载时间
+ `sample time`: prompt做tokenize的时间
+ `prompt eval time`: 预填充(`prefill`)阶段耗时
+ `eval time`: 自回归解码(`decoding`)阶段耗时

如果你是N卡独显,那你可以参考这个[链接](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md#cuda)进行编译.
