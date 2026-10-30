> English version of [InferenceEngines.md](../InferenceEngines.md). Original notes are written in Chinese/English mix.

https://www.youtube.com/watch?v=_xM8scs4_x4

<img width="786" height="471" alt="image" src="https://github.com/user-attachments/assets/99f88f52-aa77-4733-9dc9-5578f85f9dc5" />


Before 2023, whatever approach people used, it was generally for a general purpose.

The most popular thing before that was PyTorch, which could be used for both training and inference. It did everything, all in one.

# llama.cpp

<img width="720" height="566" alt="image" src="https://github.com/user-attachments/assets/387ddae0-3f7d-415c-9400-161e0e097613" />

But in 2023 llama.cpp came out, and it was specialized for inference.

<img width="678" height="375" alt="image" src="https://github.com/user-attachments/assets/d717d494-9fb9-46e9-8c25-3f1828ff3ab4" />

PyTorch, thanks to quantization, was also optimizing inference, but it still seemed to be missing a dependency. Either way, it wasn't as direct and convenient as llama.cpp.

<img width="861" height="473" alt="image" src="https://github.com/user-attachments/assets/4649c3d0-b7d7-47bb-ad9e-b55519d34abd" />

Another advantage of llama.cpp is that it uses memory mapping.

The concept is the same as the principle behind similar mechanisms in an Operating System: there isn't enough memory, so you don't load everything into memory at once.

In the most primitive approach, to run a model on a GPU you had to load the entire full model onto the GPU. In reality people often don't have that much VRAM, so llama.cpp is designed to put only the layers currently in use on the GPU, managed with virtual memory/mapping, instead of requiring the whole big model to be crammed into VRAM before it can run.

# vllm

<img width="913" height="505" alt="image" src="https://github.com/user-attachments/assets/57597de3-7c1b-4def-8def-de5f1f0565f5" />

vLLM takes a completely different approach from llama.cpp.

vLLM targets the entire context window from start to finish. More specifically, it mainly optimizes the KV cache. Why does the KV cache need optimizing? First, it's very large, and it's not a very efficient thing in itself. Honestly I didn't follow the other reasons; I'm not great at this area. I'll come back and fill this in once I understand it. For now, this is it.

Anyway, looking at the figure below, vLLM splits the one big KV cache into smaller blocks. -My own thought: this surely connects to operating systems too. Some program storage has to be contiguous, and one big, complete KV cache really is a heavy burden on storage and on running.

Once it's cut into small blocks, it's much more fault-tolerant. For example, in RAM or GPU VRAM there may not be one contiguous chunk exactly that big left for the KV cache. On paper the numbers add up, but once things are actually running, who knows how fragmented it already is.

So once vLLM splits the big KV cache into these small blocks, they can just be slotted in, because it no longer needs one complete, contiguous chunk of free storage.

<img width="855" height="522" alt="image" src="https://github.com/user-attachments/assets/ce103366-8e91-48f3-9efa-916b9c7b896f" />

Cutting it into small blocks also brings another benefit: it can process things concurrently, with more room to handle processing for different sessions.

For example, as in the figure below, there are about 6 different sessions that all need to run. Before, because there wasn't enough space, it could only run A, then B, then C, then D.

But now it has more usable space. Your total RAM doesn't change, but the actually usable space is better, so it can do continuous batching.

<img width="736" height="417" alt="image" src="https://github.com/user-attachments/assets/9fd8e802-5379-440b-8ba5-79a655250ad2" />

There are other benefits too: many sessions have nearly the same prefix, at least the fixed prefix at the start (like the system prompt) is identical. That content can be reused directly (whatever you call the thing handling this part).

<img width="775" height="556" alt="image" src="https://github.com/user-attachments/assets/4746358d-f958-4aa9-a44f-8e6f8ae07e49" />

This saves it from repeatedly moving in and out of RAM, or copying in from SSD, cutting the swapping overhead. After all, it always hits the cache, and by the operating system's mechanisms it won't easily get evicted (since it's frequently used).

This is called Radix Attention - ah, it's the same as that cache hit/miss logic from operating systems.

<img width="910" height="566" alt="image" src="https://github.com/user-attachments/assets/197a5691-4da6-4417-9182-db5716165a2b" />

# SGLang

This part is about prefixes, same as before: the KV cache for the prefix can be reused all the time. Sorry, this description really isn't good, mainly because I honestly don't understand this well. This is also one of SGLang's design principles.

<img width="915" height="471" alt="image" src="https://github.com/user-attachments/assets/45292305-c4e1-45f1-be45-6c6120a01dfb" />

# NVIDIA related 

Nvidia is a hardware vendor after all, and it started positioning itself long ago. It's a huge company with everything, so connecting software and hardware end to end is easy for it.

So over these years it started early and has been building its own inference engine, to make the ecosystem fit its own hardware better.

<img width="827" height="543" alt="image" src="https://github.com/user-attachments/assets/a6aee1c7-5bee-41a6-9c88-7e442e6e053e" />

# Overall

Looking at it now, although these different inference engines target different use cases or have different initiatives, the problems people hit in real use are about the same, and still are today.

There are plenty of smart people in the world, and they often end up on the same path. So the parts everyone optimizes and the features they build all come down to these pain points, because everyone really did run into these problems.

<img width="790" height="547" alt="image" src="https://github.com/user-attachments/assets/740599f8-e658-4a71-b703-2fb148a11dac" />
