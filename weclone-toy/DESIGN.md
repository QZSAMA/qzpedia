# WeClone-Toy — 快速数字分身玩具项目设计文档

> "先跑起来，再优化。"

## 1. 项目概述

### 1.1 定位

WeClone-Toy 是一个**快速验证型玩具项目**，基于 WeClone 开源项目搭建，目标是**最快速度**产出一个能模仿你说话风格的数字分身。

它不是 Chronipedia 的替代品，而是一个"快速出效果"的实验场——让你先体验"和自己的数字分身对话"是什么感觉。

### 1.2 与 Chronipedia 的关系

| 维度 | WeClone-Toy | Chronipedia |
|------|-------------|-------------|
| 目标 | 快速出分身，体验效果 | 完整数字分身系统 |
| 周期 | 1-2 天搭建 | 数月迭代 |
| 数据 | 仅聊天记录 | 全类型数据 |
| 能力 | 模仿说话风格 | 记忆+思维+交互 |
| 架构 | 基于 WeClone | 自研架构 |
| 用途 | 玩具/验证 | 长期项目 |

### 1.3 核心价值

- **快速验证**：1-2 天内就能和"自己的分身"对话
- **低门槛**：消费级显卡即可运行 (QLoRA 4-bit, 6GB 显存)
- **可迁移**：积累的数据和经验可迁移到 Chronipedia Phase 3

## 2. 技术架构

### 2.1 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                  WeClone-Toy 架构                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ 数据导出  │───▶│ 数据预处理│───▶│ 模型微调  │          │
│  │ PyWxDump │    │ 清洗/格式化│    │ LoRA/SFT │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│                                       │                  │
│                                       ▼                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ 对话界面  │◀───│ 推理服务  │◀───│ 模型部署  │          │
│  │ Web/CLI  │    │ FastAPI  │    │ vLLM     │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│                                                          │
│  可选扩展:                                               │
│  ┌──────────┐    ┌──────────┐                          │
│  │ 语音克隆  │    │ 平台部署  │                          │
│  │Spark-TTS │    │ Telegram │                          │
│  └──────────┘    └──────────┘                          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 2.2 技术栈

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| 基础项目 | WeClone | latest | 一站式方案，16k+ stars 验证 |
| 基础模型 | Qwen2.5-7B-Instruct | 7B | 默认配置，效果够用 |
| 微调框架 | LLaMA Factory | latest | WeClone 底层依赖 |
| 微调方法 | QLoRA 4-bit | - | 6GB 显存可用 |
| 数据导出 | PyWxDump | latest | 微信聊天记录导出 |
| 推理服务 | vLLM / transformers | - | 高效推理 |
| Web UI | Gradio | 4.x | 快速搭建界面 |
| API 服务 | FastAPI | 0.100+ | 轻量 API |
| 语音克隆 | Spark-TTS (可选) | - | 增强真实感 |
| 包管理 | uv | latest | 快速安装 |

### 2.3 硬件需求

| 配置 | 显存 | 适用模型 | 效果预期 |
|------|------|----------|----------|
| 最低 (QLoRA 4-bit) | 6GB | 7B | 可用，效果一般 |
| 推荐 (QLoRA 4-bit) | 12GB | 7B-14B | 效果较好 |
| 理想 (LoRA 16-bit) | 32GB | 14B+ | 效果优秀 |

> 无 GPU 时可使用云端 Colab/Kaggle/AutoDL 进行微调，本地仅推理。

## 3. 实施流程

### 3.1 流程总览

```
Step 1: 环境准备        Step 2: 数据准备        Step 3: 模型微调
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ 克隆 WeClone │       │ 导出微信记录 │       │ 配置训练参数 │
│ 安装依赖     │──────▶│ 数据清洗     │──────▶│ 启动微调     │
│ 下载基础模型 │       │ 格式转换     │       │ 监控训练     │
└─────────────┘       └─────────────┘       └─────────────┘
                                                    │
                                                    ▼
Step 6: 体验对话        Step 5: 部署服务        Step 4: 模型测试
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ Web UI 对话  │◀──────│ 启动 API    │◀──────│ CLI 测试    │
│ 分身互动     │       │ 加载微调模型 │       │ 风格验证    │
└─────────────┘       └─────────────┘       └─────────────┘
```

### 3.2 Step 1: 环境准备

```bash
# 1. 克隆 WeClone
git clone https://github.com/xming521/WeClone.git weclone-toy
cd weclone-toy

# 2. 创建虚拟环境 (使用 uv)
uv venv .venv --python 3.10
source .venv/bin/activate

# 3. 安装依赖
uv pip install --group main -e .

# 4. 安装 LLaMA Factory (最新版)
uv pip install --upgrade git+https://github.com/hiyouga/LLaMA-Factory.git

# 5. 下载基础模型
git lfs install
git clone https://www.modelscope.cn/Qwen/Qwen2.5-7B-Instruct.git ./models/Qwen2.5-7B

# 6. 复制配置模板
cp settings.template.jsonc settings.jsonc
```

### 3.3 Step 2: 数据准备

#### 3.3.1 导出微信聊天记录

```bash
# 1. 使用 PyWxDump 导出
# 下载: https://github.com/xaoyaoo/PyWxDump
# 将手机聊天记录迁移到电脑微信
# 使用 PyWxDump 解密数据库并导出为 CSV

# 2. 将 CSV 放入项目目录
cp ~/Downloads/wechat_export.csv ./data/csv/
```

#### 3.3.2 数据预处理

WeClone 自动处理：
- CSV → JSON 格式转换
- 隐私信息过滤 (手机号、身份证、邮箱)
- 对话分段与时间戳保留
- 训练集/验证集分割

```bash
# 运行预处理
python src/data_process.py
```

#### 3.3.3 数据质量检查

```python
# scripts/check_data.py
"""检查微调数据质量"""
import json
from collections import Counter

def check_dataset(path: str):
    with open(path) as f:
        data = [json.loads(line) for line in f]

    print(f"总样本数: {len(data)}")

    # 检查对话长度分布
    lengths = [len(d["conversations"]) for d in data]
    print(f"平均对话轮数: {sum(lengths)/len(lengths):.1f}")

    # 检查内容长度
    char_counts = [len(c["content"]) for d in data for c in d["conversations"]]
    print(f"平均内容长度: {sum(char_counts)/len(char_counts):.1f} 字符")

    # 检查是否有过短样本
    short = sum(1 for c in char_counts if c < 5)
    print(f"过短样本 (<5字符): {short} ({short/len(char_counts)*100:.1f}%)")

    # 数据量评估
    if len(data) < 1000:
        print("⚠️  警告: 数据量较少 (<1000)，微调效果可能不佳")
    elif len(data) < 5000:
        print("💡 数据量中等 (1000-5000)，基本可用")
    else:
        print("✅ 数据量充足 (>5000)，预期效果较好")

if __name__ == "__main__":
    check_dataset("./data/train.json")
```

### 3.4 Step 3: 模型微调

#### 3.4.1 配置文件

```jsonc
// settings.jsonc (关键配置)
{
  "model_name_or_path": "./models/Qwen2.5-7B-Instruct",
  "dataset": "weclone",  // WeClone 预定义数据集
  "finetuning_type": "lora",
  "lora_rank": 16,
  "lora_alpha": 32,
  "lora_dropout": 0.05,
  "quantization_bit": 4,  // QLoRA 4-bit, 显存不足时使用
  "num_train_epochs": 3,
  "per_device_train_batch_size": 2,
  "gradient_accumulation_steps": 8,
  "learning_rate": 2e-4,
  "warmup_ratio": 0.03,
  "output_dir": "./checkpoints/weclone-lora"
}
```

#### 3.4.2 启动微调

```bash
# 使用 LLaMA Factory CLI
llamafactory-cli train settings.jsonc

# 或使用 WeClone 封装脚本
python src/train_sft.py
```

#### 3.4.3 训练监控

```bash
# 使用 TensorBoard 监控
tensorboard --logdir ./checkpoints/weclone-lora
```

关注指标：
- `loss`: 应持续下降，最终收敛
- `eval_loss`: 验证集损失，不应反弹（过拟合信号）
- `learning_rate`: 学习率调度曲线

### 3.5 Step 4: 模型测试

#### 3.5.1 CLI 测试

```bash
python src/cli_demo.py
# 进入交互式对话，测试分身效果
```

#### 3.5.2 风格一致性测试

```python
# scripts/style_test.py
"""分身风格一致性测试"""
import requests

API_URL = "http://localhost:8000/v1/chat/completions"

TEST_PROMPTS = [
    "你好啊，最近怎么样？",
    "周末有什么计划？",
    "我今天好累啊",
    "你觉得这个想法怎么样？",
    "帮我出个主意，朋友过生日送什么好",
]

def test_style():
    for prompt in TEST_PROMPTS:
        resp = requests.post(API_URL, json={
            "model": "weclone-toy",
            "messages": [{"role": "user", "content": prompt}],
            "temperature": 0.7,
        })
        answer = resp.json()["choices"][0]["message"]["content"]
        print(f"Q: {prompt}")
        print(f"A: {answer}\n")

if __name__ == "__main__":
    test_style()
```

### 3.6 Step 5: 部署服务

#### 3.6.1 API 服务

```bash
# 启动 FastAPI 服务
python src/api_service.py
# 默认监听 http://localhost:8000
```

#### 3.6.2 Web UI

```bash
# 启动 Gradio Web 界面
python src/web_demo.py
# 访问 http://localhost:7860
```

### 3.7 Step 6 (可选): 语音克隆与平台部署

#### 3.7.1 语音克隆

```bash
# 使用 WeClone-audio 子模块
cd WeClone-audio

# 需要 5 秒以上的语音样本
python voice_clone.py --sample ./voice_sample.wav --text "你好，我是你的数字分身"
```

#### 3.7.2 Telegram 部署

```bash
# 配置 Telegram Bot Token
export TELEGRAM_BOT_TOKEN="your-token"

# 启动 Telegram 机器人
python src/telegram_bot.py
```

## 4. 项目结构

```
weclone-toy/
├── README.md
├── DESIGN.md                 # 本文档
├── settings.jsonc            # WeClone 配置
│
├── data/                     # 数据目录
│   ├── csv/                  # 微信导出的 CSV
│   └── train.json            # 预处理后的训练数据
│
├── models/                   # 模型目录
│   └── Qwen2.5-7B-Instruct/  # 基础模型
│
├── checkpoints/              # 微调检查点
│   └── weclone-lora/
│
├── scripts/                  # 自定义脚本
│   ├── check_data.py         # 数据质量检查
│   ├── style_test.py         # 风格测试
│   └── export_model.py       # 导出合并模型
│
├── tests/                    # 测试
│   ├── test_data_prep.py     # 数据预处理测试
│   └── test_api.py           # API 测试
│
└── docs/                     # 文档
    ├── setup_guide.md        # 详细部署指南
    └── troubleshooting.md    # 常见问题
```

## 5. 测试策略

### 5.1 测试原则

虽然是玩具项目，但关键环节仍需测试验证，遵循 TDD 核心思想。

### 5.2 测试用例

```python
# tests/test_data_prep.py
import pytest
import json
import tempfile
import os

class TestDataPreprocessing:
    def test_csv_to_json_conversion(self):
        """CSV 聊天记录能正确转为 JSON 训练格式"""
        csv_content = """时间,发送人,内容,类型
2024-01-01 10:00:00,我,你好,True
2024-01-01 10:00:05,对方,嗨,True
2024-01-01 10:00:10,我,最近怎么样,True
"""
        with tempfile.NamedTemporaryFile(mode="w", suffix=".csv", delete=False) as f:
            f.write(csv_content)
            csv_path = f.name

        result = preprocess_wechat_csv(csv_path)

        assert len(result) > 0
        assert "conversations" in result[0]
        assert result[0]["conversations"][0]["content"] == "你好"

        os.unlink(csv_path)

    def test_privacy_filter_removes_phone_numbers(self):
        """隐私过滤去除手机号"""
        text = "我的电话是 13812345678，记得打给我"
        filtered = filter_privacy(text)
        assert "13812345678" not in filtered
        assert "***" in filtered

    def test_short_messages_filtered(self):
        """过短消息被过滤"""
        conversations = [
            {"role": "user", "content": "嗯"},
            {"role": "assistant", "content": "好"},
            {"role": "user", "content": "今天天气真好啊"},
            {"role": "assistant", "content": "是啊，适合出去走走"},
        ]
        filtered = filter_short_conversations(conversations, min_length=3)
        assert len(filtered) == 2


# tests/test_api.py
import pytest
from fastapi.testclient import TestClient

class TestCloneAPI:
    def test_health_check(self):
        """API 健康检查"""
        client = TestClient(app)
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json()["status"] == "ok"

    def test_chat_endpoint_returns_response(self):
        """聊天接口返回响应"""
        client = TestClient(app)
        response = client.post("/v1/chat/completions", json={
            "model": "weclone-toy",
            "messages": [{"role": "user", "content": "你好"}],
        })
        assert response.status_code == 200
        assert "choices" in response.json()
        assert len(response.json()["choices"]) > 0
```

### 5.3 效果评估

```python
# scripts/evaluate.py
"""分身效果评估脚本"""
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

class CloneEvaluator:
    """评估微调模型的风格一致性"""

    def __init__(self, base_model_path, finetuned_model_path):
        self.tokenizer = AutoTokenizer.from_pretrained(base_model_path)

        # 加载基础模型
        self.base_model = AutoModelForCausalLM.from_pretrained(
            base_model_path, torch_dtype=torch.float16, device_map="auto"
        )

        # 加载微调模型
        self.finetuned_model = AutoModelForCausalLM.from_pretrained(
            finetuned_model_path, torch_dtype=torch.float16, device_map="auto"
        )

    def compare_responses(self, prompt: str):
        """对比基础模型和微调模型的回答"""
        inputs = self.tokenizer(prompt, return_tensors="pt").to(self.base_model.device)

        base_output = self.base_model.generate(**inputs, max_new_tokens=200)
        finetuned_output = self.finetuned_model.generate(**inputs, max_new_tokens=200)

        return {
            "prompt": prompt,
            "base_response": self.tokenizer.decode(base_output[0]),
            "finetuned_response": self.tokenizer.decode(finetuned_output[0]),
        }

    def evaluate_style_consistency(self, test_prompts: list[str]):
        """评估风格一致性"""
        results = []
        for prompt in test_prompts:
            comparison = self.compare_responses(prompt)
            results.append(comparison)
            print(f"\nPrompt: {prompt}")
            print(f"Base:     {comparison['base_response'][:100]}...")
            print(f"Tuned:    {comparison['finetuned_response'][:100]}...")

        return results
```

## 6. 可行性分析

### 6.1 技术可行性

| 环节 | 可行性 | 依据 |
|------|--------|------|
| WeClone 部署 | ✅ 高 | 16k+ stars，大量成功案例 |
| 微信数据导出 | ✅ 高 | PyWxDump 成熟工具 |
| QLoRA 微调 | ✅ 高 | LLaMA Factory 文档完善 |
| 7B 模型推理 | ✅ 高 | 消费级显卡可运行 |
| 语音克隆 | ⚠️ 中 | Spark-TTS 效果因人而异 |
| Telegram 部署 | ✅ 高 | 标准 Bot API |

### 6.2 数据可行性

**最低数据要求**：
- 单个联系人 500+ 条消息 → 可训练，效果一般
- 单个联系人 2000+ 条消息 → 效果较好
- 多个联系人总计 5000+ 条消息 → 效果优秀

**数据质量要求**：
- 1对1对话优于群聊（群聊会混淆风格）
- 文字消息优于语音/表情包
- 近期数据优于远古数据（风格会变化）

### 6.3 时间可行性

| 步骤 | 预估时间 | 备注 |
|------|----------|------|
| 环境准备 | 1-2 小时 | 含模型下载 |
| 数据导出 | 30 分钟 | PyWxDump 操作 |
| 数据预处理 | 30 分钟 | 自动化处理 |
| 模型微调 | 2-4 小时 | 7B QLoRA, 取决于数据量 |
| 测试部署 | 1 小时 | CLI + Web UI |
| **总计** | **5-8 小时** | 一天内可完成 |

### 6.4 已知限制

1. **仅模仿说话风格**：不具备真正的记忆能力，不知道"你做过什么"
2. **7B 模型效果有限**：WeClone 官方文档提示 7B 效果一般，14B+ 更好
3. **群聊数据质量差**：建议只用 1对1 对话数据
4. **Windows 支持有限**：建议使用 Linux 或 WSL2
5. **无长期记忆**：每次对话独立，不记得之前聊过什么（除非接入 Mem0）

### 6.5 风险与缓解

| 风险 | 缓解措施 |
|------|----------|
| 微信封号 | 使用小号测试，WeClone 官方建议 |
| 显存不足 | 降级到 QLoRA 4-bit 或更小模型 |
| 数据量不足 | 先用公开对话数据测试流程 |
| 效果不理想 | 升级到 14B 模型，增加数据量 |

## 7. 从 Toy 到 Chronipedia 的迁移路径

WeClone-Toy 积累的资产可迁移到 Chronipedia：

```
WeClone-Toy 产出                    Chronipedia 复用
─────────────────                   ─────────────────
微信聊天记录 (清洗后)        ──▶    Phase 1 数据源
微调数据集 (SFT 格式)        ──▶    Phase 3 微调数据
LoRA 适配器权重              ──▶    Phase 3 人格模型 (初始版)
风格评估方法                 ──▶    Phase 3 评估基准
部署经验 (API/Web)           ──▶    Phase 4 部署参考
```

## 8. 快速启动检查清单

### 8.1 环境检查

- [ ] Python 3.10+ 已安装
- [ ] uv 已安装 (`pip install uv`)
- [ ] CUDA 12.4+ 已安装 (GPU 用户)
- [ ] git lfs 已安装 (下载模型)
- [ ] 显存 >= 6GB (QLoRA 4-bit) 或 >= 16GB (LoRA)

### 8.2 数据检查

- [ ] 微信聊天记录已迁移到电脑
- [ ] PyWxDump 已安装并能解密数据库
- [ ] 至少 1 个联系人有 500+ 条消息
- [ ] CSV 文件已放入 `data/csv/` 目录

### 8.3 模型检查

- [ ] Qwen2.5-7B-Instruct 已下载到 `models/` 目录
- [ ] `settings.jsonc` 已配置正确的模型路径
- [ ] CUDA 可用: `python -c "import torch; print(torch.cuda.is_available())"`

### 8.4 部署检查

- [ ] 微调完成，检查点在 `checkpoints/` 目录
- [ ] CLI 测试通过: `python src/cli_demo.py`
- [ ] API 服务启动: `python src/api_service.py`
- [ ] Web UI 可访问: `http://localhost:7860`
