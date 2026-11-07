# Azure AI Foundry Related（第 1 段：Prompt agent + Responses API）

> 学习时的疑问和解答。对照 AWS（Bedrock / AgentCore）理解。
> 官方文档：Foundry quickstart（Set up resources / Get started with SDK），2026-11 核对。

## 0. 我做了什么（流程）

| 步骤 | 做什么 |
|---|---|
| 1 | ai.azure.com 打开 **New Foundry** 开关 |
| 2 | 建 project（同时生成 resource group） |
| 3 | Build → Models → Deploy → Deploy a base model → gpt-5-mini → Default settings（Global Standard） |
| 4 | 预算（Budgets）：只发邮件提醒，不会自动停止花钱；这次跳过 |
| 5 | Build → Agents → Create agent，名字 clair，instructions 一句话 |
| 6 | 聊两句再追问，确认记得上文 |
| 7 | Python：`azure-ai-projects` 2.x + `DefaultAzureCredential`，先调模型，再调 agent |

## 1. Azure Portal 和 Foundry 门户是一回事吗？

不是。

| | Azure Portal（portal.azure.com） | Foundry 门户（ai.azure.com） |
|---|---|---|
| 用途 | 管理资源（账单、权限、删除资源组） | 建项目、部署模型、建 agent |
| AWS 对照 | AWS Console | Bedrock 自己的 Studio |

New Foundry 开关在 ai.azure.com 顶部。

## 2. 页面上的 API key / endpoint 要复制吗？

| 项目 | 说明 |
|---|---|
| Project endpoint | 写代码时用，放环境变量，**不要写进公开仓库** |
| Azure OpenAI endpoint | 旧式入口，不用 |
| API key | 不用，用 `DefaultAzureCredential` |

## 3. 能选"无审查 / NSFW"的模型吗？

不能指望。Content filter（内容过滤器）是 **平台层**的，不是模型层的：
- Grok 等模型在 Foundry 上同样被过滤，和在 xAI 官方 API 上不同
- 调低过滤要申请，仍受 Acceptable Use Policy（可接受使用政策）约束
- 违规可能导致限流或停用订阅

这次学的是 Agent Service 的 API 和流程，换模型只是改一个参数。学习阶段用 **gpt-5-mini**：便宜，官方文档就用它。

## 4. 模型名后面的 Codex 后缀是什么？

Codex 后缀 = 专门为写代码调过的版本。

| | 无后缀 | 带 Codex |
|---|---|---|
| 定位 | 通用：聊天、问答、写作 | 编程：写、改、审代码，agent 式编码 |
| 常见限制 | 基本都能用 | 通常只支持 Responses API，看模型卡 |

我们做通用 agent，不选 Codex。

## 5. Model playground 里的 Knowledge / Memory 为什么不能改？

| 你在 | 能做什么 |
|---|---|
| Model playground（模型试玩场） | 只和模型直接聊，改 Instructions；Tools / Knowledge / Memory 只是预览 |
| Agent 页面 | 才能真正添加 Tools / Knowledge / Memory |

所以按钮写 **Save as agent**：先存成 agent，才能挂东西。
AWS 对照：先调模型，再建 AgentCore agent 才能挂 tools 和 knowledge base。
Knowledge 属于第 2 段 RAG（retrieval-augmented generation，检索增强生成）。

## 6. 为什么 agent 的 Version 是 2？

每次保存（改 instructions 或模型）都生成一个新 Version。对应代码里的 `create_version`。

## 7. 三种 agent：Prompt vs Hosted

| | Prompt agent（本次做的） | Hosted agent（托管 agent） |
|---|---|---|
| 是什么 | 模型 + instructions（可选 tools），没有自己写的服务代码 | 自己写 agent 代码，打包成容器，Foundry 托管运行 |
| 谁跑逻辑 | Foundry 平台 | 自己的代码 |
| 门户 Type 列 | Prompt | Hosted |
| AWS 对照 | Bedrock Agents（配置式） | AgentCore Runtime（部署自己的代码） |

第 2 部分的代码**不是**创建 agent，只是用 Python 调门户里已建好的 clair。
`azd`（Azure Developer CLI，Azure 开发者命令行）主要用于 hosted agent 一键部署，本次不用；本次只用 `az login` 和 Python SDK。

## 8. 脚本 1 和脚本 2

脚本 1（`01_call_model.py`）：

```python
project = AIProjectClient(endpoint=endpoint, credential=DefaultAzureCredential())
openai = project.get_openai_client()
response = openai.responses.create(model="gpt-5-mini", input="...")
```

脚本 2（`02_call_agent.py`）：

```python
openai = project.get_openai_client(agent_name="clair")
conversation = openai.conversations.create()
r1 = openai.responses.create(conversation=conversation.id, input="...")
r2 = openai.responses.create(conversation=conversation.id, input="...")
openai.conversations.delete(conversation.id)
```

| | 脚本 1 | 脚本 2 |
|---|---|---|
| 接口 | Responses API | **也是 Responses API** |
| 指向 | 模型 `model="gpt-5-mini"` | agent `agent_name="clair"` |
| 记忆 | 无 | 有（加了 conversation） |
| 门户对应 | Models 页面试玩 | Agents 页面里的 clair |

脚本 1 不是 agent，只是直接调模型。

类比：脚本 1 是直接给厨师（模型）下单；脚本 2 是通过餐厅（agent）下单，餐厅有固定规矩（instructions）和点单记录（conversation）。

**版本注意**：azure-ai-projects 2.x 用 `create_version` / `conversations` / `responses`；不要用旧版 `create_agent` / `threads` / `runs`（1.x，已是 classic）。

## 9. 为什么调用成功后，模型反问 "Which Microsoft Foundry do you mean?"

gpt-5-mini 的训练数据里没有这个新产品名，它不知道，所以反问。不是错误，反而证明：认证、endpoint、部署名、Responses API 全部正常。

## 10. 为什么 clair 记得 Mochi？conversation id 在哪来的？

- 脚本里 `conversations.create()` **当场新建**一个 conversation，id 自动返回，不需要去门户找
- 历史存在 Foundry 服务端的 conversation 里
- 每次请求带上同一个 id，平台把历史一起送给模型
- 模型本身不记东西，只看当次的 context window
- 门户里聊的 conversation 和脚本里的 conversation **互不相通**
- 证据：脚本 1 没有 conversation id，就没有记忆；换新 conversation，clair 就不认识 Mochi

## 11. `DefaultAzureCredential` 做了什么？为什么不用 API key？

| 点 | 说明 |
|---|---|
| 做什么 | 按顺序找可用身份（环境变量、managed identity、`az login` 等），换成 Entra ID（Azure 身份系统）的短期 token |
| 本次 | 找到了 `az login` 的登录状态 |
| 授权为什么通过 | 账号在项目上有角色（RBAC，基于角色的访问控制，如 Foundry User） |
| 为什么不推荐 API key | key 是长期静态密钥，泄露就能被滥用，也不知道是谁在用；token 短期有效、按人授权、有审计记录 |
| API key 能用吗 | 能，只是不推荐 |

AWS 对照：Entra ID + RBAC 像 IAM role + 临时凭证（STS）；API key 像长期 access key。

## 面试一句话

"Responses API 是统一调用接口：直接调模型没有上下文和 instructions；调 agent 加 conversation，才有 instructions 和多轮记忆。认证用 `DefaultAzureCredential` 走 Entra ID，不用 API key。"

## 收尾

- 学完要**删除资源组**（Azure Portal → 资源组 → Delete，或 `az group delete --name <资源组名> --yes --no-wait`），否则可能继续产生费用。
- endpoint 放环境变量，不写进文件、不放公开仓库。

## 12. Entra ID vs RBAC（总是搞混）

类比：进公司大楼。

| | 对应 | 回答的问题 |
|---|---|---|
| **Entra ID**（微软身份服务，旧名 Azure AD） | 门口保安查工牌 | 你是谁？（authentication，认证） |
| **RBAC**（Role-Based Access Control，基于角色的访问控制） | 工牌上写的权限 | 你能做什么？（authorization，授权） |

顺序：先 Entra ID 确认你是你，发 token（临时名牌）；再拿着 token 去 RBAC 查你在这个资源上有什么角色。

本次例子：`az login` 是 Entra ID 这一步；脚本调 Foundry 时，Azure 用 RBAC 查你在项目上有没有角色。

| 报错 | 哪一步坏了 |
|---|---|
| 401 Unauthorized | Entra ID：没登录或 token 过期，不认识你 |
| 403 Forbidden | RBAC：认识你，但没权限 |

口诀：401 不认识你，403 认识但不让你进。

核心概念：

| 概念 | 一句话 | AWS 对照 |
|---|---|---|
| Tenant（租户） | 一个组织的 Entra 目录 | 身份目录 |
| User / Group | 人 / 一批人 | IAM user / group |
| Service principal（服务主体） | 给应用或脚本用的身份 | IAM role（应用用） |
| Managed identity（托管身份） | Azure 资源自带的身份，不用管密码 | EC2/Lambda 的 instance role |
| Token（令牌） | 短期凭证，约 1 小时 | STS 临时凭证 |

容易混淆：
- Entra 角色（管目录：用户、租户，如 Global Administrator） vs Azure RBAC 角色（管资源，如 Owner、Contributor、Foundry User）
- Tenant 管身份；Subscription 管账单和资源，一个租户下可有多个订阅

面试一句话："Entra ID 负责认证，Azure RBAC 负责授权。应用在 Azure 上优先用 managed identity，本地开发用 `az login`，都通过 `DefaultAzureCredential` 统一获取 token。"

---

# 第 2 段：RAG（Retrieval-Augmented Generation，检索增强生成）

## 13. 托管 File Search（文件搜索工具）怎么给 agent 接知识库

| 步骤 | 做了什么 | AWS 对照 |
|---|---|---|
| `vector_stores.create` + 上传 | 托管索引：自动解析、切块、向量化、建关键词索引 | Bedrock Knowledge Base（知识库）+ S3 同步 |
| `agents.create_version` 挂 `FileSearchTool` | 给同名 agent 发新版本 | 更新 AgentCore 的工具配置 |
| 提问 | 看 `output` 里有没有 `file_search_call` | 看 trace 里有没有 retrieve 调用 |

- 不用建 Azure AI Search：File Search 用微软托管的搜索和存储。
- 验证方法：同一个问题，接知识库前 agent 反问或编造，接后答对，且输出项多了 `file_search_call`。
- 注意：接知识库前后问同一个资料里才有的问题，才能证明答案来自检索，不是模型自带知识。

## 14. File Search 的工作流和默认值

查询处理 4 步：改写查询 → 拆成并行搜索 → hybrid search（混合检索，关键词 + 语义）→ rerank（重排序）。

| 设置 | 默认值 |
|---|---|
| Chunk size（切块大小） | 800 tokens |
| Chunk overlap（切块重叠） | 400 tokens |
| 嵌入模型 | text-embedding-3-large（256 维） |
| 进上下文的最大块数 | 20 |

| 概念 | 作用 |
|---|---|
| Chunking（切块） | 把长文拆成可检索的小段；重叠是为了避免答案正好被切在边界上 |
| Hybrid search（混合检索） | 关键词抓字面，向量抓语义，两者互补 |
| Reranking（重排序） | 对候选块重新按相关度排序，只留最相关的 |

## 15. reranker 是黑盒

- 托管 File Search 的 reranker：文档只写 "Reranks results"，**没有公开模型**。
- 能调的只有 `score_threshold`（0 到 1，越高越严格，返回的块越少）。
- 想要可控、可解释的重排序：改用 Azure AI Search 的 semantic ranker（语义排序器）。
- AWS 对照：Bedrock Knowledge Base 可以显式选 reranker 模型，更可控；Foundry File Search 更省事但不透明。

## 16. 费用

| 项目 | 说明 |
|---|---|
| File Search 存储 | 前 1 GB 免费，超出约 $0.10/GB/天（Azure 零售价 API 查到的 North Central US 价格） |
| File Search 调用 | 按次收费，价格页显示占位符，没查到单价 |
| 学习用小文件 | 在免费存储额度内；结束后仍应删除 vector store |

## 17. 踩坑

- `create_version` 的版本号由服务端自动加 1，只增不减；恢复旧内容也会占新号。
- 版本号不要写死在文档里，以脚本输出为准。
- 模型反问"你指哪个 Whisker Club"不是 bug，是没有知识库时的正常表现。

## 第 2 段面试一句话

"我用托管 File Search 给 prompt agent 接知识库：上传文件到 vector store，用 `create_version` 发布挂了工具的新版本，并用输出里的 `file_search_call` 验证真的检索了。它默认切块 800 tokens、重叠 400，做 hybrid search 加 rerank，但 reranker 是黑盒，只能调 `score_threshold`；需要可控排序时，我会换 Azure AI Search，或者像 Bedrock 那样显式选 reranker。"
