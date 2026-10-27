Data governance in AI agents means managing what data an AI agent can access, how it uses that data, and how to prevent misuse.

For example, suppose a company develops an AI agent connected to its internal databases, CRM, and internal documents. The agent can help employees query customer information, analyze sales data, and even modify customer records. This introduces several key challenges:

1. 权限管理
普通员工不能利用 agent 查询老板的薪资。Agent 必须严格遵守员工原本对应的数据访问权限。
2. 数据安全
当 agent 读取客户资料时，能否将身份证号、电话号码等 PII（个人身份信息）发送给外部的大模型 API？这必须根据公司的安全规范来决定。
3. 数据质量
如果 agent 读取到的是过时（stale）的数据，比如抓取到了旧版本的报价，就绝对不能直接提供给客户。系统需要具备识别数据是否过期及是否可信的能力。
4. 操作审计
在这种场景下，agent 具备修改客户记录的能力。当 agent 不小心修改了客户信息时，公司需要能够追查：它是什么时候修改的、为什么修改，以及具体修改了什么内容。
5. 数据生命周期
当客户要求删除个人信息后，agent 的记忆、缓存以及向量数据库中是否仍保留这些数据？在这方面 Postgres 做得非常好，能够切实将向量数据库中的对应内容清理干净。

为什么要在 AI agent 里面强调这个东西？因为它是跟传统软件有区别的。

在传统软件中，数据访问和操作流程本身就是在程序里提前写好的；但 AI agent 是由 LLM 根据任务动态决定要用哪些工具、查询哪些数据、甚至执行哪些操作，这就增加了很多随机性。你的 AI agent 是一个 stochastic system，正如那句话说的，在任何存在这类特性的场景里，大家殊途同归，最终的解决办法都是从一开始就不要让它有犯错的可能性。

正因为 AI 有这种自主性，意味着不能只靠 prompt 告诉 agent 不要读取敏感数据。不然 context 一长，里面的 inference 一迭代，鬼知道你第一句话写了什么、前面 prompt 写了什么。不要那么轻信，该走规章流程还是得走规章流程。

这个和 RAG 的区别在于：

1. RAG 要解决的是 agent 怎么找到需要的数据
2. Data governance 解决的是这些数据能不能给 agent、agent 怎么使用，以及使用过程是否合规

这两者一般都是配合在一起使用的。
