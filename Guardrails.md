# 并不存在一个统一的“Guardrail AI”

## Input Guardrail
user's input -> Guard 判定是不是安全问题/有无敏感信息/是不是禁止话题等 

## LLM's, in the system prompt

## Output Guardrail
模型生成完以后，output rail 可以决定 放行、修改或者直接拦截
并不存在一个统一的“Guardrail AI”,就越简单越直接的事情，它有时候是越快的。比如说，你直接搞一个 Word filter。
然后再往上一点，就是搞一个 safety model，一个比较小型的模型，因为不可能把世界上所有词都搞成 keyword 吧。这个模型只负责考虑说这句话有没有问题，所以比主 Language Model 肯定要更快，也没有那么大的智力需求

所以就会把 safety classifier 模型的输出，送到这个 safety model 里面，然后它会输出一个 probability，比如说针对 violence、sexual、hate 这种分类的概率值

<img width="815" height="170" alt="image" src="https://github.com/user-attachments/assets/a189a345-9154-446f-ae3c-3fa1464ab3f1" />

##  LLM-as-a-Judge OR LLM self-check / policy check。

就是在整个 model 的输出后面再加一层，送到另外一个session作为它的 input。然后由另外一个session去判断你这整个输出对不对，好不好。 同样的，如果不行就rewrite/block

## Execution / Tool Guardrail。
此外，考虑到现在的系统都有 Agent 的功能（说是 Agent，实际上扒到底下就是一个 Tool Calling 的功能），所以这个 Guardrail 又涉及到了一些需要去判断“允不允许该操作发生”的事情。这部分的guardrail是完全不用是LLM， 死的代码逻辑不管从稳定性上还是可实现性上都更好。

主模型负责说：“我想调用 transfer_money(amount=5000, to=Bob)”

Guardrail 不负责重新规划整件事，它只问几个非常机械的问题：这个工具能不能调用？→ 这个用户有没有权限？→ 参数合不合法？→ 金额有没有超限？→ 是否需要人工确认？-如果是这样的话，那很好理解，权限/参数这些都是可以用代码写死的东西

然后给结果：通过 → 真正调用工具
或者：不通过 → 拒绝 / 要求用户确认


## Retrieval Guardrail for Indirect Prompt Injection
如果系统用了 RAG 的话，你 RAG 里面要是写了什么 Jailbreak 的词，比如 prompt injection、data leakage 之类的，那这个就是 indirect prompt injection 了。
RAG Search -> Retrieval Guard -> LLM
