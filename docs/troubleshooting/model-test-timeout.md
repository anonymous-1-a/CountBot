# 模型测试卡死问题排查

## 问题描述

在测试模型连接时（特别是国内 AI 厂商的模型），界面卡死，日志显示：

```
'timed out' thrown while requesting HEAD https://huggingface.co/MiniMax-M2.5/resolve/main/tokenizer.json
Retrying in 1s [Retry 1/5].
```

## 根本原因

### 问题链路

1. **用户操作**：在前端测试模型连接（如 MiniMax-M2.5）
2. **后端处理**：调用 LiteLLM 发送测试请求
3. **LiteLLM 行为**：尝试使用 tiktoken 计算 token 数量
4. **tiktoken 行为**：发现不认识的模型名称，尝试从 HuggingFace 下载 tokenizer
5. **网络问题**：HuggingFace 在国内访问很慢或无法访问
6. **结果**：请求超时，不断重试（最多 5 次），每次等待 1 秒
7. **用户体验**：界面卡死 5-10 秒

### 为什么之前正常？

- 使用 OpenAI 标准模型（如 gpt-3.5-turbo）时，tiktoken 已内置 tokenizer
- 不需要从网络下载，所以不会卡死

### 为什么国内模型会触发？

- 国内 AI 厂商的模型名称（如 MiniMax-M2.5）不在 tiktoken 的内置列表中
- tiktoken 会尝试从 HuggingFace 下载对应的 tokenizer
- 但这些模型的 tokenizer 可能不存在于 HuggingFace

---

## 解决方案

### 方案 1：禁用 tiktoken 网络请求（已实施）

在 `backend/modules/providers/litellm_provider.py` 中添加环境变量：

```python
# 在 __init__ 方法中
os.environ["TRANSFORMERS_OFFLINE"] = "1"
os.environ["HF_HUB_OFFLINE"] = "1"

# 在 _suppress_litellm_logging 方法中
os.environ["TRANSFORMERS_OFFLINE"] = "1"
os.environ["HF_HUB_OFFLINE"] = "1"
os.environ["HF_DATASETS_OFFLINE"] = "1"
```

**效果**：
- tiktoken 不会尝试从 HuggingFace 下载
- 如果找不到 tokenizer，会跳过 token 计数
- 不影响模型调用

---

### 方案 2：设置 tiktoken 缓存（已实施）

```python
os.environ["TIKTOKEN_CACHE_DIR"] = os.path.join(os.path.expanduser("~"), ".cache", "tiktoken")
```

**效果**：
- tiktoken 会将下载的 tokenizer 缓存到本地
- 下次使用时不需要重新下载

---

### 方案 3：使用代理（可选）

如果需要使用 tiktoken 的完整功能，可以设置代理：

```bash
# 在启动前设置环境变量
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890

# 或在代码中设置
os.environ["HTTP_PROXY"] = "http://127.0.0.1:7890"
os.environ["HTTPS_PROXY"] = "http://127.0.0.1:7890"
```

---

## 验证修复

### 测试步骤

1. 重启 CountBot
2. 在前端测试国内 AI 厂商的模型（如 MiniMax-M2.5）
3. 观察是否还有 HuggingFace 的请求日志
4. 确认测试能快速完成（1-2 秒内）

### 预期结果

```
2026-03-04 01:02:40 | INFO | Testing connection to custom_openai with model MiniMax-M2.5
2026-03-04 01:02:40 | INFO | Using Custom API (OpenAI), model: MiniMax-M2.5
2026-03-04 01:02:40 | INFO | Calling LiteLLM: MiniMax-M2.5
2026-03-04 01:02:41 | INFO | Stream finished with reason: length
2026-03-04 01:02:41 | INFO | Connection test successful
```

**不应该出现**：
```
'timed out' thrown while requesting HEAD https://huggingface.co/...
```

---

## 相关配置

### 环境变量说明

| 环境变量 | 作用 | 默认值 |
|---------|------|--------|
| `TRANSFORMERS_OFFLINE` | 禁用 transformers 库的网络请求 | 0 |
| `HF_HUB_OFFLINE` | 禁用 HuggingFace Hub 的网络请求 | 0 |
| `HF_DATASETS_OFFLINE` | 禁用 HuggingFace Datasets 的网络请求 | 0 |
| `TIKTOKEN_CACHE_DIR` | tiktoken 缓存目录 | 系统临时目录 |
| `LITELLM_LOCAL_MODEL_COST_MAP` | 使用本地模型价格映射 | False |

### LiteLLM 配置

```python
litellm.suppress_debug_info = True
litellm.set_verbose = False
litellm.drop_params = True
litellm.telemetry = False
litellm.turn_off_message_logging = True
```

---

## 常见问题

### Q1: 禁用 tiktoken 会影响功能吗？

**A**: 不会。tiktoken 主要用于：
- 计算 token 数量（用于成本估算）
- 验证输入长度

禁用后：
- LiteLLM 会跳过 token 计数
- 直接发送请求到 AI 服务商
- AI 服务商会自己处理 token 限制

### Q2: 为什么不直接移除 tiktoken 依赖？

**A**: 因为：
- LiteLLM 依赖 tiktoken
- 对于 OpenAI 等标准模型，tiktoken 很有用
- 只需要禁用网络请求，不需要完全移除

### Q3: 如何查看 tiktoken 缓存？

**A**: 
```bash
# 查看缓存目录
ls -la ~/.cache/tiktoken/

# 清空缓存
rm -rf ~/.cache/tiktoken/
```

### Q4: 其他模型也会有这个问题吗？

**A**: 可能会。任何不在 tiktoken 内置列表中的模型都可能触发：
- 国内 AI 厂商的模型（MiniMax、智谱、Kimi 等）
- 自定义模型
- 新发布的模型

---

## 技术细节

### tiktoken 的工作原理

1. 接收模型名称（如 "MiniMax-M2.5"）
2. 查找内置的 tokenizer 列表
3. 如果找不到，尝试从 HuggingFace 下载
4. 下载 URL 格式：`https://huggingface.co/{model_name}/resolve/main/tokenizer.json`
5. 如果下载失败，抛出异常或超时

### LiteLLM 的 token 计算流程

```python
# 简化的流程
def calculate_tokens(model, messages):
    try:
        encoding = tiktoken.encoding_for_model(model)
        return encoding.encode(messages)
    except:
        # 如果失败，跳过 token 计数
        return None
```

### 我们的修复策略

```python
# 设置离线模式
os.environ["TRANSFORMERS_OFFLINE"] = "1"
os.environ["HF_HUB_OFFLINE"] = "1"

# tiktoken 会检测到离线模式
# 如果找不到 tokenizer，直接返回 None
# LiteLLM 会跳过 token 计数，继续发送请求
```

---

## 相关文件

- `backend/modules/providers/litellm_provider.py` - LiteLLM Provider 实现
- `backend/api/settings.py` - 模型测试 API
- `requirements.txt` - 依赖配置

---

**最后更新**：2026-03-04
