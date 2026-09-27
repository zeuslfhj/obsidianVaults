---
tags:
  - RAG
  - QueryRewriting
  - Retrieval
source: https://aws.amazon.com/blogs/machine-learning/from-rag-to-fabric-lessons-learned-from-building-real-world-rags-at-genaiic-part-1/
published: 2024-10-24
reviewed: 2026-09-27
---

# AWS RAG Part 1：阅读笔记

来源：[From RAG to fabric: Lessons learned from building real-world RAGs at GenAIIC – Part 1](https://aws.amazon.com/blogs/machine-learning/from-rag-to-fabric-lessons-learned-from-building-real-world-rags-at-genaiic-part-1/)，Aude Genevay，AWS，2024-10-24。

关联：[[LLM Rewrite 资料]] · [[AI Reference/Query Rewriting：Elastic 文章笔记与检索评估指标|Elastic 查询改写与评估笔记]]

## 1. 文章想说明什么

**可靠的文本 RAG，需要同时管理检索证据的相关性、完整性与生成答案的依据。** 作者从实际项目出发，强调先定位失败原因，再选择优化手段；查询改写只是其中一环。Part 1 聚焦文本，结构化数据和图片留给 Part 2，不能仅凭标题将本篇理解为完整的多模态架构方案。

### 正文结构速览

| 章节 | 主旨 |
| --- | --- |
| Anatomy of RAG | 解释入库、检索、上下文增强、生成。 |
| Implementation on AWS | 区分托管能力与自定义流程。 |
| Overview of use cases | 用客服、手册、产品和新闻等任务说明需求差异。 |
| Evaluating a RAG solution | 分别评价检索与生成，结合指标及人工判断。 |
| Troubleshooting RAG | 区分漏召回、噪声过多、上下文残缺和生成错误。 |
| Practical guide：Hybrid search | 结合语义与词项匹配。 |
| Adding metadata | 为切分后的文本恢复所属文档的信息。 |
| Small-to-large / Section-based chunking | 兼顾定位精度与上下文完整性。 |
| Rewriting / Metadata filtering | 将问题转成检索输入及过滤条件。 |
| Training custom embeddings | 常规优化不足后再考虑训练。 |
| Improving reliability | 约束回答依据，用引用支持核验。 |
| Conclusion | 根据评估迭代整条流水线。 |

以上为[原文](https://aws.amazon.com/blogs/machine-learning/from-rag-to-fabric-lessons-learned-from-building-real-world-rags-at-genaiic-part-1/)的简要归纳。下文围绕 chunk 上下文、关键词与术语提取、改写用途三个阅读重点展开，结合原文核验与自拟示例，按入库、检索和生成阶段组织。

## 2. 入库：为 chunk 补充文档标题与上下文

给 chunk 补充标题有助于匹配，但关键是把**所属文档的名称或标题**带入各 chunk，恢复“这段内容属于哪个对象”的信息，并非必须为每个 chunk 生成新标题。对应原文的 Adding metadata information to text chunks 一节。

以下使用自拟产品示例解释。

```text
文档标题：A17 电机维护手册
章节：润滑周期
chunk 正文：每运行 600 小时补充润滑脂。
```

如果只索引正文，搜索“A17 的润滑周期”时，系统可能难以区分它与其他设备的相似维护要求。可以把送入检索表示的文本构造成：

```text
文档：A17 电机维护手册
章节：润滑周期
正文：每运行 600 小时补充润滑脂。
```

这里的信息来自原文结构，不是模型凭空补充。对于短而依赖上文的 chunk，产品名、章节名等能帮助恢复语境；泛泛的标题却未必有用。

### 要区分两种 metadata 用法

| 用法 | 放在哪里 | 作用 |
| --- | --- | --- |
| 上下文进入检索文本 | 标题、章节等拼进实际索引／embedding 的文本 | 参与文本或向量表示，帮助匹配。 |
| 元数据作为独立字段 | `document_id`、`product_id`、`section` 等 | 用于过滤、取邻接块、定位来源。 |

只把标题存入一个没有参与搜索的展示字段，不会自动改变正文 embedding。反过来，把产品名拼进正文，也不等于实现了精确的产品过滤。两种机制可以配合，但不能互相替代。

此外，标题能补充“这段讲谁”，却不能补回被切断的步骤、条件或例外说明。如果命中的 chunk 只包含操作步骤的前半部分，应考虑章节切分或取回相邻／父级文本。判断是否需要扩展上下文，要依据任务而非固定追求大 chunk。

## 3. 查询改写：按用途拆分检索文本、关键词与实体

Rewrite 的目标是保留有效检索信息，而非增加关键词数量。原文 Rewriting the user query 一节用三个字段分别服务于语义检索、词法检索和实体过滤：`rewritten_query`、`keywords`、`product_name`。其中 `keywords` 明确不包含产品名；属性术语与用于限定范围的实体分开处理。

自拟用户问题：

> A17 电机多久补一次润滑脂？请用三条要点回答。

可设计如下结构化结果；其中 `answer_format` 是本笔记的工程补充，不是 AWS 原文的输出字段。

```json
{
  "rewritten_query": "A17 电机 润滑脂 补充周期",
  "keywords": ["润滑脂", "补充周期"],
  "product_name": "A17",
  "answer_format": "三条要点"
}
```

| 字段 | 后续处理 | 避免的混淆 |
| --- | --- | --- |
| `rewritten_query` | 计算查询 embedding，用于语义检索 | 它是检索表述，不是最终答案。 |
| `keywords` | 构造词法检索部分 | 只选有效词，不追求更多词。 |
| `product_name` | 先解析为可信实体，再构造过滤条件 | 抽取值不是已验证事实；不能随意猜型号。 |
| `answer_format` 或保留的原始问题 | 传给回答生成阶段 | 从检索输入中删去格式要求，不等于从整个任务中丢弃要求。 |

应用层还应校验字段类型、缺失值及实体是否明确。无实体时可用 JSON `null`，不要让模型编造产品；存在多个可能实体时，不应把不确定结果直接变成硬过滤。精确过滤会提高针对性，也可能因抽错实体而漏掉全部正确文档。

可用一条处理链理解：

```text
原始问题
  → 提取检索文本、关键词与实体
  → 校验并构造查询
  → 检索与筛选候选
  → 必要时补上下文、重排
  → 将证据连同原始问题交给生成模型
  → 核验答案与引用
```

“为了搜索”还不够具体，提示词最好明确：哪个字段用于向量检索，哪个用于词法检索，哪个会成为必须满足的约束。

## 4. 故障定位与效果评估

下面是结合文章问题分类整理的学习框架，表中的诊断问题与验证方式为本笔记的分析建议。

| 观察到的问题 | 先问自己 | 可以验证的改动 |
| --- | --- | --- |
| 根本没有找到答案证据 | 是实体词匹配失败，还是查询表达与文档不同？ | 对比词法、语义及混合检索；检查改写前后的命中。 |
| 找到相关片段，但答案缺步骤 | 是切分破坏了完整语义吗？ | 对比独立 chunk、相邻块、整节文本。 |
| 同类产品信息混入 | 产品约束是只加分还是硬过滤？ | 检查候选是否全部满足实体条件。 |
| 证据齐全，回答仍出错 | 是否误解证据、串用实体或忽略条件？ | 单独评估生成结果，而不是继续增加召回量。 |
| 答案带引用，看起来可信 | 引文是否存在，且真的支持该结论？ | 分别核对引用出处与论断支持关系。 |

### 评估时不要只看“最终答得像不像”

- **命中**：前 k 条至少有一个相关结果吗？这对应文章的 Top-k accuracy，可理解为 Hit@k。
- **顺序**：第一条相关结果排在哪里？MRR 对每个查询取首个相关结果名次的倒数，再平均；它不衡量后续全部证据的覆盖。
- **覆盖与噪声**：Recall 与 Precision 分别关注漏掉多少相关材料、带入多少无关材料。
- **生成**：证据是否支持答案，是否完整遵循用户要求？这需要与检索指标分开评估。

这些标准检索指标通常在 `[0,1]`，越高越好。只有一篇标注相关 chunk 时，单查询 Hit@k 和 Recall@k 一样；多篇相关 chunk 时，命中一篇不代表证据已齐全。更多指标解释见 [[AI Reference/Query Rewriting：Elastic 文章笔记与检索评估指标#5. 评估指标：区间、用途、公式和例子|检索评估指标笔记]]。

## 5. 实现注意与原文核验

### 5.1 产品名“强制过滤”与代码不一致

原文 Metadata filtering 示例文字与注释描述了强制匹配产品名，但代码将 `match_phrase` 加入的是 `bool.should`，随后语义和词法查询也加入 `should`。这样不能保证所有返回结果都命中产品名：文档可能仅满足其他子句。

按 OpenSearch 布尔查询语义，必须满足的条件应放到 `must` 或 `filter`；`filter` 适用于不需要参与相关性评分的约束。仅包含 `should` 时，默认要求命中其中至少一个，也不等于必须命中特定的产品名子句。[OpenSearch bool 文档](https://docs.opensearch.org/latest/query-dsl/compound/bool/)

### 5.2 `match_phrase` 不等于字段值完全相等

`match_phrase` 匹配分析后的短语；一段更长的字段内容也可能包含该短语。如果要求唯一产品一致，更适合将可信产品 ID 或规范化名称存为 `keyword` 类型，并用 `term` 做精确值匹配。字段的 normalizer、大小写与别名映射也应保持一致。[match_phrase 文档](https://docs.opensearch.org/latest/query-dsl/full-text/match-phrase/)、[term 文档](https://docs.opensearch.org/latest/query-dsl/term/term/)

下面仅示意过滤部分，不是完整混合检索请求：

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "product_id": "motor-a17" } }
      ]
    }
  }
}
```

前提是 `product_id` 已映射为适合精确匹配的字段，而且 `motor-a17` 是经过解析确认的 ID。

### 5.3 提取了 keywords，不代表后续代码真的用了它

原文最后的组合示例读取了 `json_query["keywords"]`，但词法 `match` 实际仍使用原始 `query`。因此不能依据这段代码声称它已将抽取关键词用于词法分支。复现时应检查每个输出字段最终接入哪里。

### 5.4 引文存在，不等于答案一定正确

这是对原文可靠性措辞的补充判断：逐字检查可以验证引用是否出现在来源中，不能单独保证结论正确。例如原文说“A17 在高温下缩短润滑周期”，答案却借这句话断言“所有电机都缩短周期”；引文存在，但泛化错误。应分别验证：引用存在、对象一致、条件保留、结论由证据支持。

本文发表于 2024 年，模型上下文长度、微调支持、托管服务功能等属于当时背景。本笔记不把这些历史描述作为当前产品能力保证，也没有运行原文代码。

## 6. 与 Elastic 查询改写笔记的衔接

从已有学习记录看，两篇文章适合分别承担不同角色：Elastic 帮助理解“如何约束生成内容并组合检索分数”；AWS 帮助建立“从入库到生成，问题出在哪个环节”的排查顺序。

不要直接把 Elastic 中保留原查询的加分模板，当成 AWS 所有场景的唯一实现；也不要把 AWS 的结构化改写理解为自由生成整段查询代码。共同可采用的工程原则是：**让模型输出职责明确的内容，应用程序负责校验和执行，再用相应指标验证效果。**
