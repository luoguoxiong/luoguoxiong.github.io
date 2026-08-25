# RAG 向量库如何选型：多维度深度对比

> 向量库是 RAG 系统的"蓄水池"，选型直接决定检索性能、可扩展性和运维成本。面对 FAISS、Milvus、Qdrant、pgvector、Chroma 等十几种选项，如何根据业务场景选对？本文从 10+ 维度系统对比主流向量库，并给出场景化选型建议。

## 一、选型前先想清楚的三个问题

在翻对比表之前，先回答三个问题，它们直接决定选型方向：

```
问题 1: 你的数据量级是多少？
        < 100万向量  →  嵌入式库够用，零运维
        100万~1亿   →  需要独立服务，关注单机性能
        > 1亿       →  必须分布式，关注水平扩展

问题 2: 你的部署环境是什么？
        纯本地/边缘  →  不能依赖云服务，优先本地库
        公有云       →  可以用托管服务（Pinecone、Weaviate Cloud）
        私有云/混合  →  需要私有化部署能力

问题 3: 你需要"过滤检索"吗？
        只做纯语义搜索     →  只需要 ANN 能力
        按时间/标签/分类搜 →  必须支持元数据过滤（Filtered Search）
        还要做关键词搜索   →  需要支持稀疏向量 / BM25 混合检索
```

这三个问题的答案，能把候选范围从 10+ 种缩小到 2~3 种。

## 二、向量库的核心技术维度

### 2.1 ANN 索引算法

向量检索的核心是**近似最近邻（ANN）**。精确最近邻（Flat / KNN）是 O(N) 的，百万级向量就扛不住了，所以生产环境都用近似算法。

| 算法     | 原理                     | 召回率 | 查询速度 | 内存占用 | 构建速度 | 动态增删 |
| -------- | ------------------------ | ------ | -------- | -------- | -------- | -------- |
| **HNSW** | 分层小世界图（多层跳表） | ★★★★★ | ★★★★★ | ★★☆☆☆（高） | ★★★☆☆ | ★★★★★ |
| **IVF**  | 倒排文件（聚类分桶）     | ★★★☆☆ | ★★★★☆ | ★★★★☆（低） | ★★★★★ | ★★★☆☆ |
| **IVF-PQ** | IVF + 乘积量化压缩    | ★★☆☆☆ | ★★★★★ | ★★★★★（极低） | ★★★★☆ | ★★☆☆☆ |
| **DiskANN** | 磁盘友好的图索引     | ★★★★☆ | ★★★☆☆ | ★★★★★ | ★★☆☆☆ | ★★★☆☆ |
| **SCaNN** | Google，量化 + 剪枝    | ★★★★☆ | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ |

**选型经验：**

- **HNSW 是当前最主流的选择**：召回率高、查询快、支持动态增删，唯一缺点是吃内存（100 万 768 维向量约占 6~8GB 内存）。
- **内存紧张用 IVF-PQ**：能把内存压缩到 1/10~1/30，但召回率下降明显，适合冷数据/海量数据场景。
- **超大规模用 DiskANN**：向量放磁盘，内存只存索引，单机可支撑十亿级向量，但查询延迟高（几十到几百 ms）。

### 2.2 HNSW 的关键参数

大多数向量库默认用 HNSW，调这三个参数直接决定性能：

```
HNSW 有三个核心参数：

  M:          每层每个节点的连接数（默认 16~32）
              ↑ M 越大 → 召回越高、内存越大、构建越慢
              ↓ M 越小 → 速度越快、召回越低

  efConstruction: 构建时搜索邻居的范围（默认 100~500）
              ↑ 越大 → 索引质量越高、构建越慢
              ↓ 越小 → 构建快、索引可能差

  ef(Search): 查询时搜索邻居的范围（运行时可调）
              ↑ 越大 → 召回越高、查询越慢
              ↓ 越小 → 查询快、召回可能低
```

**典型调参：**

| 场景           | M    | efConstruction | ef (Search) |
| -------------- | ---- | -------------- | ----------- |
| 追求极致召回   | 48~64 | 400~800       | 256~512     |
| 平衡（默认）   | 16~32 | 100~200       | 64~128      |
| 追求极致速度   | 8~12 | 50~100        | 16~32       |

> **经验法则**：先把 ef(Search) 调到能接受 Recall@K 的最小值。比如你要 Recall@5 > 0.95，先试 ef=64，不够再加到 128，以此类推。

### 2.3 元数据过滤（Filtered Search）

RAG 很少做纯语义检索，通常要加过滤：

```
"搜 2024 年之后发布的，关于 React 的文档"
       ↑ 时间过滤          ↑ 标签过滤

"只在『产品手册』这个分类里搜密码重置流程"
       ↑ 分类（元数据）过滤
```

**过滤的两种实现策略：**

| 策略         | 做法                        | 优势                 | 劣势                     | 代表       |
| ------------ | --------------------------- | -------------------- | ------------------------ | ---------- |
| **预过滤**   | 先按过滤条件缩小候选集，再 ANN | 精度高，不漏结果     | 过滤后候选太少 → 退化为线性扫描 | Qdrant、Milvus |
| **后过滤**   | 先 ANN 取 top-N，再按条件过滤   | 性能稳定             | 过滤后可能不剩结果 / 召回低 | 早期 FAISS |
| **原生过滤** | 索引中嵌入过滤结构，搜索时同步剪枝 | 兼顾精度和性能 | 实现复杂 | Qdrant（HNSW 内建 payload 索引） |

> **选型关键**：如果你的业务重度依赖过滤（比如电商、法律文书、带时间维度的知识库），Qdrant 的原生过滤是当前体验最好的。

## 三、主流向量库全景对比

我们选最常用的 7 个向量库，从 12 个维度对比。

### 3.1 基础信息总览

| 维度         | FAISS              | Milvus             | Qdrant             | pgvector           | Chroma             | Weaviate           | Pinecone           |
| ------------ | ------------------ | ------------------ | ------------------ | ------------------ | ------------------ | ------------------ | ------------------ |
| **开发方**   | Meta               | Zilliz             | Qdrant (德国)      | pgvector 社区      | Chroma (美国)      | Weaviate (荷兰)    | Pinecone (美国)    |
| **开源协议** | MIT                | Apache 2.0         | Apache 2.0         | PostgreSQL License | Apache 2.0         | BSD-3              | 闭源（SaaS）       |
| **语言实现** | C++ (Python 绑定)  | Go + C++           | Rust               | C (PG 扩展)        | Python + Rust      | Go                 | 未公开             |
| **部署形态** | 嵌入式库           | 独立服务 + 集群    | 独立服务 + 集群    | PG 扩展            | 嵌入式 / 服务端    | 独立服务 + 集群    | 纯托管 SaaS        |

### 3.2 功能维度对比

| 功能                    | FAISS | Milvus | Qdrant | pgvector | Chroma | Weaviate | Pinecone |
| ----------------------- | :---: | :----: | :----: | :------: | :----: | :------: | :------: |
| HNSW 索引               |  ✅   |   ✅   |   ✅   |    ✅    |   ✅   |    ✅    |    ✅    |
| IVF / IVF-PQ            |  ✅   |   ✅   |   ❌   |    ✅    |   ❌   |    ✅    |    ✅    |
| DiskANN                 |  ❌   |   ✅   |   ❌   |    ❌    |   ❌   |    ❌    |    ❌    |
| **元数据过滤**          |  ⚠️   |   ✅   | ✅(原生) |    ✅    |   ✅   |    ✅    |    ✅    |
| 混合检索 (BM25/稀疏)    |  ❌   |   ✅   |   ✅   |    ⚠️   |   ❌   |    ✅    |    ✅    |
| 多向量字段              |  ✅   |   ✅   |   ✅   |    ❌    |   ✅   |    ✅    |    ✅    |
| 动态增删改              |  ⚠️   |   ✅   |   ✅   |    ✅    |   ✅   |    ✅    |    ✅    |
| 持久化                  |  ⚠️   |   ✅   |   ✅   |    ✅    |   ✅   |    ✅    |    ✅    |
| 分布式/集群             |  ❌   |   ✅   |   ✅   |    ❌    |   ❌   |    ✅    |    ✅    |
| GPU 加速                |  ✅   |   ✅   |   ❌   |    ❌    |   ❌   |    ❌    |    ❌    |

> ⚠️ = 有限支持 / 需要额外实现

### 3.3 性能与扩展性

| 维度                    | FAISS  | Milvus  | Qdrant | pgvector | Chroma | Weaviate | Pinecone |
| ----------------------- | :----: | :-----: | :----: | :------: | :----: | :------: | :------: |
| **单机数据量上限**      | 千万级 | 十亿级  | 亿级   | 千万级   | 百万级 | 亿级     | 无限（托管） |
| **单机 QPS (1M 向量)**  | 高     | 很高    | 很高   | 中       | 中     | 中       | 很高（弹性） |
| **P95 查询延迟 (ms)**   | 5~20   | 5~30    | 5~25   | 10~50    | 20~80  | 15~60    | 10~50    |
| **水平扩展**            | ❌      | ✅ (分片+副本) | ✅ (分片) | ❌ | ❌ | ✅ | ✅（托管自动） |
| **内存效率**            | ★★★☆☆ | ★★★★☆ | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ | ★★★☆☆ | 托管无需关心 |
| **冷数据支持 (磁盘)**   |  ❌    | ✅(DiskANN) | ⚠️(mmap) | ✅(PG表空间) | ❌ | ⚠️ | 托管自动 |

### 3.4 运维与生态

| 维度                    | FAISS  | Milvus  | Qdrant | pgvector | Chroma | Weaviate | Pinecone |
| ----------------------- | :----: | :-----: | :----: | :------: | :----: | :------: | :------: |
| **部署难度**            | ★☆☆☆☆ | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ | ★☆☆☆☆ | ★★★★☆ | ★☆☆☆☆（零运维） |
| **监控/可观测性**        |  ❌    | ✅(Prometheus) | ✅(REST) | ✅(PG监控) | ⚠️ | ✅ | ✅(控制台) |
| **数据备份/恢复**        | 手动   | ✅(快照) | ✅(快照) | ✅(PG备份) | ⚠️ | ✅ | ✅（自动） |
| **LangChain 集成**       | ✅      | ✅      | ✅      | ✅       | ✅     | ✅       | ✅       |
| **LlamaIndex 集成**      | ✅      | ✅      | ✅      | ✅       | ✅     | ✅       | ✅       |
| **客户端 SDK 语言**      | Python | 多语言 | 多语言 | 所有SQL语言 | Python | 多语言 | 多语言 |
| **社区活跃度**           | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★★ | ★★★☆☆ | ★★★☆☆ | 闭源 |

## 四、逐个解读：每个库的"人设"

### 4.1 FAISS：纯库之王，性能天花板

```
人设: 没有感情的跑分机器
适合: 追求极致性能、有工程能力自己封装的团队
```

**优点：**
- 纯 C++ 实现，性能是所有开源库的**天花板级别**
- 算法最丰富：HNSW、IVF、IVF-PQ、LSH、Flat、GPU 加速全覆盖
- 嵌入式，零运维，想怎么嵌就怎么嵌

**缺点：**
- 没有服务端形态，要自己封装：HTTP 接口、持久化、分布式都得自己写
- 原生不支持元数据过滤，得先过滤 ID 再检索（预过滤实现麻烦）
- 没有动态增量索引（IndexIVF 新增数据需要重建或定期 merge）

**一句话**：如果你有足够的工程能力，FAISS + 自研封装是性能最优解。否则直接用上层服务。

### 4.2 Milvus：分布式扛把子，海量数据首选

```
人设: 企业级重型卡车，拉得多跑得稳
适合: 数据量大（千万+）、要集群、要高可用的生产场景
```

**优点：**
- 分布式最成熟：支持分片、副本、负载均衡，单机不够可以横向扩
- 存储分层：热数据在内存（HNSW）、冷数据在磁盘（DiskANN），成本可控
- 功能最全：BM25 混合检索、标量过滤、多向量、GPU 索引，全都有
- Attu 可视化管理后台，运维体验好

**缺点：**
- **重**：最小部署依赖 etcd + MinIO(S3) + 3 个 Milvus 组件，至少 5 个容器
- 资源消耗大：轻量场景用它是"杀鸡用牛刀"
- 运维复杂度高：调优参数多，需要学习曲线

**一句话**：知识库超过千万向量，或者需要高可用集群 → Milvus 是首选。

### 4.3 Qdrant：后起之秀，过滤检索体验最佳

```
人设: 精心调教的运动轿车，操控感一流
适合: 中大规模数据、重度依赖元数据过滤、混合检索的场景
```

**优点：**
- **元数据过滤体验最好**：HNSW 内建 payload 索引，过滤+向量同步剪枝，性能损失极小
- Rust 实现，内存安全、资源占用比 Milvus 低
- 部署简单：单 binary 或单 Docker 即可运行，最小依赖
- 支持稀疏向量（BM25），混合检索开箱即用
- API 设计优雅，SDK 体验好

**缺点：**
- 分布式能力不如 Milvus 成熟（分片可以，自动再平衡在演进）
- 生态不如 Milvus 丰富（但主流框架都支持了）
- 不支持 GPU 加速

**一句话**：单机/中小规模场景，Qdrant 在"体验/性能/运维成本"三者之间平衡最好。

### 4.4 pgvector：和业务库共存，小而美

```
人设: 全能家用SUV，啥都能装不挑路
适合: 数据量<千万、已有 PG、不想运维额外组件的场景
```

**优点：**
- **零额外运维**：就是 PostgreSQL 一个扩展，直接 `CREATE EXTENSION vector`
- 和业务数据共存：向量 + 关系数据一张表，Join 查询丝滑
- PG 的所有能力直接继承：备份、监控、权限、事务、JSONB 元数据过滤
- 小数据量（<100 万）性能不输专用向量库

**缺点：**
- 单表千万级以上性能开始下降（PG 的 MVCC + 扩展实现的限制）
- 不支持分布式，单机上限明确
- 不支持 GPU、不支持 BM25（要加 paradedb 扩展）
- HNSW 索引构建慢、占用大（比 FAISS/Milvus 慢不少）

**一句话**：PG 用户 + 数据量不大 → pgvector 真香，别折腾别的。

### 4.5 Chroma：原型神器，上手五分钟

```
人设: 轻量级滑板车，说走就走
适合: Demo、原型验证、个人项目、小体量知识库
```

**优点：**
- 嵌入 Python，5 行代码跑起来：
```python
import chromadb
client = chromadb.Client()
col = client.create_collection("test")
col.add(documents=["hello world"], ids=["1"])
col.query(query_texts=["hi"], n_results=1)
```
- 默认做了 embedding 封装，连 embedding 模型都帮你选好
- LangChain/LlamaIndex 默认支持，文档案例多

**缺点：**
- 嵌入式 Python，性能瓶颈明显，百万级以上很吃力
- 服务端形态刚出不久，生产能力未经验证
- 索引算法只有 HNSW，算法丰富度不够
- 内存占用偏高

**一句话**：做 Demo / 验证想法 → 用 Chroma 最快，上生产再换。

### 4.6 Weaviate：全栈 RAG 平台

```
人设: 精装修公寓，拎包入住
适合: 不想搭积木、要"一站式"方案的场景
```

**优点：**
- 不仅是向量库，还内置了：Embedding 模块、Rerank、BM25、LLM 集成、GraphQL 查询
- 自带模块化设计，`text2vec-openai`、`generative-openai` 直接配
- 支持混合型（向量 + 倒排 + 图）检索

**缺点：**
- 性能不如 Milvus/Qdrant 纯粹（功能多自然有取舍）
- Go 写的 HNSW 不如 C++/Rust 版本快
- 概念多、学习曲线陡
- 自定义空间相对小

**一句话**：要快速出一个带向量化的完整 RAG，不想组装多个组件 → 可以试试 Weaviate。

### 4.7 Pinecone：托管鼻祖，纯 SaaS

```
人设: 代驾司机，省心但不能自己改
适合: 不想运维任何基础设施、预算充足的团队
```

**优点：**
- **零运维**：API 直接用，不用管服务器、扩容、备份、监控
- 弹性伸缩：自动水平扩展，从 1 万到 10 亿向量无缝
- 性能稳定：SLA 保证可用性
- 功能齐全：命名空间（隔离租户）、过滤、混合检索都有

**缺点：**
- **贵**：按 pod（计算+存储单元）计费，月费几百到几千刀不等
- 数据安全：向量和元数据都存在第三方，敏感数据要评估合规
- 黑盒：无法看到底层实现，无法调内部参数
- 国内访问延迟和网络稳定性问题

**一句话**：海外业务 + 预算足 + 不想运维 → Pinecone。其他场景请三思。

## 五、决策矩阵：按场景选

### 5.1 按数据量级

```
数据量级 < 100万 向量:
  ├─ 有 PG 实例          → pgvector  ✅ （不新增任何组件）
  ├─ Python 原型/快速验证 → Chroma    ✅ （5分钟跑通）
  └─ 要高性能、自封装     → FAISS     ✅ （性能天花板）

数据量级 100万 ~ 1亿 向量:
  ├─ 重度依赖过滤检索     → Qdrant    ✅ （原生过滤最佳平衡）
  ├─ 有 PG + 不超500万   → pgvector  ✅ （继续用）
  ├─ 要混合检索(BM25)    → Qdrant / Milvus ✅
  └─ 要高可用集群         → Milvus    ✅ （分布式最成熟）

数据量级 > 1亿 向量:
  ├─ 私有部署            → Milvus 集群 ✅ （DiskANN + 冷热分层）
  ├─ 海外公有云           → Pinecone  ✅ （托管省事）
  └─ 混合云              → Weaviate 集群 ✅
```

### 5.2 按部署环境

```
纯本地 / 边缘设备 (笔记本/手机/盒子):
  → FAISS (C++, 轻量) 或 Chroma (嵌入式)
  ✗ 不要选需要独立服务的 Milvus/Qdrant（太重）

公有云 (AWS/阿里云/腾讯云):
  预算充足:    → Pinecone (托管) / Weaviate Cloud
  自己运维:    → Milvus Operator (K8s) / Qdrant Cluster
  数据<千万:   → 云数据库 PG + pgvector 扩展

私有云 / 私有化部署 (金融/政企/医疗):
  → Milvus / Qdrant 二选一
  ✗ Pinecone、Weaviate Cloud 直接排除（公有 SaaS）
  选的标准: 数据超亿级 → Milvus；否则 → Qdrant
```

### 5.3 按业务场景

| 业务场景                     | 推荐方案                        | 理由                                   |
| ---------------------------- | ------------------------------- | -------------------------------------- |
| **企业知识库问答** (10万~百万) | pgvector 或 Qdrant              | 知识更新不频繁，过滤需求中等           |
| **电商商品搜索** (千万级 SKU) | Qdrant / Milvus                 | 重度依赖属性过滤（类目、价格、品牌）   |
| **法律/医疗文档** (百万~千万) | Qdrant                          | 精确过滤 + 合规私有化                  |
| **论文/学术检索** (亿级)      | Milvus (DiskANN)                | 数据量大，冷热分层降本                 |
| **多租户 SaaS RAG**          | Qdrant (集合隔离) / Pinecone (命名空间) | 租户隔离 + 弹性扩展            |
| **对话系统短期记忆** (万级)   | Chroma / pgvector               | 小数据、快速读写                       |
| **图像/视频特征检索** (千万级) | FAISS (GPU) / Milvus (GPU 索引) | 高维向量 + 需要 GPU 加速               |

## 六、实际踩坑经验

### 坑 1：低估内存开销

```
很多人以为 100万 × 768维 × 4字节(float32) = 3GB 内存就够了。
实际上：

  HNSW 索引的额外开销:
    节点连接指针: M(16~32) × 4字节 × 层数(4~6) ≈ 向量本身的 1~2倍
    向量原始数据: 也要存（返回结果时需要）
    元数据: JSON 字段、倒排索引等

  实际内存 ≈ 原始向量大小 × (2.5 ~ 4) 倍

100万 768维向量 → 实际要 7.5GB ~ 12GB 内存。
很多人上线后才发现 OOM，就是没算清楚这笔账。
```

**解决**：
- 预算内存的时候 ×3 打余量
- 内存不够就降维度（bge-small-zh 512 维 vs bge-large-zh 1024 维）
- 或者用 IVF-PQ 压缩

### 坑 2：过滤条件太苛刻，召回骤降

```
场景: "只搜 2024 年 3 月的文档，且分类=财务"

  向量库有 1000 万条数据
  过滤后只剩 2000 条符合条件
  HNSW 在这 2000 条里搜 → 图结构稀疏，连接断了
  结果: 明明有匹配的文档，就是搜不出来
```

**解决**：
- 过滤后候选集太小（<1 万）就切到 Flat 暴力搜
- Qdrant 会自动处理（原生过滤好得多），Milvus 可以调 `search_params` 的 `nprobe`
- 或者先放宽过滤，ANN 取 top-N 之后再做后过滤（牺牲部分召回换性能）

### 坑 3：冷启动 / 增量更新问题

```
HNSW 是一种"增量构建会退化"的索引：
  批量一次性构建 → 图结构质量最高
  每天逐条插入 → 图结构越来越乱，召回下降 5%~15%

症状: 刚上线效果很好，跑了两个月后检索越来越差。
```

**解决**：
- 定期（每周/每月）后台离线重建索引，再热切换
- Milvus 有 `compact` 指令，Qdrant 有自动优化器可以定期触发
- 批量更新优于单条逐条更新（攒 1000 条再 upsert）

### 坑 4：距离度量选错了

```
Embedding 模型输出后，不同模型要求的距离函数不一样：

  模型 normalize 了（向量模长=1）
    → Inner Product (内积) = Cosine Similarity，效果一样快
    → 用 Inner Product 更快（不用算分母）

  模型没 normalize
    → 必须用 Cosine
    → 用 L2 距离会出问题（长向量天然更近）

大部分中文 embedding (bge、m3e) 推荐 normalize 后用 Inner Product。
选 L2 / Cosine 直接选错了 → 检索效果差一个档次。
```

**解决**：
- 看 embedding 模型文档，推荐啥就用啥
- bge 系列：先 `normalize_embeddings=True`，然后用 IP/Cosine
- 不确定就都 normalize + Cosine，效果不会太差

### 坑 5：pgvector 里存了千万向量还在硬扛

```
pgvector 很方便，但它不是银弹。

  500万 768维向量:
    PG 内存够 → HNSW 还能打，p95 延迟 20~50ms  ✅
    PG 内存不够（索引全走磁盘） → p95 延迟飙到 500ms+  ❌

  1000万+ 向量:
    HNSW 索引构建需要几十个 GB 内存 + 几小时
    每次 VACUUM / 索引重建痛不欲生
```

**解决**：
- 500 万以内大胆用 pgvector
- 500~1000 万观察，内存够就继续
- 超 1000 万 → 迁移到 Qdrant 或 Milvus

## 七、最终选型流程图

### 7.1 总流程图：一张图跑完选型

> **说明**：下方是纯内联 SVG（零 JS、零外部依赖，任何渲染环境都能直接显示）。Mermaid 源文件附在最后（便于复制到其他工具）。

<svg viewBox="0 0 780 980" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;display:block;">
  <defs>
    <marker id="arrow0" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto"><path d="M1 1 L7 4 L1 7 Z" fill="#9ca3af"/></marker>
    <marker id="arrowB" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto"><path d="M1 1 L7 4 L1 7 Z" fill="#4f46e5"/></marker>
    <style>
      .c-start{fill:#ecfdf5;stroke:#10b981;stroke-width:2}
      .c-decision{fill:#fff7ed;stroke:#f59e0b;stroke-width:2}
      .c-end{fill:#eef2ff;stroke:#6366f1;stroke-width:2}
      .c-rec{fill:#ffffff;stroke:#6b7280;stroke-width:1.5}
      .c-brand{fill:#eef2ff;stroke:#4f46e5;stroke-width:2}
      .t-title{font:600 14px/18px system-ui,-apple-system,sans-serif;fill:#1f2937}
      .t-body{font:12px/16px system-ui,-apple-system,sans-serif;fill:#6b7280}
      .t-label{font:12px/16px system-ui,-apple-system,sans-serif;fill:#6b7280}
      .edge{fill:none;stroke:#9ca3af;stroke-width:1.75;stroke-linecap:round;stroke-linejoin:round}
      .edge-b{fill:none;stroke:#6366f1;stroke-width:2.25;stroke-linecap:round;stroke-linejoin:round}
      .edge-pill{fill:#fff;stroke:#e5e7eb}
      .edge-pill-brand{fill:#eef2ff;stroke:none}
    </style>
  </defs>
  <!-- START -->
  <rect class="c-start" x="310" y="20" width="160" height="52" rx="26"/>
  <text class="t-title" x="390" y="42" text-anchor="middle">开始向量库选型</text>
  <text class="t-body" x="390" y="60" text-anchor="middle">先回答顶层三个问题</text>

  <path class="edge" d="M390 72 L390 100 L110 100 L110 120" marker-end="url(#arrow0)"/>
  <path class="edge" d="M390 72 L390 100 L390 120" marker-end="url(#arrow0)"/>
  <path class="edge" d="M390 72 L390 100 L670 100 L670 120" marker-end="url(#arrow0)"/>
  <rect x="36" y="96" width="60" height="18" rx="9" class="edge-pill-brand"/>
  <text class="t-label" x="66" y="109" text-anchor="middle" fill="#4f46e5">托管SaaS</text>
  <rect x="360" y="96" width="60" height="18" rx="9" class="edge-pill-brand"/>
  <text class="t-label" x="390" y="109" text-anchor="middle" fill="#4f46e5">自建</text>
  <rect x="640" y="96" width="60" height="18" rx="9" class="edge-pill-brand"/>
  <text class="t-label" x="670" y="109" text-anchor="middle" fill="#4f46e5">嵌入式</text>

  <polygon class="c-decision" points="110,120 175,160 110,200 45,160"/>
  <text class="t-title" x="110" y="157" text-anchor="middle">合规地区</text>
  <polygon class="c-decision" points="390,120 455,160 390,200 325,160"/>
  <text class="t-title" x="390" y="157" text-anchor="middle">数据量级</text>
  <polygon class="c-decision" points="670,120 735,160 670,200 605,160"/>
  <text class="t-title" x="670" y="157" text-anchor="middle">用途</text>

  <path class="edge" d="M45 160 L10 160 L10 240 L100 240" marker-end="url(#arrow0)"/>
  <path class="edge" d="M175 160 L210 160 L210 290 L260 290 L260 340" marker-end="url(#arrowB)"/>
  <rect x="14" y="220" width="170" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="99" y="233" text-anchor="middle">可出境 海外节点</text>
  <rect x="10" y="290" width="215" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="117" y="303" text-anchor="middle">极敏 中国大陆无节点 → 转自建</text>

  <rect class="c-rec" x="10" y="240" width="184" height="70" rx="8"/>
  <text class="t-title" x="102" y="267" text-anchor="middle">Pinecone / Weaviate C</text>
  <text class="t-body" x="102" y="286" text-anchor="middle">零运维 弹性扩缩</text>

  <path class="edge" d="M325 160 L325 230 L200 230 L200 320" marker-end="url(#arrow0)"/>
  <path class="edge" d="M390 200 L390 320" marker-end="url(#arrow0)"/>
  <path class="edge" d="M455 160 L455 230 L580 230 L580 320" marker-end="url(#arrow0)"/>
  <rect x="212" y="220" width="150" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="287" y="233" text-anchor="middle">100万以内</text>
  <rect x="320" y="270" width="140" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="390" y="283" text-anchor="middle">100万至1亿</text>
  <rect x="487" y="220" width="150" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="562" y="233" text-anchor="middle">1亿以上</text>

  <polygon class="c-decision" points="200,320 245,355 200,390 155,355"/>
  <text class="t-title" x="200" y="353" text-anchor="middle">已有PG</text>
  <polygon class="c-decision" points="390,320 440,360 390,400 340,360"/>
  <text class="t-title" x="390" y="357" text-anchor="middle">过滤需求重吗</text>
  <rect class="c-brand" x="490" y="302" width="180" height="76" rx="8"/>
  <text class="t-title" x="580" y="332" text-anchor="middle">Milvus 集群</text>
  <text class="t-body" x="580" y="352" text-anchor="middle">DiskANN 冷热分层</text>
  <text class="t-body" x="580" y="368" text-anchor="middle">面向亿级向量</text>

  <path class="edge-b" d="M155 355 L80 355 L80 440" marker-end="url(#arrowB)"/>
  <path class="edge" d="M245 355 L320 355 L320 420 L360 420 L360 440" marker-end="url(#arrow0)"/>
  <rect x="52" y="398" width="94" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="99" y="410" text-anchor="middle">有</text>
  <rect x="272" y="398" width="156" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="350" y="410" text-anchor="middle">无 → 要快速Demo</text>

  <rect class="c-brand" x="20" y="440" width="160" height="60" rx="8"/>
  <text class="t-title" x="100" y="464" text-anchor="middle">pgvector</text>
  <text class="t-body" x="100" y="484" text-anchor="middle">无新增组件</text>

  <polygon class="c-decision" points="360,440 400,468 360,496 320,468"/>
  <text class="t-title" x="360" y="466" text-anchor="middle">快速Demo</text>
  <path class="edge-b" d="M320 468 L250 468 L250 540" marker-end="url(#arrowB)"/>
  <path class="edge-b" d="M400 468 L470 468 L470 540" marker-end="url(#arrowB)"/>
  <rect x="218" y="504" width="64" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="250" y="516" text-anchor="middle">是</text>
  <rect x="440" y="504" width="64" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="472" y="516" text-anchor="middle">否</text>

  <rect class="c-brand" x="170" y="540" width="160" height="60" rx="8"/>
  <text class="t-title" x="250" y="564" text-anchor="middle">Chroma</text>
  <text class="t-body" x="250" y="584" text-anchor="middle">5分钟跑通</text>
  <rect class="c-brand" x="390" y="540" width="160" height="60" rx="8"/>
  <text class="t-title" x="470" y="564" text-anchor="middle">Qdrant</text>
  <text class="t-body" x="470" y="584" text-anchor="middle">单机Docker即用</text>

  <path class="edge-b" d="M340 360 L260 360 L260 460 L260 620 L620 620 L620 660" marker-end="url(#arrowB)"/>
  <path class="edge-b" d="M440 360 L520 360 L520 430 L560 430 L560 460" marker-end="url(#arrowB)"/>
  <rect x="240" y="418" width="118" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="299" y="430" text-anchor="middle">一般 → 能运维集群</text>
  <rect x="492" y="408" width="82" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="533" y="420" text-anchor="middle">重过滤</text>

  <rect class="c-brand" x="470" y="460" width="180" height="60" rx="8"/>
  <text class="t-title" x="560" y="484" text-anchor="middle">Qdrant</text>
  <text class="t-body" x="560" y="504" text-anchor="middle">原生HNSW 过滤</text>

  <polygon class="c-decision" points="560,540 605,570 560,600 515,570"/>
  <text class="t-title" x="560" y="569" text-anchor="middle">能运维集群</text>
  <path class="edge-b" d="M515 570 L420 570 L420 640 L340 640 L340 660" marker-end="url(#arrowB)"/>
  <path class="edge-b" d="M605 570 L700 570 L700 660" marker-end="url(#arrowB)"/>
  <rect x="410" y="620" width="70" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="445" y="632" text-anchor="middle">能</text>
  <rect x="670" y="620" width="60" height="16" rx="8" class="edge-pill"/>
  <text class="t-label" x="700" y="632" text-anchor="middle">只需单机</text>

  <rect class="c-brand" x="260" y="660" width="160" height="60" rx="8"/>
  <text class="t-title" x="340" y="684" text-anchor="middle">Milvus Qdrant</text>
  <text class="t-body" x="340" y="704" text-anchor="middle">集群高可用</text>
  <rect class="c-brand" x="620" y="660" width="160" height="60" rx="8"/>
  <text class="t-title" x="700" y="684" text-anchor="middle">Qdrant单机</text>
  <text class="t-body" x="700" y="704" text-anchor="middle">或 pgvector 继续扛</text>

  <path class="edge" d="M605 160 L560 160 L560 240 L560 340" marker-end="url(#arrow0)"/>
  <path class="edge" d="M735 160 L770 160 L770 240 L770 340" marker-end="url(#arrow0)"/>
  <rect x="472" y="190" width="180" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="562" y="203" text-anchor="middle">Demo 原型 小项目</text>
  <rect x="700" y="190" width="150" height="18" rx="9" class="edge-pill"/>
  <text class="t-label" x="775" y="203" text-anchor="middle">嵌入业务 性能</text>

  <rect class="c-brand" x="470" y="340" width="180" height="60" rx="8"/>
  <text class="t-title" x="560" y="364" text-anchor="middle">Chroma</text>
  <text class="t-body" x="560" y="384" text-anchor="middle">Python内嵌零依赖</text>

  <polygon class="c-decision" points="770,340 815,370 770,400 725,370"/>
  <text class="t-title" x="770" y="368" text-anchor="middle">需要GPU</text>
  <path class="edge-b" d="M725 370 L670 370 L670 470" marker-end="url(#arrowB)"/>
  <path class="edge-b" d="M815 370 L870 370 L870 470" marker-end="url(#arrowB)"/>

  <rect class="c-brand" x="580" y="470" width="180" height="60" rx="8"/>
  <text class="t-title" x="670" y="494" text-anchor="middle">FAISS GPU</text>
  <text class="t-body" x="670" y="514" text-anchor="middle">性能天花板</text>
  <rect class="c-brand" x="780" y="470" width="180" height="60" rx="8"/>
  <text class="t-title" x="870" y="494" text-anchor="middle">FAISS</text>
  <text class="t-body" x="870" y="514" text-anchor="middle">自研服务层封装</text>

  <rect class="c-end" x="330" y="900" width="180" height="56" rx="28"/>
  <text class="t-title" x="420" y="922" text-anchor="middle">完成选型</text>
  <text class="t-body" x="420" y="940" text-anchor="middle">3 张流程图 9 条场景推荐</text>
</svg>

<details style="margin-top:8px">
<summary style="cursor:pointer;color:#6b7280;font-size:12px">📋 查看 Mermaid 源码（可复制到其他 Mermaid 工具渲染）</summary>

```
flowchart TD
A([开始]) --> B{部署形态}
B -->|托管SaaS| C1{合规和地区}
B -->|私有化自建| D1[进入自建流程]
B -->|嵌入式本地| E1{用途}
C1 -->|可出境+海外节点| C2[Pinecone或WeaviateCloud]
C1 -->|极敏感或中国大陆| C3[转自建分支]
D1 --> D2{数据量级}
D2 -->|100万以内| D3{已有PostgreSQL}
D2 -->|100万到1亿| D4{过滤需求重吗}
D2 -->|1亿以上| D5[Milvus集群\nDiskANN冷热分层]
D3 -->|有| D3a[pgvector\n无新增组件]
D3 -->|无| D3b{要快速Demo}
D3b -->|是| D3b1[Chroma\n5分钟跑通]
D3b -->|否| D3b2[Qdrant\n单机Docker即用]
D4 -->|重过滤 多条件组合| D4a[Qdrant\n原生HNSW加过滤]
D4 -->|一般| D4b{能运维K8s集群}
D4b -->|能| D4b1[Milvus或Qdrant集群]
D4b -->|只需单机Docker| D4b2[Qdrant单机\n或pgvector继续扛]
E1 -->|Demo原型小项目| E2[Chroma\nPython内嵌零依赖]
E1 -->|嵌入业务系统性能优先| E3{需要GPU加速}
E3 -->|需要 图像多模态| E3a[FAISSGPU\n性能天花板]
E3 -->|不需要| E3b[FAISS\n自研服务层封装]
C2 --> Z([完成选型])
C3 --> D1
D3a --> Z
D3b1 --> Z
D3b2 --> Z
D4a --> Z
D4b1 --> Z
D4b2 --> Z
D5 --> Z
E2 --> Z
E3a --> Z
E3b --> Z
```

</details>

### 7.2 私有化自建细化流程（80% 场景走这里）

<svg viewBox="0 0 780 720" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;display:block;">
  <defs>
    <marker id="z1" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto"><path d="M1 1 L7 4 L1 7 Z" fill="#9ca3af"/></marker>
    <marker id="z2" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto"><path d="M1 1 L7 4 L1 7 Z" fill="#4f46e5"/></marker>
    <style>
      .s{font:600 14px/18px system-ui,-apple-system,sans-serif;fill:#1f2937}
      .b{font:12px/16px system-ui,-apple-system,sans-serif;fill:#6b7280}
      .l{font:12px/16px system-ui,-apple-system,sans-serif;fill:#6b7280}
      .e0{fill:none;stroke:#9ca3af;stroke-width:1.75;stroke-linecap:round;stroke-linejoin:round}
      .e1{fill:none;stroke:#6366f1;stroke-width:2.25;stroke-linecap:round;stroke-linejoin:round}
      .cs{fill:#ecfdf5;stroke:#10b981;stroke-width:2}
      .cd{fill:#fff7ed;stroke:#f59e0b;stroke-width:2}
      .cb{fill:#eef2ff;stroke:#4f46e5;stroke-width:2}
      .cr{fill:#fff;stroke:#6b7280;stroke-width:1.5}
      .ce{fill:#fff;stroke:#e5e7eb}
      .cw{fill:#fffbeb;stroke:#fde68a;stroke:none}
    </style>
  </defs>
  <rect class="cs" x="290" y="16" width="200" height="50" rx="25"/>
  <text class="s" x="390" y="38" text-anchor="middle">私有化自建流程</text>
  <text class="b" x="390" y="56" text-anchor="middle">80% 企业场景走这条路径</text>
  <path class="e0" d="M390 66 L390 92" marker-end="url(#z1)"/>
  <polygon class="cd" points="390,92 455,140 390,188 325,140"/>
  <text class="s" x="390" y="136" text-anchor="middle">向量规模</text>

  <path class="e0" d="M325 140 L220 140 L220 188 L110 188" marker-end="url(#z1)"/>
  <path class="e0" d="M390 188 L390 250" marker-end="url(#z1)"/>
  <path class="e0" d="M455 140 L560 140 L560 188 L670 188" marker-end="url(#z1)"/>
  <path class="e0" d="M455 140 L710 140 L710 360 L710 400" marker-end="url(#z1)"/>
  <rect x="120" y="168" width="160" height="18" rx="9" class="cw"/>
  <text class="l" x="200" y="181" text-anchor="middle">500万以内</text>
  <rect x="320" y="218" width="140" height="18" rx="9" class="cw"/>
  <text class="l" x="390" y="231" text-anchor="middle">500万到5000万</text>
  <rect x="565" y="168" width="180" height="18" rx="9" class="cw"/>
  <text class="l" x="655" y="181" text-anchor="middle">5000万到1亿</text>
  <rect x="690" y="280" width="130" height="18" rx="9" class="cw"/>
  <text class="l" x="755" y="293" text-anchor="middle">1亿以上</text>

  <polygon class="cd" points="110,188 155,220 110,252 65,220"/>
  <text class="s" x="110" y="219" text-anchor="middle">已有PG</text>
  <path class="e1" d="M65 220 L20 220 L20 330" marker-end="url(#z2)"/>
  <path class="e0" d="M155 220 L210 220 L210 300 L260 300 L260 330" marker-end="url(#z1)"/>
  <rect x="20" y="275" width="40" height="16" rx="8" class="ce"/>
  <text class="l" x="40" y="287" text-anchor="middle">有</text>
  <rect x="212" y="280" width="94" height="16" rx="8" class="ce"/>
  <text class="l" x="259" y="292" text-anchor="middle">无 预算团队</text>

  <rect class="cb" x="-10" y="330" width="160" height="66" rx="8"/>
  <text class="s" x="70" y="354" text-anchor="middle">pgvector</text>
  <text class="b" x="70" y="372" text-anchor="middle">表Join备份监控</text>
  <text class="b" x="70" y="388" text-anchor="middle">PG能力全复用</text>

  <polygon class="cd" points="260,330 305,360 260,390 215,360"/>
  <text class="s" x="260" y="358" text-anchor="middle">预算团队</text>
  <path class="e1" d="M215 360 L150 360 L150 450" marker-end="url(#z2)"/>
  <path class="e1" d="M305 360 L360 360 L360 450" marker-end="url(#z2)"/>
  <rect class="cb" x="70" y="450" width="160" height="58" rx="8"/>
  <text class="s" x="150" y="472" text-anchor="middle">Chroma</text>
  <text class="b" x="150" y="492" text-anchor="middle">极简 原型验证</text>
  <rect class="cb" x="280" y="450" width="160" height="58" rx="8"/>
  <text class="s" x="360" y="472" text-anchor="middle">Qdrant 单机</text>
  <text class="b" x="360" y="492" text-anchor="middle">Rust 单 binary Docker部署</text>

  <polygon class="cd" points="390,250 440,286 390,322 340,286"/>
  <text class="s" x="390" y="284" text-anchor="middle">过滤权重</text>
  <path class="e1" d="M340 286 L280 286 L280 380 L430 380 L430 420" marker-end="url(#z2)"/>
  <path class="e0" d="M440 286 L500 286 L500 340 L540 340 L540 370" marker-end="url(#z1)"/>
  <rect class="cb" x="350" y="420" width="160" height="66" rx="8"/>
  <text class="s" x="430" y="444" text-anchor="middle">Qdrant</text>
  <text class="b" x="430" y="462" text-anchor="middle">HNSW 内置 Payload 索引</text>
  <text class="b" x="430" y="478" text-anchor="middle">过滤召回性能双优</text>

  <polygon class="cd" points="540,370 580,396 540,422 500,396"/>
  <text class="s" x="540" y="394" text-anchor="middle">技术栈</text>
  <path class="e1" d="M500 396 L440 396 L440 520" marker-end="url(#z2)"/>
  <path class="e1" d="M580 396 L640 396 L640 520" marker-end="url(#z2)"/>
  <path class="e1" d="M540 422 L540 480 L710 480 L710 520" marker-end="url(#z2)"/>
  <rect class="cb" x="360" y="520" width="160" height="58" rx="8"/>
  <text class="s" x="440" y="542" text-anchor="middle">Milvus 单机</text>
  <text class="b" x="440" y="562" text-anchor="middle">功能最全 Go加K8s</text>
  <rect class="cb" x="560" y="520" width="160" height="58" rx="8"/>
  <text class="s" x="640" y="542" text-anchor="middle">Qdrant 单机</text>
  <text class="b" x="640" y="562" text-anchor="middle">资源占用低 Rust</text>
  <rect class="cb" x="620" y="598" width="160" height="58" rx="8"/>
  <text class="s" x="700" y="620" text-anchor="middle">pgvector 观察</text>
  <text class="b" x="700" y="640" text-anchor="middle">PG老用户 内存延迟监控</text>

  <polygon class="cd" points="670,188 715,220 670,252 625,220"/>
  <text class="s" x="670" y="219" text-anchor="middle">能运维集群</text>
  <path class="e1" d="M625 220 L560 220 L560 400 L640 400 L640 660 L360 660" marker-end="url(#z2)"/>
  <path class="e0" d="M715 220 L780 220 L780 300" marker-end="url(#z1)"/>
  <rect class="cb" x="280" y="660" width="160" height="58" rx="8"/>
  <text class="s" x="360" y="682" text-anchor="middle">Milvus Operator</text>
  <text class="b" x="360" y="702" text-anchor="middle">K8s 分片 副本 S3</text>

  <polygon class="cd" points="780,300 820,326 780,352 740,326"/>
  <text class="s" x="780" y="324" text-anchor="middle">多台机器</text>
  <path class="e1" d="M740 326 L670 326 L670 400 L560 400 L560 660 L470 660" marker-end="url(#z2)"/>
  <path class="e1" d="M820 326 L870 326 L870 400 L870 660 L550 660" marker-end="url(#z2)"/>
  <rect class="cb" x="470" y="660" width="160" height="58" rx="8"/>
  <text class="s" x="550" y="682" text-anchor="middle">Qdrant 集群</text>
  <text class="b" x="550" y="702" text-anchor="middle">Raft 分片 副本</text>
  <rect class="cb" x="650" y="660" width="160" height="58" rx="8"/>
  <text class="s" x="730" y="682" text-anchor="middle">Qdrant 大内存</text>
  <text class="b" x="730" y="702" text-anchor="middle">或 Milvus DiskANN</text>

  <rect class="cb" x="650" y="400" width="180" height="66" rx="8"/>
  <text class="s" x="740" y="424" text-anchor="middle">Milvus Cluster</text>
  <text class="b" x="740" y="442" text-anchor="middle">DiskANN 冷热分层</text>
  <text class="b" x="740" y="458" text-anchor="middle">亿级向量降本首选</text>
</svg>

> **📌 特殊需求补充**（任一满足即额外调优）：
> - 需要 **GPU 加速**（图像/多模态）→ FAISS(GPU) 或 Milvus GPU 索引
> - 需要 **BM25 + 向量混合检索** → Qdrant（稀疏向量）或 Milvus（BM25）
> - 多租户 SaaS 隔离 → Qdrant（多集合）或 Pinecone（命名空间）

### 7.3 托管 SaaS 细化流程

<svg viewBox="0 0 780 560" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;display:block;">
  <defs>
    <marker id="t1" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto"><path d="M1 1 L7 4 L1 7 Z" fill="#9ca3af"/></marker>
    <marker id="t2" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto"><path d="M1 1 L7 4 L1 7 Z" fill="#4f46e5"/></marker>
    <style>
      .ts{font:600 14px/18px system-ui,-apple-system,sans-serif;fill:#1f2937}
      .tb{font:12px/16px system-ui,-apple-system,sans-serif;fill:#6b7280}
      .tl{font:12px/16px system-ui,-apple-system,sans-serif;fill:#6b7280}
      .te0{fill:none;stroke:#9ca3af;stroke-width:1.75;stroke-linecap:round;stroke-linejoin:round}
      .te1{fill:none;stroke:#6366f1;stroke-width:2.25;stroke-linecap:round;stroke-linejoin:round}
      .tcs{fill:#ecfdf5;stroke:#10b981;stroke-width:2}
      .tcd{fill:#fff7ed;stroke:#f59e0b;stroke-width:2}
      .tcb{fill:#eef2ff;stroke:#4f46e5;stroke-width:2}
      .tcr{fill:#fff;stroke:#6b7280;stroke-width:1.5}
      .tce{fill:#fff;stroke:#e5e7eb}
      .tcw{fill:#dcfce7;stroke:none}
      .tce2{fill:#fee2e2;stroke:none}
      .tcx{fill:#fef2f2;stroke:#dc2626;stroke-width:2}
    </style>
  </defs>
  <rect class="tcs" x="310" y="16" width="160" height="50" rx="25"/>
  <text class="ts" x="390" y="38" text-anchor="middle">托管 SaaS 流程</text>
  <text class="tb" x="390" y="56" text-anchor="middle">零运维但要考虑合规、延迟、费用</text>
  <path class="te0" d="M390 66 L390 92" marker-end="url(#t1)"/>

  <polygon class="tcd" points="390,92 455,140 390,188 325,140"/>
  <text class="ts" x="390" y="136" text-anchor="middle">数据可以出境</text>
  <path class="te0" d="M325 140 L170 140 L170 230" marker-end="url(#t1)"/>
  <path class="te0" d="M455 140 L570 140 L570 188" marker-end="url(#t1)"/>
  <rect x="120" y="190" width="200" height="18" rx="9" class="tce2"/>
  <text class="tl" x="220" y="203" text-anchor="middle" fill="#b91c1c">不可以 合规 极敏</text>
  <rect x="530" y="168" width="110" height="18" rx="9" class="tcw"/>
  <text class="tl" x="585" y="181" text-anchor="middle" fill="#166534">可以</text>

  <rect class="tcx" x="80" y="230" width="180" height="66" rx="8"/>
  <text class="ts" x="170" y="254" text-anchor="middle" fill="#991b1b">❌ 放弃托管</text>
  <text class="tb" x="170" y="272" text-anchor="middle">转私有化自建分支</text>
  <text class="tb" x="170" y="288" text-anchor="middle">Qdrant Milvus 优先</text>

  <polygon class="tcd" points="570,188 620,222 570,256 520,222"/>
  <text class="ts" x="570" y="220" text-anchor="middle">所在地区</text>
  <path class="te0" d="M520 222 L430 222 L430 310" marker-end="url(#t1)"/>
  <path class="te0" d="M620 222 L700 222 L700 280" marker-end="url(#t1)"/>
  <rect x="402" y="268" width="120" height="18" rx="9" class="tce2"/>
  <text class="tl" x="462" y="281" text-anchor="middle" fill="#b91c1c">中国大陆</text>
  <rect x="652" y="248" width="180" height="18" rx="9" class="tcw"/>
  <text class="tl" x="742" y="261" text-anchor="middle" fill="#166534">海外云 AWS GCP Azure</text>

  <rect class="tcx" x="340" y="310" width="180" height="66" rx="8"/>
  <text class="ts" x="430" y="334" text-anchor="middle" fill="#991b1b">❌ Pinecone 无节点</text>
  <text class="tb" x="430" y="352" text-anchor="middle">转 Qdrant / Milvus 自建</text>
  <text class="tb" x="430" y="368" text-anchor="middle">网络延迟和合规双硬伤</text>

  <polygon class="tcd" points="700,280 750,314 700,348 650,314"/>
  <text class="ts" x="700" y="312" text-anchor="middle">预算充足</text>
  <path class="te0" d="M650 314 L540 314 L540 400" marker-end="url(#t1)"/>
  <path class="te0" d="M750 314 L820 314 L820 400" marker-end="url(#t1)"/>
  <rect x="470" y="358" width="160" height="18" rx="9" class="tce"/>
  <text class="tl" x="550" y="371" text-anchor="middle">月预算 200美元 以内</text>
  <rect x="778" y="358" width="150" height="18" rx="9" class="tce"/>
  <text class="tl" x="853" y="371" text-anchor="middle">月预算 200美元 以上</text>

  <rect class="tcb" x="450" y="400" width="180" height="66" rx="8"/>
  <text class="ts" x="540" y="424" text-anchor="middle">轻量方案</text>
  <text class="tb" x="540" y="442" text-anchor="middle">Weaviate Free Tier 起步</text>
  <text class="tb" x="540" y="458" text-anchor="middle">或直接自建 Qdrant</text>

  <polygon class="tcd" points="820,400 870,432 820,464 770,432"/>
  <text class="ts" x="820" y="430" text-anchor="middle">核心需求</text>
  <path class="te1" d="M770 432 L680 432 L680 500" marker-end="url(#t2)"/>
  <path class="te1" d="M820 464 L820 500" marker-end="url(#t2)"/>
  <path class="te1" d="M870 432 L950 432 L950 500" marker-end="url(#t2)"/>

  <rect class="tcb" x="590" y="500" width="180" height="58" rx="8"/>
  <text class="ts" x="680" y="522" text-anchor="middle">Pinecone</text>
  <text class="tb" x="680" y="542" text-anchor="middle">纯向量 弹性 S3 稳定</text>
  <rect class="tcb" x="730" y="500" width="180" height="58" rx="8"/>
  <text class="ts" x="820" y="522" text-anchor="middle">Weaviate Cloud</text>
  <text class="tb" x="820" y="542" text-anchor="middle">内置向量化 加 LLM 模块</text>
  <rect class="tcb" x="860" y="500" width="180" height="58" rx="8"/>
  <text class="ts" x="950" y="522" text-anchor="middle">Atlas Vector</text>
  <text class="tb" x="950" y="542" text-anchor="middle">MongoDB 用户省一份运维</text>
</svg>

> **⚠️ 托管选型注意**：
> - **隐私条款**（GDPR/HIPAA/等保）：务必与供应商确认数据存储地域
> - **实测延迟**：中国大陆访问海外节点，300ms 以上很常见
> - **峰值费用**：按 Pod 计费要压测峰值，避免月底账单爆炸
> - **出口锁定**：Pinecone 无数据导出 API，迁移成本极高

### 7.4 速查手册：30 秒定结论

如果不想走流程图，直接查表：

| 你的场景 | 首选 | 备选 | 别选 |
| --- | --- | --- | --- |
| **PG 用户 + <500 万向量** | pgvector ✅ | Chroma | Milvus（太重） |
| **快速 Demo / 原型验证** | Chroma ✅ | FAISS | Pinecone（贵+慢启动） |
| **100万~5000万 + 过滤多** | Qdrant ✅ | Milvus | pgvector（过滤性能弱） |
| **>1 亿向量 + 冷热分层** | Milvus 集群 ✅ | Qdrant 集群 | pgvector / Chroma |
| **海外业务 + 零运维** | Pinecone ✅ | Weaviate Cloud | 任何自建（人力成本） |
| **图像/多模态 + GPU 加速** | FAISS(GPU) ✅ | Milvus(GPU) | pgvector / Qdrant |
| **政企私有化 + <千万** | Qdrant ✅ | pgvector | Pinecone / Weaviate Cloud |
| **内置向量化 + 不想组装** | Weaviate ✅ | — | FAISS（要全自研） |
| **已有 ES 集群 + 小体量** | ES k-NN ✅ | pgvector | Milvus（重复运维） |

## 八、总结：选型的本质是权衡

向量库选型没有"最好"，只有"最适合"。核心是三个维度的权衡：

```
           性能
            ▲
            │  Milvus(分布式)
            │  FAISS(纯库)
            │
            │        Qdrant
            │
            │  pgvector     Chroma
            │
            └─────────────────────► 运维复杂度
           /
          /
         ▼
      功能丰富度
```

**三条实操建议：**

1. **先简单后复杂**：先用 pgvector/Chroma 跑起来，数据量上来了再迁移。向量库迁移不难（导出 embedding → 导入新库，最多半天工作量）。
2. **过滤多 → 选 Qdrant**：元数据过滤是 RAG 的高频需求，Qdrant 在这块体验最好。
3. **数据超亿 → 选 Milvus**：只有它的冷热分层 + DiskANN 能把成本压下来，分布式也最成熟。

向量库是 RAG 的基础设施，但不是 RAG 质量的瓶颈。**RAG 效果的上限，70% 由切分策略、embedding 模型、重排序决定，30% 由向量库决定。** 别在选型上花太多时间纠结，跑起来再迭代才是正道。
