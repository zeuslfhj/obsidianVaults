Semantic Router: 它是一个使用embedding来进行router选择的一个仓库，它不执行query的rewrite等行为，而是将query通过embedding来进行判定，并选择合适的router

相关概念：[[AI Reference/文本向量表示与检索：稠密、稀疏及相关概念|文本向量表示与检索]]，对比稠密向量、稀疏向量及相关编码与检索方法，并解释它们在语义检索和路由中的用途。

- [[AI Reference/Query Rewriting：Elastic 文章笔记与检索评估指标|Elastic Query Rewriting 文章笔记]]：Prompt 与 DSL 模板、章节概览、实验结果、术语和指标范围，以及原文疑点核验。
- [AWS RAG](https://aws.amazon.com/blogs/machine-learning/from-rag-to-fabric-lessons-learned-from-building-real-world-rags-at-genaiic-part-1/?utm_source=chatgpt.com) 提到需要给chunk一个title or title name，让段落能够有更好的匹配。此外rewrite时候需要提供足够的关键词，首先需要能够使用keywords来进行搜索匹配，另一个需要能够extract匹配当前所需要使用到的术语，最后是需要使用的目的，例如搜索。
- [FlagEmbedding library on Hugging Face](https://huggingface.co/BAAI/bge-large-en#frequently-asked-questions) 可以看看是什么内容，主要是怎么做到embedding的

## AWS RAG 阅读补充与核对

- [[AI Reference/AWS RAG Part 1：文章整理与重点理解核对|AWS RAG Part 1 整理与理解核对]]：保留上面的原始关注，逐项核对并补充全文主旨。
- 修订理解：给 chunk 补充所属文档标题、实体等真实上下文，同时保留可过滤的元数据。Rewrite 分别生成语义检索文本、词法关键词和明确的实体过滤值，不是单纯增加关键词；格式要求保留给回答生成阶段。先判断是漏召回、噪声过多、上下文残缺还是生成错误，再选择优化手段。
- 实现注意：原文产品过滤示例将产品名条件放进 `should`，不能保证强制匹配；此外提取的 `keywords` 未实际接入该示例的词法分支，不能直接照搬代码。
