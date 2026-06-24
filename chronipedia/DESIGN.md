# Chronipedia — 个人数字分身系统设计文档

> "时间的存在是由记忆去构建的。"

## 1. 项目概述

### 1.1 愿景

Chronipedia 是一个渐进式个人数字分身系统。它从一个"人生记录器"出发，逐步演化为拥有你所有记忆、思维方式和做事方式的数字自我。你可以问它你已经遗忘的事情，它会"记得"。

### 1.2 核心理念

- **记忆是地基**：先有结构化的记忆，才能有可信的分身
- **渐进式演化**：每个阶段都有独立可用的产出，不追求一步到位
- **数据主权**：个人数据本地优先，可选云端同步
- **零成本优先**：所有组件必须可零费用自托管，不依赖付费服务
- **效果优先**：技术选型以最终效果为导向，接受一定复杂度

### 1.3 与 qzpedia 原始想法的关系

qzpedia 的原始想法是"记录自己不想遗忘的事情，编写属于自己的人生百科"。Chronipedia 是这个想法的延伸：从"被动记录"到"主动回忆"，从"知识库"到"数字分身"。

## 2. 系统架构

### 2.1 四阶段架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                     Chronipedia 整体架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Phase 4: 代理交互层 (Agent Interaction)                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  语音克隆    │ │  多平台部署  │ │  对话代理    │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                         ▲                                       │
│  Phase 3: 思维模拟层 (Cognitive Simulation)                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  人格微调    │ │  决策模拟    │ │  风格迁移    │               │
│  │ (CharLoRA)  │ │             │ │             │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                         ▲                                       │
│  Phase 2: 记忆检索层 (Memory Retrieval)                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  RAG 引擎    │ │  时序知识图谱│ │  问答系统    │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                         ▲                                       │
│  Phase 1: 记忆引擎层 (Memory Engine)                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  数据采集    │ │  记忆存储    │ │  记忆索引    │               │
│  │ (Ingestion) │ │ (Storage)   │ │ (Indexing)  │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 技术栈选型

| 层级 | 组件 | 选型 | 理由 |
|------|------|------|------|
| 数据采集 | 屏幕/音频捕获 | Screenpipe (开源) | 本地优先，跨平台，MIT 协议 |
| 数据采集 | 聊天记录 | PyWxDump + 自定义导出器 | WeClone 生态验证过 |
| 数据采集 | 手动记录 | Markdown + CLI 工具 | GBrain 模式，可版本控制 |
| 记忆存储 | 向量数据库 | sqlite-vec (MVP) → Postgres+pgvector (扩展) | 零依赖起步，渐进升级，全程免费 |
| 记忆存储 | 时序知识图谱 | 自研时序图谱引擎 (借鉴 Zep/Graphiti 设计) | 时序推理是核心需求，自研零依赖更可控 |
| 记忆存储 | 认知分离层 | 四网络逻辑分离 (借鉴 Hindsight) | 事实-观点分离是分身关键能力 |
| 记忆存储 | 原始文件 | Markdown + Git (借鉴 GBrain) | 数据主权，可版本控制 |
| 记忆索引 | 嵌入模型 | bge-m3 (本地，通过 Ollama) | 中英文效果好，零费用 |
| 记忆索引 | 全文索引 | SQLite FTS5 | 零依赖，内置全文检索 |
| RAG 引擎 | 检索框架 | LlamaIndex | 个人 RAG 生态最成熟 |
| LLM 推理 | 基础模型 | Qwen2.5-14B-Instruct (Ollama 本地) | 中文能力强，零费用 |
| 人格微调 | 微调框架 | LLaMA Factory + LoRA/CharLoRA | WeClone 验证过的流程，开源免费 |
| 语音克隆 | TTS | Spark-TTS 或 GPT-SoVITS | 中文语音克隆效果好，开源免费 |
| 部署 | 后端 | FastAPI (Python) | 与 AI 生态契合 |
| 部署 | 前端 | Gradio / Streamlit (MVP) → React (后续) | 快速迭代 |

## 3. Phase 1: 记忆引擎层

### 3.1 目标

建立个人数据的采集、存储、索引基础设施。产出：一个可搜索的个人记忆库。

### 3.2 数据模型

```python
# 核心数据模型 (伪代码，用于说明结构)

class Memory:
    """一条记忆的统一表示"""
    id: str                    # UUID
    source: str                # 数据来源: screenpipe|wechat|diary|photo|email
    content: str               # 文本内容
    content_type: str          # text|image|audio|video
    raw_path: str              # 原始文件路径 (如有)
    timestamp: datetime        # 这件事发生的时间
    ingested_at: datetime      # 被录入系统的时间
    metadata: dict             # 额外元数据 (地点、人物、标签等)
    embedding: list[float]     # 向量嵌入
    summary: str               # LLM 生成的摘要


class MemoryEvent:
    """时序知识图谱中的事件节点"""
    id: str
    memory_id: str             # 关联的 Memory
    event_time: datetime       # 事件发生时间
    entities: list[str]        # 涉及的实体 (人、地点、事物)
    relations: list[Relation]  # 与其他事件的关系
    valid_from: datetime       # 事实有效期开始
    valid_to: datetime | None  # 事实有效期结束 (None = 仍有效)
```

### 3.3 数据采集管道

```
数据源                  预处理                    存储
┌──────────┐         ┌──────────┐           ┌──────────┐
│Screenpipe│──┐      │          │           │          │
│(屏幕/音频)│  │      │  清洗    │           │ 原始存储  │
└──────────┘  │      │  去重    │           │ (SQLite  │
              │      │  脱敏    │           │  +文件)  │
┌──────────┐  ├─────▶│  分块    │──────────▶│          │
│微信聊天   │  │      │  摘要    │           └──────────┘
│(PyWxDump)│  │      │  嵌入    │                 │
└──────────┘  │      │  实体抽取│                 │
              │      └──────────┘                 ▼
┌──────────┐  │             │              ┌──────────────┐
│手动日记   │──┘             │              │ 向量索引      │
│(Markdown)│                │              │ (sqlite-vec) │
└──────────┘                │              └──────────────┘
                            │                    │
┌──────────┐                │                    │
│照片/视频  │────────────────┘                    │
│(EXIF+VLM)│                                     │
└──────────┘                                     ▼
                                           ┌──────────────┐
                                           │ 时序图谱      │
                                           │ (自研引擎)    │
                                           └──────────────┘
```

#### 3.3.1 各数据源采集策略

**屏幕/音频 (Screenpipe 集成)**
- Screenpipe 持续捕获屏幕文本和音频，本地存储
- Chronipedia 定时从 Screenpipe 的本地数据库拉取数据
- 对屏幕内容做去重（相同内容不重复存储）
- 对音频转写文本做摘要和实体抽取

**聊天记录 (微信/QQ/Telegram)**
- 使用 PyWxDump 导出微信聊天记录为 CSV/JSON
- 解析为统一的 Memory 格式，保留时间戳和对话上下文
- 隐私过滤：正则去除手机号、身份证号、邮箱等

**手动日记 (Markdown)**
- 支持 Markdown 格式的日记/笔记
- 前置元数据（YAML front matter）记录日期、地点、人物、标签
- 类似 GBrain 的 Markdown-first 模式，可 Git 版本控制

**照片/视频**
- 提取 EXIF 信息（时间、GPS）
- 使用 VLM (视觉语言模型，如 Qwen-VL) 生成图片描述
- 将描述文本作为 Memory 内容，原始文件路径存 metadata

### 3.4 记忆索引与检索

#### 3.4.1 三级索引

1. **向量索引 (sqlite-vec)**：语义相似度检索，"找到和这件事相关的内容"
2. **时序图谱 (自研引擎)**：时间推理检索，"2019年夏天我去了哪里"
3. **全文索引 (SQLite FTS5)**：关键词精确匹配，"提到'张三'的所有记忆"

#### 3.4.2 检索融合策略

```python
def retrieve_memories(query: str, top_k: int = 10) -> list[Memory]:
    # 并行检索三个索引
    vector_results = sqlite_vec.search(query_embedding(query), limit=top_k * 2)
    temporal_results = temporal_engine.search(query, time_filter=extract_time(query))
    keyword_results = sqlite_fts.search(query, limit=top_k * 2)

    # 融合排序 (Reciprocal Rank Fusion)
    fused = reciprocal_rank_fusion(
        [vector_results, temporal_results, keyword_results]
    )

    # 去重 + 返回 top_k
    return deduplicate(fused)[:top_k]
```

### 3.5 Phase 1 验收标准

- [ ] 能从至少 2 个数据源采集数据（Screenpipe + 手动日记）
- [ ] 数据存入统一 Memory 模型，含时间戳和元数据
- [ ] 向量索引 + 全文索引可用，支持语义和关键词检索
- [ ] 检索延迟 < 500ms (10万条记忆以内)
- [ ] CLI 工具可执行：`chronipedia ingest <source>`、`chronipedia search <query>`

## 4. Phase 2: 记忆检索层

### 4.1 目标

在 Phase 1 的记忆库之上，构建能回答"我某时某地做了什么"的问答系统。产出：一个能帮你回忆的 AI。

### 4.2 RAG 架构

```
用户提问
    │
    ▼
┌──────────────┐
│  查询理解    │  ← 时间提取、实体识别、意图分类
└──────────────┘
    │
    ▼
┌──────────────┐
│  多路检索    │  ← 向量检索 + 时序检索 + 关键词检索
└──────────────┘
    │
    ▼
┌──────────────┐
│  重排序      │  ← Cross-encoder 重排，时间衰减加权
└──────────────┘
    │
    ▼
┌──────────────┐
│  上下文组装  │  ← 按时间线组织记忆，加入元数据
└──────────────┘
    │
    ▼
┌──────────────┐
│  LLM 生成    │  ← 基于检索到的记忆回答问题
└──────────────┘
    │
    ▼
┌──────────────┐
│  引用标注    │  ← 每个回答附带记忆来源
└──────────────┘
```

### 4.3 时序推理能力

这是 Chronipedia 区别于普通 RAG 的核心能力。

**场景示例**：
- "我2019年夏天去了哪里？" → 时序图谱检索 2019年6-8月的事件
- "我上次见张三是什么时候？" → 图谱查询人物关系 + 时间排序
- "我去年这个时候在做什么？" → 相对时间推理

**实现方式**：
- 自研时序知识图谱引擎（借鉴 Zep/Graphiti 设计），基于 SQLite + sqlite-vec 实现
- 每条事实携带有效期 (valid_from/valid_to)，自动维护事实时效性
- 查询时，LLM 先解析时间意图，转化为时间过滤器
- 检索结果按时间线组织，而非简单的相关度排序

### 4.4 引用与可信度

每条回答必须附带来源引用：
```
Q: 我2019年夏天去了哪里？
A: 根据你的记录，2019年7月你去了日本京都旅行 [1]，8月去了上海出差 [2]。

[1] 来源：手动日记，2019-07-15，"今天到了京都，入住..."
[2] 来源：微信聊天，2019-08-03，与李四的对话
```

### 4.5 Phase 2 验收标准

- [ ] 能回答"何时/何地/谁"类时序问题
- [ ] 回答附带来源引用，可追溯
- [ ] 支持 Web UI (Gradio) 进行问答交互
- [ ] 在 1000 条测试记忆上，时序问题准确率 > 80%

## 5. Phase 3: 思维模拟层

### 5.1 目标

让分身不仅能"回忆"，还能以你的思维方式给出建议。产出：一个"像你一样思考"的 AI。

### 5.2 人格捕获策略

采用 **RAG + 微调混合方案**：

```
┌─────────────────────────────────────────────┐
│              人格捕获双引擎                   │
├──────────────────────┬──────────────────────┤
│   知识层 (RAG)        │   风格层 (微调)       │
│                      │                      │
│  "你知道什么"         │  "你怎么说话/思考"    │
│                      │                      │
│  Phase 1-2 的记忆库   │  LoRA/CharLoRA 微调  │
│  检索你的经历和知识   │  学习你的表达风格     │
│                      │  和思维模式           │
└──────────────────────┴──────────────────────┘
```

### 5.3 微调数据准备

#### 5.3.1 数据来源

| 数据类型 | 用途 | 处理方式 |
|----------|------|----------|
| 聊天记录 | 学习日常表达风格 | 转为指令-回复对 |
| 日记/文章 | 学习书面风格和思考深度 | 转为续写/摘要任务 |
| 邮件 | 学习正式沟通风格 | 转为回复生成任务 |
| 决策记录 | 学习决策模式 | 转为"遇到X情况，我会怎么做"问答 |

#### 5.3.2 数据格式 (SFT)

```json
{
  "conversations": [
    {"role": "system", "content": "你是用户的数字分身，以用户的思维方式回答问题。"},
    {"role": "user", "content": "朋友约我周末去爬山，但我有点累，我一般会怎么回复？"},
    {"role": "assistant", "content": "哈哈说实话我有点心动但身体不太给力，要不咱们改个轻松点的？找个咖啡馆坐坐也行啊，好久没聊了"}
  ]
}
```

### 5.4 CharLoRA 微调方案

参考 CharacterBot 论文的 CharLoRA 方法：

- **共享风格专家**：从所有个人文本中学习通用的语言风格
- **任务专家**：针对不同任务（问答、建议、续写）的专用适配器
- **混合推理**：推理时组合共享专家 + 任务专家

```
基础模型 (Qwen2.5-14B)
    │
    ├── CharLoRA-shared (共享风格专家)
    │       └── 从日记、文章、聊天中学习通用风格
    │
    ├── CharLoRA-qa (问答任务专家)
    │       └── 从"我会怎么回答"数据中学习
    │
    └── CharLoRA-advice (建议任务专家)
            └── 从"我会怎么做"数据中学习
```

### 5.5 Phase 3 验收标准

- [ ] 微调后的模型在风格一致性测试中得分 > 基础模型
- [ ] 能以用户的典型表达方式回答问题
- [ ] 结合 RAG 记忆，能给出"符合用户性格"的建议
- [ ] 提供 A/B 对比工具：基础模型 vs 微调模型的回答对比

## 6. Phase 4: 代理交互层

### 6.1 目标

让分身能代替用户与他人交流。产出：一个可以对外服务的数字分身。

### 6.2 能力组成

- **语音克隆**：用 Spark-TTS/GPT-SoVITS 克隆用户声音
- **多平台部署**：微信、Telegram、Web API
- **对话代理**：以用户身份进行对话，保持风格一致
- **权限控制**：分身可回答的范围限制（不涉及敏感隐私）

### 6.3 Phase 4 验收标准

- [ ] 语音克隆相似度 > 85%
- [ ] 至少部署到 1 个聊天平台
- [ ] 对话风格一致性人工评测通过

## 7. 项目结构

```
chronipedia/
├── README.md
├── DESIGN.md                 # 本文档
├── pyproject.toml            # 项目配置 (uv)
├── settings.jsonc            # 用户配置
│
├── chronipedia/              # 主包
│   ├── __init__.py
│   ├── cli.py                # CLI 入口
│   ├── config.py             # 配置管理
│   │
│   ├── ingestion/            # 数据采集 (Phase 1)
│   │   ├── __init__.py
│   │   ├── base.py           # 采集器基类
│   │   ├── screenpipe.py     # Screenpipe 集成
│   │   ├── wechat.py         # 微信聊天记录
│   │   ├── diary.py          # Markdown 日记
│   │   └── media.py          # 照片/视频 (VLM)
│   │
│   ├── storage/              # 记忆存储 (Phase 1)
│   │   ├── __init__.py
│   │   ├── models.py         # 数据模型 (Memory, MemoryEvent)
│   │   ├── sqlite_store.py   # SQLite 原始存储 + FTS5
│   │   ├── sqlite_vec_store.py  # sqlite-vec 向量存储
│   │   └── temporal_store.py # 自研时序图谱存储
│   │
│   ├── retrieval/            # 记忆检索 (Phase 2)
│   │   ├── __init__.py
│   │   ├── vector.py         # 向量检索
│   │   ├── temporal.py       # 时序检索
│   │   ├── keyword.py        # 关键词检索
│   │   └── fusion.py         # 融合排序
│   │
│   ├── qa/                   # 问答系统 (Phase 2)
│   │   ├── __init__.py
│   │   ├── rag.py            # RAG 管道
│   │   ├── query_understanding.py  # 查询理解
│   │   └── citation.py       # 引用标注
│   │
│   ├── persona/              # 人格模拟 (Phase 3)
│   │   ├── __init__.py
│   │   ├── data_prep.py      # 微调数据准备
│   │   ├── finetune.py       # 微调脚本
│   │   └── inference.py      # 人格推理
│   │
│   └── agent/                # 代理交互 (Phase 4)
│       ├── __init__.py
│       ├── voice.py          # 语音克隆
│       └── deploy.py         # 多平台部署
│
├── web/                      # Web UI
│   └── app.py                # Gradio/Streamlit 应用
│
├── tests/                    # 测试 (TDD)
│   ├── __init__.py
│   ├── test_ingestion.py
│   ├── test_storage.py
│   ├── test_retrieval.py
│   ├── test_qa.py
│   └── test_persona.py
│
├── scripts/                  # 工具脚本
│   ├── setup_ollama.sh       # Ollama + 模型安装
│   └── benchmark.py
│
└── docs/                     # 文档
    ├── architecture.md
    └── deployment.md
```

## 8. 测试策略 (TDD)

### 8.1 测试原则

遵循 TDD：先写测试，看它失败，再写最小实现。

### 8.2 测试分层

| 层级 | 测试类型 | 工具 | 覆盖目标 |
|------|----------|------|----------|
| 单元测试 | 纯函数逻辑 | pytest | 数据模型、解析、融合算法 |
| 集成测试 | 组件交互 | pytest + testcontainers | 存储-检索管道 |
| 端到端测试 | 完整流程 | pytest | 采集→存储→检索→问答 |
| 评估测试 | 效果质量 | 自定义评估脚本 | 检索准确率、风格一致性 |

### 8.3 关键测试用例示例

```python
# tests/test_storage.py
class TestMemoryStorage:
    def test_store_and_retrieve_memory_by_id(self):
        """存储一条记忆后，能通过 ID 取回"""
        memory = Memory(
            id="test-001",
            source="diary",
            content="今天去了公园",
            timestamp=datetime(2024, 1, 1),
        )
        store = SQLiteStore(":memory:")
        store.save(memory)

        retrieved = store.get("test-001")
        assert retrieved.content == "今天去了公园"
        assert retrieved.source == "diary"

    def test_search_by_time_range(self):
        """能按时间范围检索记忆"""
        store = SQLiteStore(":memory:")
        store.save(Memory(id="1", content="事件A", timestamp=datetime(2024, 1, 1), source="diary"))
        store.save(Memory(id="2", content="事件B", timestamp=datetime(2024, 6, 1), source="diary"))

        results = store.search_by_time(
            start=datetime(2024, 3, 1),
            end=datetime(2024, 12, 31),
        )
        assert len(results) == 1
        assert results[0].content == "事件B"


# tests/test_retrieval.py
class TestRetrievalFusion:
    def test_reciprocal_rank_fusion_combines_results(self):
        """RRF 融合多路检索结果"""
        vector_results = [("m1", 1), ("m2", 2), ("m3", 3)]
        keyword_results = [("m2", 1), ("m4", 2)]

        fused = reciprocal_rank_fusion([vector_results, keyword_results])

        # m2 在两路中都出现，应该排名靠前
        assert fused[0][0] == "m2"
```

### 8.4 评估基准

建立个人记忆评估基准：
- **记忆召回测试**：人工标注 100 个"我记得的事"，测试系统能否检索到
- **时序推理测试**：50 个时序问题，测试回答准确率
- **风格一致性测试**：盲测对比微调模型 vs 基础模型的回答风格

## 9. 可行性分析

### 9.1 技术可行性

| 组件 | 成熟度 | 风险 | 缓解措施 |
|------|--------|------|----------|
| Screenpipe 集成 | 高 (16k stars) | 低 | 有完整 API 和文档 |
| sqlite-vec 向量检索 | 高 (SQLite 官方扩展) | 低 | 零依赖，10万条以内够用 |
| 自研时序图谱引擎 | 中 (借鉴 Zep 设计) | 中 | 先实现基础时序查询，逐步增强 |
| LLaMA Factory 微调 | 高 (WeClone 验证) | 低 | 文档完善，社区活跃 |
| CharLoRA | 中 (学术论文) | 中 | 可先用标准 LoRA，后续升级 |
| Spark-TTS 语音 | 中 | 中 | 可选 GPT-SoVITS 替代 |
| Ollama 本地推理 | 高 (生产级) | 低 | 零费用，一键部署 |

### 9.2 资源可行性

**硬件需求 (Phase 1-2)**：
- CPU: 8 核以上
- 内存: 16GB 以上
- 存储: 100GB (10万条记忆估算)
- GPU: 不需要 (使用 API 或轻量本地模型)

**硬件需求 (Phase 3 微调)**：
- GPU: RTX 4090 (24GB) 或云端 A100 租用
- 14B 模型 LoRA 微调约需 32GB 显存

**硬件需求 (Phase 4 推理)**：
- GPU: RTX 3080 (10GB) 起步 (QLoRA 4-bit)
- 或使用 API 调用

### 9.3 数据可行性

- 微信聊天记录：PyWxDump 可导出，数据量通常足够
- 日记/笔记：取决于用户积累
- 屏幕记录：Screenpipe 持续采集，数据量大
- **关键约束**：微调至少需要 5000+ 条高质量对话数据

### 9.4 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| 数据隐私泄露 | 高 | 本地优先，敏感数据脱敏，不强制上云 |
| 微调过拟合 | 中 | 留验证集，监控风格 vs 知识平衡 |
| 自研时序引擎复杂度 | 中 | Phase 1 先用简单时序索引，Phase 2 逐步增强图谱能力 |
| 长期维护成本 | 中 | 模块化设计，组件可替换 |
| 本地 LLM 性能不足 | 中 | 可选云端 API 作为后备，但非必须 |

## 10. 实施路线图

### Phase 1: 记忆引擎 (MVP)
- Week 1-2: 项目骨架 + 数据模型 + SQLite 存储 + FTS5
- Week 3-4: Markdown 日记采集器 + CLI 工具
- Week 5-6: sqlite-vec 向量索引 + 语义检索
- Week 7: Screenpipe 集成
- Week 8: 微信聊天记录导入

### Phase 2: 记忆检索
- Week 9-10: RAG 管道 + 查询理解
- Week 11-12: 自研时序图谱引擎 (基础版)
- Week 13: 检索融合 + 重排序
- Week 14: Web UI (Gradio) + 引用标注

### Phase 3: 思维模拟
- Week 15-16: 微调数据准备管道
- Week 17-18: LoRA 微调 (基础版)
- Week 19-20: CharLoRA 升级 + 评估

### Phase 4: 代理交互
- Week 21-22: 语音克隆
- Week 23-24: 多平台部署

## 11. 配置示例

```jsonc
// settings.jsonc
{
  "storage": {
    "sqlite_path": "./data/chronipedia.db",
    "sqlite_vec_path": "./data/chronipedia_vec.db"
  },
  "embedding": {
    "provider": "local",  // local (通过 Ollama)
    "model": "bge-m3",
    "ollama_base_url": "http://localhost:11434"
  },
  "llm": {
    "provider": "local",  // local (Ollama，零费用)
    "model": "qwen2.5:14b",
    "ollama_base_url": "http://localhost:11434"
  },
  "ingestion": {
    "screenpipe_db_path": "~/.screenpipe/data/db.sqlite",
    "wechat_csv_dir": "./data/wechat_exports",
    "diary_dir": "./data/diaries"
  },
  "privacy": {
    "filter_patterns": ["phone", "id_card", "email"],
    "redact_names": false
  }
}
```
