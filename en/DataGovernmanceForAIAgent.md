> English version of [DataGovernmanceForAIAgent.md](../DataGovernmanceForAIAgent.md). Original notes are written in Chinese/English mix.

Data governance in AI agents means managing what data an AI agent can access, how it uses that data, and how to prevent misuse.

For example, suppose a company develops an AI agent connected to its internal databases, CRM, and internal documents. The agent can help employees query customer information, analyze sales data, and even modify customer records. This introduces several key challenges:

1. Access control
A regular employee must not be able to use the agent to look up the boss's salary. The agent must strictly follow the data access permissions the employee already has.
2. Data security
When the agent reads customer records, can it send PII (personally identifiable information) such as ID numbers and phone numbers to an external LLM API? This has to be decided by the company's security policies.
3. Data quality
If the agent reads stale data, for example an old version of a quote, it must never pass it straight to a customer. The system needs to be able to tell whether data is outdated and whether it can be trusted.
4. Audit trail
In this scenario the agent can modify customer records. If the agent accidentally changes customer information, the company needs to be able to trace when it made the change, why, and exactly what it changed.
5. Data lifecycle
After a customer asks for their personal information to be deleted, is that data still kept in the agent's memory, caches, or vector database? Postgres does this very well: it can genuinely clean the corresponding content out of the vector store.

Why emphasize this for AI agents? Because they differ from traditional software.

In traditional software, data access and operation flows are written into the program ahead of time. But with an AI agent, the LLM dynamically decides which tools to use, which data to query, and even which operations to perform based on the task, which adds a lot of randomness. Your AI agent is a stochastic system. As the saying goes, in any setting with this property, all roads lead to the same place: the final solution is to make mistakes impossible from the start.

Because AI has this autonomy, you can't rely on a prompt alone to tell the agent not to read sensitive data. Otherwise, once the context gets long and the inference iterates, who knows what your first sentence or earlier prompt said. Don't be so trusting; the proper rules and processes still have to be followed.

How this differs from RAG:

1. RAG solves how the agent finds the data it needs
2. Data governance solves whether that data can be given to the agent, how the agent uses it, and whether that use is compliant

The two are usually used together.
