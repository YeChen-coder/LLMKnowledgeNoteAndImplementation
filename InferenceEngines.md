https://www.youtube.com/watch?v=_xM8scs4_x4

<img width="786" height="471" alt="image" src="https://github.com/user-attachments/assets/99f88f52-aa77-4733-9dc9-5578f85f9dc5" />


在 2023 年之前，大家不管用什么样的方法，一般都是为了一个 general purpose（通用目的）。

在那之前最流行的其实是 PyTorch，它既可以用于训练，也可以用于 inference（推理），什么都行，是一体化的。

# llama.cpp

<img width="720" height="566" alt="image" src="https://github.com/user-attachments/assets/387ddae0-3f7d-415c-9400-161e0e097613" />

但是到了 2023 年，llama.cpp 出来了，这个东西是专门（specialized）给做 inference 用的

<img width="678" height="375" alt="image" src="https://github.com/user-attachments/assets/d717d494-9fb9-46e9-8c25-3f1828ff3ab4" />

PyTorch 那边因为有了 quantization 技术，所以其实也是在 optimize inference，但是好像还是缺一个 dependency，反正就是不如 llama.cpp 直接和方便。

<img width="861" height="473" alt="image" src="https://github.com/user-attachments/assets/4649c3d0-b7d7-47bb-ad9e-b55519d34abd" />

llama.cpp 还有一个好处是用了 memory mapping。

其实它的概念跟操作系统（Operating System）里类似机制背后的原理是一样的：因为内存不够，所以不把所有的东西一次性全往内存里塞。

在最原始的做法里，如果想让模型在 GPU 上跑，就得把整个全量模型全部塞进 GPU。但现实中大家往往没有那么大的显存，所以 llama.cpp 的设计是只把当前要用的 layer 放到 GPU，再配合 virtual memory/map 来管理，而不是必须把一整个大模型全都硬塞进显存才能跑。

# vllm

<img width="913" height="505" alt="image" src="https://github.com/user-attachments/assets/57597de3-7c1b-4def-8def-de5f1f0565f5" />

vLLM 和 llama.cpp 完全是不一样的思路。

vLLM 针对的是从头到尾整个 context window。更具体来说，它主要是对 KV cache 做优化。为什么需要优化 KV cache 呢？首先是它本身的体量非常大，而且本身并不是一个很 efficient 的东西。其他原因我真没听懂，我对这方面也确实一般，之后搞明白了会再回来补上，现在就先这样。

反正看下图，vLLM 那边的意思是它把原来一整个大的 KV cache 分成了 smaller block -我自己想，这个肯定也得跟操作系统那边联系上。因为有一些程序的存储必须是连续的，如果按照原来那么大一个完整的 KV cache，那对于存储空间以及运行来说，确实是一个很大的负担。

现在把它切成小块之后，容错性就更高了。比如说原来不管是 RAM 还是 GPU 显存，里面可能没有恰好那么大一块连续容量留给 KV cache。原本光看数字大小是够的，但真运行起来，里面指不定已经乱七八糟成什么样了。

所以当 vLLM 把原本一个大的 KV cache 分成这些 small blocks 之后，就可以直接塞进去了，因为它不再要求一块非常完整、连续的空闲存储空间

<img width="855" height="522" alt="image" src="https://github.com/user-attachments/assets/ce103366-8e91-48f3-9efa-916b9c7b896f" />

也是因为把它切成小块带来了一些好处，让它可以同时处理，有更多空间去处理不同 session 的 processing。

就比如像下图这样，大概有 6 个不同的 session，每个都要在上面跑。原本因为空间不够，只能跑完 A 再跑 B、再跑 C、再跑 D。

但现在因为它有更多可用的空间，也就是虽然你整体的 RAM 不变，但实际可用空间更好了，它就可以去做一些 continuous batching 的工作了

<img width="736" height="417" alt="image" src="https://github.com/user-attachments/assets/9fd8e802-5379-440b-8ba5-79a655250ad2" />

还有其他好处：很多 session 的 prefix 其实差不多，起码开头的那些固定 prefix（比如 system prompt）都是一样的。这些内容就可以直接被复用（不管处理这块的是叫程序还是什么）。

<img width="775" height="556" alt="image" src="https://github.com/user-attachments/assets/4746358d-f958-4aa9-a44f-8e6f8ae07e49" />

这样就省得反复在 RAM 进进出出，或者从 SSD 复制进来，少掉了来回轮换的开销。毕竟它总是 cache 命中，根据操作系统那边的机制，它也不会轻易被踢出去（因为属于 frequently used）

这个叫Radix Attention - 啊，就是那个操作系统那边什么那个命中不命中的逻辑，一样的

<img width="910" height="566" alt="image" src="https://github.com/user-attachments/assets/197a5691-4da6-4417-9182-db5716165a2b" />

# SGLang

这边就是 prefix 的东西，和之前一样，prefix 那边的 kV cache 就可以一直reuse 了。对不起啊，这边描述得确实不好，主要是因为我实在是不太懂这个。然后，这个也是 SG-Link 的一个设计原则。

<img width="915" height="471" alt="image" src="https://github.com/user-attachments/assets/45292305-c4e1-45f1-be45-6c6120a01dfb" />

# NVIDIA related 

Nvidia 毕竟是一个拥有硬件的厂商，人家很久之前就做布局了。人家那边家大业大，什么都有，从软到硬非常好打通。

所以它在这些年中就很早开始布局，也在搞自己的inference engine，去让这个生态更加适配它自己的硬件。

<img width="827" height="543" alt="image" src="https://github.com/user-attachments/assets/a6aee1c7-5bee-41a6-9c88-7e442e6e053e" />

# Overall

站在现在的角度来看，虽然这些不同的 inference engine 为了不同的 use case，或者有着不同的initivate，但大家在真实使用时遇到的问题其实都差不多，放到今天来看也差不多。

世界上的聪明人很多，而且往往会走向同一条路。这就导致大家去做优化的部分、搞的这些 feature，归根结底都是针对这些痛点来的，毕竟大家是真的都遇到了这些问题

<img width="790" height="547" alt="image" src="https://github.com/user-attachments/assets/740599f8-e658-4a71-b703-2fb148a11dac" />

