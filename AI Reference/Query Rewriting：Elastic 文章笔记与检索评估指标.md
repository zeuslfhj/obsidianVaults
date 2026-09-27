---
tags:
  - Search
  - QueryRewriting
  - Elasticsearch
  - RAG
source: https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve
published: 2026-01-30
reviewed: 2026-09-27
---

# Query Rewriting：Elastic 文章笔记与检索评估指标

原文：[Query rewriting strategies for LLMs and search engines to improve results](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve)，作者 Christina Nasika、Emilia Garcia Casademont。

关联：[[LLM Rewrite 资料]] · [[AI Reference/文本向量表示与检索：稠密、稀疏及相关概念|文本向量表示与检索]]

> 阅读重点：先规定检索结构，再让 LLM 提供可插入该结构的内容。评估时分清“候选集有没有相关文档”和“相关文档排得是否靠前”。下文将原文实验记录、官方资料补充、自拟实现建议分开标明，不把实验观察当作普遍保证。

## 1. 文章各节与段落群的大意

按原文顺序归纳正文；提示词、示例、表格合并到对应段落群，不逐句翻译，也不包含网站导航和推广内容。各节可从[原文目录](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve)定位。

| 正文位置 | 段落群的主旨 |
| --- | --- |
| 导言 | 缩小 LLM 在搜索中的职责，研究可控的查询增强。 |
| LLMs and search engines：开头 | 搜索能为生成提供依据；模型也能参与索引、召回、重排。 |
| 同节：后半 | 多跳、对话、Agent 搜索及合成训练数据扩展了应用范围。 |
| Query rewriting and optimization strategies | 先区分找资料与需要计算的查询。 |
| Retrieval queries | 改善文本匹配，找到能回答问题的材料。 |
| Computational queries | 聚合、计算和严格筛选需要结构化执行。 |
| Design methodology | 固定 DSL，限制模型生成的内容与用途。 |
| Query optimization strategies | 候选方法包括改述、去噪、纠错、补词、伪答案。 |
| Experiments and results：设置 | 用多个语料与语言基准衡量排序及召回。 |
| Lexical keyword enrichment：提示词比较 | 规则更复杂未必带来更好结果。 |
| 同节：例子 | 抽词能兼顾纠错、去噪和缩写展开。 |
| Pseudo-answer generation | 假设性答案或文档描述可提供检索词。 |
| Letting the model choose a method | 比较模型选策略与固定策略，并检验保留原查询的作用。 |
| Large language models versus small language models | 窄任务值得测试较小模型。 |
| Query rewriting in Elasticsearch | 用布尔查询明确准入与加分规则。 |
| Dense vector search as base retriever：前半 | 将 QR 与调优后的混合检索比较。 |
| 同节：后半 | 融合方式影响效果；测试集调参用于探索上限。 |
| First-stage retriever and reranking | 先保障候选覆盖，再优化排序。 |
| Strategy domain adaptation | 领域示例、规则及按需启用可改善适配。 |
| Conclusions / Key take-aways | 收益取决于基线、任务和模型能力。 |
| Task-focused tuning | 分阶段确定职责、指标和参数。 |
| Modern search pipelines | 在应用层组合提示词与查询模板。 |
| References / 结果表链接 | 提供进一步核验和复现入口。 |

## 2. 核心方法：Prompt + Query DSL Template

### 2.1 两种“模板”各管什么

| 层次 | 应控制的内容 | 不应混淆的地方 |
| --- | --- | --- |
| Prompt 模板 | 输出关键词还是伪答案；允许怎样补充；输出格式和数量 | 约束输出形式并不保证生成内容正确，也不保证完全确定性。 |
| DSL 模板 | 搜索字段、必须满足的条件、加分项、权重及候选窗口 | 结构由程序控制；模型可以只生成文本槽位，无须生成完整 DSL。 |

理解原文方法时，要把“让 LLM 按要求生成内容”与“由程序决定内容如何参与检索”同时考虑。原文采用这两层配合，而不只是写一句“请改写问题”。参见[设计方法章节](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve#design-methodology-template-based-expansion)。

### 2.2 一个自拟的可实施例子

以下是学习用改编，不是原文提示词、原文代码或经过验证的生产配置。

用户查询：`如何排查 PostgreSQL 慢查询？`

```text
任务：为词法检索生成少量加分词。
输入：用户查询。
要求：
1. 保留产品名、版本号、错误码和其他必要限定。
2. 选择对问题最关键的实体或术语。
3. 仅在查询缺少必要信息时补充紧密相关术语。
4. 不增加未经用户表达的时间、地域、版本或因果结论。
5. 只返回 JSON：{"terms": ["词语"]}；最多 4 项。
```

可能的输出：

```json
{"terms": ["PostgreSQL", "慢查询", "执行计划", "EXPLAIN"]}
```

应用先校验 JSON，再通过查询构造器填值：

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "text": "如何排查 PostgreSQL 慢查询？" } }
      ],
      "should": [
        { "match": { "text": "执行计划" } },
        { "match": { "text": "EXPLAIN" } }
      ],
      "minimum_should_match": 0
    }
  }
}
```

这里 `must` 决定文档必须满足的查询条件，`should` 提供额外分数。存在 `must` 或 `filter` 时，`minimum_should_match` 默认是 0；例子显式写出以免误读。只有 `should` 而没有 `must` / `filter` 时，默认通常是 1。[Elasticsearch bool 官方文档](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query)

**特别注意：`must` 包裹 `match` 不等于原句中的所有词都必须出现。** `match` 会经过分析器处理；默认词项关系、分词、同义词等都会影响匹配。严格的租户、权限、日期、状态约束应由应用根据业务规则构造过滤条件；产品编码等也要结合字段类型设计，不能仅靠 LLM 补词实现。

工程补充：可给扩展词降权，限制条数与长度，解析失败时回退原查询。这些是实现建议，不是本文验证过的最佳参数。

### 2.3 加分与扩大候选集是不同操作

若只对已有 200 篇候选重新计分，就不可能找回从未进入这 200 篇的文档；它能把原本排在第 80 的相关文档推入前 50。因此 Recall@50 可以上升，但固定候选集的 Recall@200 不会因单纯重排而上升。

同样，保留原查询为必选条件时，只匹配扩展词、完全不满足原查询条件的文档仍会被排除。若业务需要扩大覆盖面，可设计独立扩展召回再融合，但这属于另一套待评估方案。[Rescore 官方文档](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/rescore-search-results)

## 3. Experiments and results：实验记录

### 3.1 Prompt 1—5 与缩写对应

下表是实验配置索引，详细约束应回查[原文实验章节](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve#experiments-and-results)。

| 编号 | 方法 | 输出与主要限制 |
| --- | --- | --- |
| Prompt 1 | 基础抽词 | 关键词、代码词、专名；逗号列表。 |
| Prompt 2 | LKE | 只选中心实体；仅在过短且缺必要信息时扩展。 |
| Prompt 3 | 更细的抽词规则 | 少于 5 词考虑扩展，超过 10 词限制在原查询；展开缩写、补词形。 |
| Prompt 4 | PA | 生成 5 条简洁且覆盖不同方面的假设性回复。 |
| Prompt 5 | MC | 模型在抽词、伪答案、相关词扩展中选择或组合。 |

LKE 在文中有 extraction（抽取）与 enrichment（增强）两种表述；结果表中的 LKE 具体指 Prompt 2，不是两组独立实验。PA = Pseudo-Answer；MC = Model’s Choice。

原文部分提示词带有解释性输出标签。实际落地可只保留结构化结果，或增加简短的策略标签；解释文本不要混入查询加分词。Prompt 3 的长度分段也不能直接作为中文“字数”规则。

### 3.2 关键词实验的可核对数值

以下数值来自正文文本表；该表使用 BEIR 的 9 个子集：ArguAna、FiQA-2018、NQ、SciDocs、SciFact、TREC-COVID、Touché 2020、NFCorpus、Robust04。

| 指标 | 原查询 | Prompt 1 | Prompt 2 | Prompt 3 |
| --- | ---: | ---: | ---: | ---: |
| NDCG@10 | 0.346 | 0.345 | 0.356 | 0.346 |
| Recall@10 | 0.454 | 0.453 | 0.466 | 0.455 |

解读：Prompt 2 的 NDCG 绝对增加 `0.010`，换成百分制是 **1.0 个点**；相对增幅约 `0.010 / 0.346 = 2.89%`。Recall 增加 `0.012`，即 1.2 个百分点，相对约 2.64%。这是两个不同指标，不能混称“准确率提高 1%”。[原文关键词实验](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve#lexical-keyword-enrichment)

### 3.3 较小模型的单独实验

| 指标 | 原查询 | Sonnet 3.5 的 LKE | Haiku 3.5 的 LKE |
| --- | ---: | ---: | ---: |
| NDCG@10 | 0.346 | 0.364 | 0.368 |
| Recall@10 | 0.454 | 0.472 | 0.475 |

Haiku 在这张表中略高；这只能支持“值得在该窄任务上试用较小模型”，无法证明小模型普遍优于大模型。原文没有在此提供重复实验方差或显著性检验。

**数据疑点：** 此表 Sonnet 的 `0.364 / 0.472` 与前表 Prompt 2 的 `0.356 / 0.466` 不一致，正文未清楚解释差别。应分别记录，不能合并为同一次结果。[原文模型比较](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve#large-language-models-versus-small-language-models)

### 3.4 其他实验怎么读

原文报告：词法实验中 PA、MC 优于 LKE；只用生成内容替代原查询通常损害平均排序质量。向量或混合检索的收益不稳定；保留原查询并调权的一组混合实验报告 Recall@50 增加约 1—3 个百分点。这里的适用范围是该实验设置，不能外推为上线收益承诺。[原文](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve)

其余大表以图片展示，本笔记不转录未逐格核验的数字。完整结果可回查作者提供的[实验结果表](https://docs.google.com/spreadsheets/d/1kd7ToPZFwFjow3OIIwQv_-G34SDZzwYYOgTFJT76vgY/edit?gid=521501384)。注意结果表使用另一套提示词编号：抽词、伪答案、模型选择对应表中的 4、8、9，阅读时按方法名对齐，不能直接套用正文的 2、4、5。

## 4. Benchmark：基准是什么、各适合测什么

Benchmark 是评测数据和任务设置，**本身没有统一的分值区间**。它通常包含 corpus（候选语料）、queries（查询）、qrels（相关性标注）；NDCG 和 Recall 才是使用这些数据算出的指标。

| 基准 | 名称与规模口径 | 主要用途和适用场景 | 限制 |
| --- | --- | --- | --- |
| BEIR | Benchmarking IR；原论文标题为 *A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models*。本文使用 15 个英文数据集，抽词比较只用其中 9 个。 | 跨领域检索、零样本泛化；检查换到金融、科学、问答等语料后是否退化。 | 不能把“本文用 15 个”当作 BEIR 永久固定总数，也不能代表中文效果。 |
| MLDR | **Multilingual Long-Document Retrieval**，13 种语言。 | 多语言长文档检索，适合测试长文档中的局部信息是否能被找到。 | 查询根据文档内容由 GPT-3.5 生成；与真实搜索日志的噪声分布不同。 |
| MIRACL | **Multilingual Information Retrieval Across a Continuum of Languages**，18 种语言。 | 多语言检索，考察不同语言尤其资源较少语言的效果。 | 多语言基准不等于“查询一种语言、检索另一种语言”的跨语言任务；须看具体配置。 |

来源：[BEIR 官方仓库](https://github.com/beir-cellar/beir)、[MLDR 数据卡](https://huggingface.co/datasets/Shitao/MLDR)、[MIRACL 数据卡](https://huggingface.co/datasets/miracl/miracl)。

### 4.1 原文涉及的 BEIR 子集

下列是任务类型索引；它们的分数范围取决于选用指标，不能给每个数据集硬设“优秀分数线”。

| 名称 | 含义 / 任务 | 适合检查的能力 |
| --- | --- | --- |
| ArguAna | Argumentation / 论证检索，包含寻找反方论证的任务 | 主题相似不等于立场和任务相关。 |
| FiQA-2018 | Financial Opinion Mining and Question Answering；金融问答 | 金融术语与问题、答案之间的匹配。 |
| NQ | Natural Questions，自然问题 | 面向真实搜索问题的知识检索。 |
| SCIDOCS / SciDocs | 科学文献检索评测 | 论文间语义关系、相关论文发现。 |
| SciFact | 科学论断验证 | 根据论断找到支持或反驳的证据。 |
| TREC-COVID | Text REtrieval Conference 的 COVID 文献检索任务 | 专题科学检索和大量相关文献的覆盖。 |
| Touché 2020 | 论证检索评测 | 开放讨论问题、论据相关性。 |
| NFCorpus | NutritionFacts 相关医学／营养检索语料 | 用户表述与专业文献用词之间的差异。 |
| Robust04 | TREC 2004 Robust Track | 新闻语料中的稳健检索和困难查询。 |
| Quora | 重复问题检索 | 不同措辞是否表达相同问题。 |
| MS MARCO | Microsoft MAchine Reading COmprehension | 真实搜索查询对应的段落检索与排序。 |

Quora、MS MARCO 出现在原文关于查询来源的讨论中，不属于上面 9 子集提示词对比表。子集来源与链接见 [BEIR 数据集目录](https://github.com/beir-cellar/beir#available-datasets)。

### 4.2 数据来源的核验修正

原文把 MLDR 和 MIRACL 一起归为合成数据，这个说法不够准确：MLDR 的问题确实由模型生成；MIRACL 官方说明问题由母语者编写，相关性也由母语者标注。因此应区分 **真实用户日志、人工设计问题、模型合成问题**，而不是把“非日志”一律叫作“LLM 合成”。此外，原文将 MLDR 展开为 Multilingual Document Ranking；维护笔记时采用其官方名称 Long-Document Retrieval。[MLDR](https://huggingface.co/datasets/Shitao/MLDR)、[MIRACL](https://huggingface.co/datasets/miracl/miracl)

## 5. 评估指标：区间、用途、公式和例子

### 5.1 先分清三类数字

| 类别 | 例子 | 数字在回答什么问题 |
| --- | --- | --- |
| 排序时的文档分数 | BM25、向量相似度、RRF score | 这篇文档相对于其他候选该排在哪里？ |
| 离线评估指标 | NDCG@10、Recall@10、Recall@50 | 整份检索结果相对于人工标注有多好？ |
| 模型与系统规格 | tokens、参数量、价格、延迟 | 系统的容量与资源消耗是多少？ |

BM25 = 12 不能与 NDCG = 0.6 比较，也不能理解成“1200% 相关”。

### 5.2 指标速查

| 指标 | 全称 / 中文 | 标准区间 | 高分意味着什么 | 最适合的场景 |
| --- | --- | --- | --- | --- |
| NDCG@10 | Normalized Discounted Cumulative Gain；前 10 个结果的归一化折损累计增益 | `[0,1]`，越大越好 | 高相关文档出现在更靠前的位置，接近理想排序 | 搜索首页、重排器、最终排序体验 |
| Recall@10 | 前 10 个结果的召回率 | `[0,1]`，越大越好 | 标注相关文档有较高比例进入前 10 | 只消费少量文档的检索／RAG |
| Recall@50 | 前 50 个结果的召回率 | `[0,1]`，越大越好 | 更多相关文档进入可供后续重排的候选集 | 两阶段检索、给 reranker 提供 50 篇候选 |
| Precision@k（补充，非本文主要报告指标） | 前 k 个结果的精确率 | `[0,1]`，越大越好 | 返回结果中相关文档占比更大 | 结果噪声成本较高、需要少而准的输出 |

`@k` 是排名截断位置，不是阈值或百分比。上述标准范围假设相关性标注非负；没有任何相关文档的查询需要按评估器约定处理。[NDCG 实现](https://github.com/usnistgov/trec_eval/blob/main/m_ndcg_cut.c)、[Recall 实现](https://github.com/usnistgov/trec_eval/blob/main/m_recall.c)

### 5.3 Recall 的计算与上限

设 `R(q)` 为查询 q 的全部已标注相关文档，`Top_k(q)` 为检索前 k 篇：

$$
Recall@k(q)=\frac{|R(q)\cap Top_k(q)|}{|R(q)|}
$$

自拟例子：某查询有 20 篇相关文档，前 10 篇命中 6 篇，前 50 篇命中 15 篇。

- Recall@10 = `6/20 = 0.30`。
- Recall@50 = `15/20 = 0.75`。
- Precision@10 = `6/10 = 0.60`。它与 Recall 的分母不同。

对于同一排序和同一查询，Recall@50 ≥ Recall@10。Recall 不关心相关文档在前 k 内部怎样排序；第 1 名与第 49 名对 Recall@50 的命中贡献相同。

**总体区间是 `[0,1]`，但单条查询的可达上限可能低于 1：** 若有 100 篇相关文档，Recall@10 最大只能为 `10/100 = 0.10`。因此跨数据集直接比较平均 Recall，容易受每条查询的相关文档数量影响。[trec_eval Recall 定义与实现](https://github.com/usnistgov/trec_eval/blob/main/m_recall.c)

### 5.4 NDCG 的计算与具体含义

设排名 i 的文档增益为 `g_i`：

$$
DCG@k=\sum_{i=1}^{k}\frac{g_i}{\log_2(i+1)}
$$

$$
NDCG@k=\frac{DCG@k}{IDCG@k}
$$

IDCG 是把标注相关文档按增益从高到低排列所得的理想 DCG。靠后的命中会受到折损，归一化后便于比较。

**实现口径很重要：** 常见教材写 `g_i = 2^{rel_i}-1`；`trec_eval` 的 `ndcg_cut` 默认直接使用 qrels 中的相关性等级作为增益。本文使用 `pytrec_eval`，复现时应核对实际 measure、版本与 qrels，不能擅自套指数增益公式。二元标注时两种写法相同，多等级标注时可能不同。[NDCG 源码说明](https://github.com/usnistgov/trec_eval/blob/main/m_ndcg_cut.c)

自拟二元例子：只有两篇相关文档，排在第 1、第 3 位。

- DCG@3 = `1/log₂2 + 1/log₂4 = 1.5`。
- IDCG@3 = `1/log₂2 + 1/log₂3 ≈ 1.6309`。
- NDCG@3 ≈ `0.9197`。

把第 3 位的相关文档移到第 2 位，Recall@3 不变，NDCG@3 升到 1。这说明两者关注的目标不同。

NDCG = 1 表示前 k 的排序达到理想增益，不表示已经找齐全部相关文档；NDCG = 0.356 也不是“35.6% 的结果正确”。**不存在跨基准通用的 0.6 合格、0.8 优秀分界线。** 应在同数据、同标注、同 k、同评估口径下比较基线和变体。

### 5.5 聚合方式与“提高几个点”

查询级得分通常先在数据集内平均，再汇总到更大的基准。本文总平均采用先得到 BEIR、MLDR、MIRACL 各自平均，再将三个基准等权平均的口径，而不是将所有查询直接混在一起平均。[原文平均方式说明](https://www.elastic.co/search-labs/blog/query-rewriting-llm-search-improve#letting-the-model-choose-a-method)

例如三个基准分别为 `0.4、0.6、0.8`，基准等权平均为 `0.6`；即使三者查询数量不同，这个计算也不改变权重。它无法说明每种语言都提升了。

| 表达 | 从 0.50 到 0.53 的计算 | 含义 |
| --- | --- | --- |
| 绝对变化 | `0.53 - 0.50 = 0.03` | 原始指标增加 0.03 |
| 百分制点数 | `0.03 × 100 = 3` | 增加 3 个点；对 Recall 可称 3 个百分点 |
| 相对提升 | `0.03 / 0.50 = 6%` | 相对于原分数提高 6% |

维护实验记录时同时保存分子分母、平均方式、样本数量和不同语言／领域的结果。qrels 往往不是对整个语料的穷尽标注，“未标注”不必然等于真实不相关。

## 6. 其他术语与分值范围

### 6.1 查询改写与实验基础术语

| 术语 | 含义与用途 |
| --- | --- |
| QR | Query Rewriting，查询改写；可包含抽取、扩展、纠错、伪答案等。 |
| OQ / oq | Original Query，用户原始查询；`qr` 表示改写生成的内容。 |
| IR | Information Retrieval，信息检索。 |
| LLM / SLM | Large / Small Language Model，大／小语言模型；“小”是相对定位，没有统一参数量分界。 |
| RAG | Retrieval-Augmented Generation，先检索证据，再基于证据生成回答。 |
| Lexical search | 词法检索，通过词项匹配等机制评分。 |
| Entity / keyword | 实体／关键词；实体可指产品、机构、人物等，关键词也可包括一般主题词。 |
| Enrichment / expansion | 增强／扩展，加入同义词、缩写全称、相关表达以弥合词汇差异。 |
| Stemming / stem-proofing | 词干化／补充词形来缓解词干匹配不一致；应结合实际分析器，过度扩展可能增加噪声。 |
| Pseudo-answer | 假设性答案，用来形成检索线索；不能未经证据核验直接当最终回答。 |
| Baseline | 对照系统，如只用原查询的 BM25；提升多少取决于对照有多强。 |
| qrels | Query relevance judgments，查询与文档的相关性标注；是指标计算的依据。 |
| pytrec_eval | 基于 trec_eval 的 Python 检索评估工具；输入 qrels 和检索 run，输出指标。 |
| Rescoring / reranking | 都会改变排序；本文的前者主要指附加检索分数，后者指再用专门模型评分。 |
| Top-k / candidate set | 前 k 个结果／候选集；后续重排只处理已进入候选的文档。 |

这些术语自身没有“分值范围”。工具口径参见 [pytrec_eval 官方仓库](https://github.com/cvangysel/pytrec_eval)。

### 6.2 向量和混合检索章节的记号

| 记号 / 术语 | 如何阅读 | 范围与适用场景 |
| --- | --- | --- |
| BM25 | 综合词频、逆文档频率和文档长度的词法评分方法 | 通常为非负分数，没有跨查询通用的固定上限；适合词项、专名等匹配。不是概率。 |
| `bm25_oq` / `bm25_qr` | 分别用原查询／生成内容计算词法分数 | 沿用 BM25 分数量纲。 |
| `vector_oq` | 原查询编码后的向量检索结果／分数 | 范围取决于度量和引擎变换；原始余弦相似度为 `[-1,1]`，不要等同 Elasticsearch `_score`。 |
| KNN | K-Nearest Neighbors，k 近邻搜索 | k 是正整数个数，不是质量分数；用于向量候选检索。 |
| Hybrid search | 组合词法与向量检索 | 本身无固定分值区间，取决于融合器。 |
| LINEAR | 对多个检索分数加权组合 | 范围依赖输入与权重；混合不同量纲时需要关注归一化或调权。 |
| `LINEAR NDCG@10 OPTIMIZED(...)` | 选择使 NDCG@10 最大的融合权重 | 报告的评估指标仍是 `[0,1]`，不是一种新的 NDCG。 |
| `LINEAR RECALL@50 OPTIMIZED(...)` | 改为针对 Recall@50 选择权重 | 适合候选召回目标，未必同时优化首页排序。 |
| RRF | Reciprocal Rank Fusion，倒数排名融合 | 根据名次融合，减少对原始分数量纲的依赖；不是概率。 |
| Optuna / Bayesian optimization | 调参框架／根据已有试验选择下一组参数的方法 | 优化工具没有统一质量分值；优化目标才有。 |
| Train / dev / test split | 训练／开发验证／最终测试划分 | 用于拟合、选择配置和独立评估，避免测试信息泄漏。 |

RRF 的常见等权形式为：

$$
RRF(d)=\sum_{j:d\in L_j}\frac{1}{c+rank_j(d)}
$$

其中排名从 1 开始，c 为 `rank_constant`。若有 m 路列表，理论范围为 `0 ≤ score ≤ m/(c+1)`；上界要求文档在全部列表都排第一。原文示例使用两路、c = 20，因此上界约 `2/21 = 0.09524`。这个小数不表示只有 9.5% 的相关性。列表窗口及 c 会影响融合结果，因此不能把原文“RRF 不能优化”的措辞理解为 RRF 没有参数。[RRF 官方文档与公式](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion)

补充：DSL = Domain-Specific Language；ES|QL = Elasticsearch Query Language；EIS = Elastic Inference Service；MMR = Maximal Marginal Relevance，用相关性与多样性平衡减少重复结果。它们都不是本文的主评估指标。

### 6.3 模型规格不是 benchmark 成绩

| 表头 | 单位与解释 | 使用时要关注什么 |
| --- | --- | --- |
| Parameters | 参数数量；B = 十亿 | 参数量不直接等于检索质量；估计值不能当官方披露。 |
| Context window | 可处理上下文的 token 容量；K 通常表示千级 | token 不等于字或单词，实际输入输出限制由接口决定。 |
| Max output | 最多生成的 token 数 | 短列表任务通常不需要很长输出。 |
| Input / output cost | 每百万输入／输出 tokens 的计费 | 两者分别计费，还受平台、缓存、批处理等影响。 |

原文模型参数量列带有估计性质，价格表也不适合作为采购依据：例如其 Haiku 输入 `$3`、输出 `$0.80`，与 Anthropic 发布说明中 2024-12-03 更新的 `$0.80 / $4`（每百万输入／输出 tokens）不符。本笔记保留实验质量表，不沿用该规格表作为可靠事实，更不作为当前报价。[Anthropic 发布说明](https://www.anthropic.com/news/3-5-models-and-computer-use)

## 7. 如何把结论用到自己的检索系统

下面是根据指标含义推导的实践建议，不是原文做过的额外实验。

| 你的场景 | 值得比较的方案 | 主要看什么 |
| --- | --- | --- |
| 现有 BM25，查询口语化或含噪声 | 原查询与受限关键词加分 | NDCG@10、Recall@10，附加延迟 |
| 用户词汇与文档表达不同 | 同义词或伪答案增强 | Recall@k、查询漂移、领域正确性 |
| 候选交给 reranker | 根据其接收数量设置 k，如 50 | Recall@50 与重排后 NDCG@10 |
| 已有较强混合检索 | 在同一基线上做 QR 消融对照 | 增益是否值得新增调用成本 |
| 精确代码、产品型号、强过滤 | 字段规则与受控查询构造 | 必要约束是否被保留、错误匹配率 |
| 平均值、计数、分组、时间筛选 | 结构化查询或工具执行 | 执行正确性；单靠 NDCG 不够 |
| 专业或内部知识 | 检索领域示例辅助改写，或按需启用 | 分领域效果、术语准确性 |

建议保存三组对照：原查询、只用改写、原查询加改写。在同一语料、同一候选预算、同一重排器下评估，才容易判断提升来自哪里。对查询类型和语言分别统计，避免总平均掩盖局部退化。

## 8. 阅读时必须保留的限制与维护记录

1. **测试集调参不是独立测试。** 原文部分实验为了探索上限，直接在评估数据上选权重。即使只调一个参数也可能有选择偏差；上线前应在训练／验证集选配置，再用未参与调参的数据评估。
2. **文章中的数值不完全自洽。** 两张 LKE 表的 Sonnet 数值不同；暂不推断原因，复现时应追踪 run 配置。
3. **模型与基准命名需核对。** MLDR 全称、MIRACL 查询来源和模型价格表已有上述修正；这些不改变“需要做自己的评估”的判断。
4. **公式记号存在疑似笔误。** 原文介绍“向量原查询 + BM25 原查询”对照时，有一行仍写 `bm25_qr`；按该段语义应核对为 `bm25_oq`，不要机械照抄。
5. **流程与变体要区分。** 重排小节概述中写 Prompt 2 实体加分，后面的变体定义又分别引用 MC、PA。复现需要按每个变体核对，不能假定所有行都用 Prompt 2。
6. **生成答案质量没有被这些指标直接测量。** 检索指标提高不自动证明 RAG 回答更准确；还要另外验证答案依据、事实正确性与业务效果。

维护实验建议记录：阅读日期、原文版本／链接、语料及划分、模型具体版本、提示词、DSL、分析器、候选窗口、融合权重、调参数据、qrels 版本、评估库及 gain 口径、聚合方式、延迟和成本。这样以后能解释“为什么这次结果与文章不同”，而不只是保存一个分数。
