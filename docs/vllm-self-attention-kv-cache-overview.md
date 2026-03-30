# vLLM：Self-Attention、Q/K/V、Prefill/Decode 与 KV Cache 总览

> 建议插入位置：`_posts/2025-3-28-vLLM：高性能大语言模型推理框架源码解析与最佳实践.md` 的 `### 5.4 KV 缓存管理` 小节下，放在 `#### KV 缓存基本原理` 之前。

在阅读 vLLM 源码时，很多读者会把几个概念混在一起：Self-Attention、Q/K/V、KV Cache、Prefill、Decode、词表采样。实际上它们共同组成了一条非常清晰的链路：**输入先被 token 化并经过多层 Self-Attention 形成上下文化表示；随后模型基于最后一个位置的隐藏状态，在词表上预测下一个 token；vLLM 则通过分页式 KV Cache 管理，把这一过程做得足够快、足够省显存。**

---

## 1. 从输入到输出的一张总图

```text
用户输入文本
   │
   ▼
Tokenizer / 词表切分
   │
   ▼
Token IDs
   │
   ▼
Embedding + Position Encoding
   │
   ▼
每一层 Transformer Block
   │
   ├─ 1) 对每个 token 的输入表示 x 做三次线性映射
   │      Q = xW_Q
   │      K = xW_K
   │      V = xW_V
   │
   ├─ 2) Self-Attention
   │      当前 token 的 Q
   │         ×
   │      所有 token 的 K
   │         ↓
   │      相关性分数
   │         ↓ softmax
   │      注意力权重
   │         ↓
   │      对所有 V 加权求和
   │
   └─ 3) 得到“结合上下文后的 token 表示”
            ↓
      残差连接 + LayerNorm + FFN
            ↓
      进入下一层
   │
   ▼
最后一层最后一个位置的 Hidden State
   │
   ▼
LM Head（线性投影到整个词表）
   │
   ▼
Vocab Logits / 每个 token 的分数
   │
   ▼
Softmax + Sampling（greedy / top-k / top-p / temperature）
   │
   ▼
选出下一个 token
   │
   ▼
拼回上下文，继续下一轮 Decode
```

这张图对应的核心结论是：

1. **Self-Attention 负责“看懂上下文”**，不是直接去词表选 token。
2. **LM Head + Sampling 负责“从词表里挑下一个 token”**。
3. **KV Cache 负责把历史 K/V 存下来，避免 Decode 阶段重复计算**。

---

## 2. Self-Attention 到底在做什么

Self-Attention 并不是直接输出“用户意图”这个标签，而是在构建上下文化语义表示。它做的事情是：

- 让每个 token 看到同一句话里的其他 token
- 建立 token 与 token 之间的依赖关系
- 得到“这个 token 在当前上下文里的语义版本”

因此，更准确的说法不是“模型先识别了一个 intent 字段”，而是：

> 模型先通过多层 Self-Attention，把输入序列变成一组包含上下文关系的隐藏状态；这些隐藏状态里隐含了任务、约束、语义重点和上下文依赖。

---

## 3. Q/K/V 是怎么来的

对某个 token 的输入表示 `x`，模型会通过三组不同的参数分别计算：

```python
Q = x @ W_Q
K = x @ W_K
V = x @ W_V
```

可以用下面的直觉来理解：

- **Q（Query）**：我现在想找什么信息
- **K（Key）**：我身上有哪些可被别人匹配的特征
- **V（Value）**：如果别人关注我，我真正提供什么内容

注意，Q/K/V 都来自同一个 token 的输入表示，但因为投影矩阵不同，最终承担的语义角色也不同。

---

## 4. Prefill 和 Decode 的区别

LLM 推理可以拆成两个阶段：

### Prefill

对整段已知输入一次性并行计算：

- 完整 prompt 进入 Transformer
- 计算所有已知 token 的 Q/K/V
- 建立上下文化表示
- 把各层历史 token 的 K/V 写入缓存

### Decode

每次只生成 1 个新 token：

- 当前步只对“新 token”计算新的 Q/K/V
- 用新 token 的 Q 去读取历史 K/V
- 得到当前 hidden state
- 投影到词表，选出下一个 token

压缩成一张图就是：

```text
[Prefill 阶段]
完整 Prompt
  └─ 并行通过多层 Transformer
      └─ 计算所有已知 token 的 Q/K/V
          └─ 把历史 K/V 写入 KV Cache

[Decode 阶段]
当前上下文 + 历史 KV Cache
  └─ 只对“新 token”计算 Q/K/V
      ├─ 用新 token 的 Q 读取历史 K/V
      ├─ 得到当前 hidden state
      ├─ 投影到词表得到 logits
      └─ 采样出下一个 token
```

---

## 5. KV Cache 为什么能提速

KV Cache 缓的不是所有东西，而是**历史 token 的 K/V**。

未来生成第 `t+1` 个 token 时，只需要：

- 当前 token 的 `Q_(t+1)`
- 历史所有 token 的 `K_1 ... K_t`
- 历史所有 token 的 `V_1 ... V_t`

历史 token 的 Q 已经完成过自己的注意力计算，后续基本不会再用到，因此**没有必要缓存历史 Q**。

所以 Decode 阶段每一步的真实情况是：

- **当前步现算**：`Q_t, K_t, V_t`
- **历史部分复用**：`K_1 ... K_(t-1), V_1 ... V_(t-1)`

这也解释了为什么：

- **有 KV Cache**：只增量计算新 token，速度快
- **没有 KV Cache**：每一步都要重算历史，速度慢

从工程角度看，KV Cache 本质上就是：

> **用显存换速度。**

---

## 6. 输出 token 在哪里挑、怎么挑

很多人会误以为“Self-Attention 最后直接产出下一个 token”，其实不是。

真正的流程是：

1. Transformer 把上下文算成最后一个位置的 hidden state
2. `LM Head` 把这个 hidden state 线性投影到整个词表
3. 得到词表中每个 token 的分数（logits）
4. 经过 softmax 和采样策略，挑出一个 token

也就是说：

```text
Self-Attention：负责把上下文看明白
LM Head：负责把最后位置映射到整个 vocab
Sampling：负责从 vocab 分布里选一个 token
```

因此，模型输出并不是凭空“发明”字符，而是：

> **每一步都从 tokenizer 的词表里选出一个 token，然后不断拼回上下文。**

---

## 7. 词表决定什么，不决定什么

词表（Vocabulary）确实会影响两个方面：

### 词表决定的

1. **输入如何被切分成 token**
2. **输出时每一步可以选择哪些 token**

### 词表不直接决定的

1. 模型能否理解复杂语义
2. 模型能否建立长距离依赖
3. 模型能否完成推理与泛化

更准确地说：

> 词表决定“语言被拆成什么单位来处理”，而模型参数与训练过程决定“这些单位最终能被理解到什么程度”。

---

## 8. 把这部分和 vLLM 的源码设计对上

如果从 vLLM 的工程实现视角看，上面的整条链路可以再压缩为：

```text
文本输入
  ↓
Tokenizer
  ↓
Prefill（并行）
  ↓
生成所有已知 token 的 K/V
  ↓
PagedAttention / Block Table / KV Blocks
  ↓
Decode（单步迭代）
  ↓
复用历史 KV Cache
  ↓
LM Head 投影到 vocab
  ↓
采样下一个 token
  ↓
持续追加到 KV Cache
```

这也正是为什么 vLLM 的优化重点会落在：

- **PagedAttention**
- **KV Block 管理**
- **Prefix Caching**
- **Continuous Batching**

因为在大规模在线推理里，真正决定吞吐和显存效率的，并不是“模型会不会算 Attention”，而是：

1. 历史 K/V 如何存
2. 如何复用
3. 如何避免碎片
4. 如何支持动态批处理

vLLM 的价值，本质上就是把标准 Transformer 推理流程做成了一套高效的工程系统。

---

## 9. 一句话总结

> **Self-Attention 负责理解上下文，Q/K/V 是注意力计算的内部表示，Prefill 负责把已知输入一次算完并建立缓存，Decode 负责基于缓存逐 token 生成，而 vLLM 则通过 PagedAttention 和 KV Cache 管理，把这条链路的性能做到了工程最优。**
