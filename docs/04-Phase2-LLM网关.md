# 04 · Phase 2：把它变成 AI Gateway（M3 + M4 + M5）

> **目标：客户端以为自己在直接调用 OpenAI，实际上所有请求都在经过你的网关。**
>
> 关键设计约束：**对外必须是 OpenAI 兼容协议。** 这样任何现成客户端（ChatBox、LangChain、Cursor、Cherry Studio）都能零改造接你的网关——这个约束会逼你读规范，是好事。

---

## 一、这一阶段的路线

```text
M3  先不接任何真模型：自己写一个 fake provider，
    把「协议 + 路由 + 转发」整条链路跑通
        ↓
M4  换上真模型：Qwen / DeepSeek，配置化注册
        ↓
M5  流式（最难） + 重试 / 超时 / 降级
```

**千万不要跳过 M3 直接接真模型。** 真模型引入了 API Key、网络波动、额度限制、计费——这些噪音会淹没你本该学的东西（协议和抽象）。**M3 用假模型跑通全链路，是这一阶段最重要的一步。**

---

# M3：OpenAI 兼容入口 + Fake Provider（2 天）

## 任务卡

| 编号 | 任务 | 验收 |
|---|---|---|
| M3-1 | 建数据模型：`ChatCompletionRequest` / `Message` / `ChatCompletionResponse` / `Choice` / `Usage` | 能用 Jackson 正确序列化/反序列化一段真实的 OpenAI 请求 JSON |
| M3-2 | 暴露 `POST /v1/chat/completions`，接收并校验（`model`、`messages` 必填） | 缺字段能返回 `400` + OpenAI 风格错误体 |
| M3-3 | 定义 `LlmProvider` 接口（`name()` / `chat(req)` / `stream(req)`） | 接口只有一个实现，但**抽象先立住** |
| M3-4 | 写 `FakeProvider`：不回网络，返回构造好的标准响应，带真实的 `usage` | 客户端能像用 OpenAI 一样用你的网关 |
| M3-5 | `ModelRegistry`：`model` 名 → Provider 的映射表（先写死在 yml 里） | `model=fake-1` 走 FakeProvider |
| M3-6 | 未知模型 → `404`，错误体格式与 OpenAI 一致 | `model=不存在的` 返回规范错误 |

## 关键讲解点

### ① 为什么「OpenAI 兼容」这个约束这么重要

因为它把**协议**和**实现**分开了：

```text
客户端 ──OpenAI 协议──▶ 你的网关 ──各家私有 API──▶ Qwen / DeepSeek / vLLM
```

只要协议统一，你就能在网关里做路由、限流、降级、缓存——**客户端完全不用改**。
反过来，如果每个模型对外暴露不同协议，网关就变成了"格式转换器"，治理能力根本加不进去。

**这是整个项目最核心的一个架构决策。** 你要能解释它。

### ② 数据建模：用 `record` 还是 class？

Java 17 的 `record` 非常适合 DTO（不可变、自动 equals/hashCode、少写 50 行）。
但要注意：**反序列化时字段可选性**（`temperature`、`stream`、`top_p` 都可以不传）。用 `record` + `@JsonInclude(NON_NULL)` 处理。

**这里会撞到第一个真问题**：请求里字段有一二十个，你要全部建模吗？
答案：**只建你需要的**（`model`、`messages`、`stream`、`temperature`、`max_tokens`），其余字段**原样保留透传**。
怎么"原样保留"？——这是 M4 的一个绝妙设计题，先自己想。

### ③ 错误体必须和 OpenAI 格式一致

```json
{
  "error": {
    "message": "The model `gpt-fake` does not exist",
    "type": "invalid_request_error",
    "code": "model_not_found"
  }
}
```

**为什么较真到字段名**：客户端库是按这个结构解析错误的。格式不对，用户看到的报错就变成了"未知错误"，你的网关就成了"黑盒故障源"。

---

# M4：真实多模型路由（2 天）

## 任务卡

| 编号 | 任务 | 验收 |
|---|---|---|
| M4-1 | `ProviderProperties`：在 yml 里声明 provider（baseUrl / apiKey 环境变量名 / 超时 / 支持的模型） | 配置化，加一个模型不改代码 |
| M4-2 | `OpenAiCompatibleProvider`：用 `RestClient` 打上游的 `/v1/chat/completions` | 能真拿到 Qwen 的回答 |
| M4-3 | 把 `FakeProvider` 和真 Provider 共存于 Registry，按 `model` 路由 | `model=qwen` / `model=fake-1` 各走各的 |
| M4-4 | API Key 从**环境变量**读，`.gitignore` 兜底 | 仓库里搜不到 Key（`git log -p \| Select-String sk-`） |
| M4-5 | **未知字段透传**：客户端传的 `top_k`、`repetition_penalty` 等原样转给上游 | 上游能收到非标准字段 |
| M4-6 | 请求/响应日志：脱敏（Key、完整 prompt 要截断） | 日志里看不到完整 Key |

## 关键讲解点

### ① Model Registry：从"if-else"到"注册表"

新手写法：

```java
if ("qwen".equals(model)) { ... } else if ("deepseek".equals(model)) { ... }
```

**问题**：加第 6 个模型时要改代码、重新编译部署。

进阶写法：**配置驱动 + 启动时构建映射表**。

```text
model 名 ──▶ Registry 查表 ──▶ Provider 实例 ──▶ baseUrl + key + 超时
```

再进一步（Phase 3 会做）：Registry 里一个 model 对应**多个实例**（负载均衡），或者**别名**（`gpt-4o` 映射到你自己的 `qwen-max`，方便客户端切换）。

**这个演进过程（硬编码 → 配置化 → 动态化）就是 AI Infra 的成长路径本身。**

### ② 未知字段怎么"原样透传"（M4-5）

这是本阶段最漂亮的设计题。提示（L1，自己想）：

> Jackson 里有一个类型，可以接收**任意 JSON 结构**而不定义字段……它叫什么？把它作为 `Map<String, Object>` 的替代，配合一个"已知字段优先、其余合并"的策略，就能做到。

（自己查 `JsonNode` 和 `ObjectNode`。想不出来再来找我。）

**为什么这是个真问题**：OpenAI 协议在不停加字段，各家厂商还各有私有字段。你如果建模了所有字段，就永远在追着别人的更新跑；透传则天然面向未来。

### ③ API Key 管理

- ❌ 写进代码 → 进 Git → 泄漏 → 被人刷爆额度（真实案例很多）
- ✅ 环境变量 → IDE 里配 Run Configuration 的环境变量
- ✅ 或者 `application-local.yml`（**必须在 `.gitignore` 里**）

**再想一层**：Gateway 存在的意义之一就是"Key 不落地到客户端"。客户端只拿你的网关 Key，上游 Key 只存在网关里。这就是**凭证托管**，是网关的核心商业价值之一。

---

# M5：流式转发 + 容错（4 天，本阶段最硬的一周）

> 如果你前 3 周只能有一个"亮点"，那就应该是这个。**流式转发是"会不会做 LLM 网关"的分水岭。**

## 任务卡

| 编号 | 任务 | 验收 |
|---|---|---|
| M5-1 | 认识 SSE：先写一个最小的 SSE 服务端，用 curl 看原始输出 | `curl -N` 能看到 `data: xxx` 逐条出现 |
| M5-2 | 上游 `stream:true` 的原始响应长什么样（用 curl 直接打 Qwen 看） | 记录了真实的 chunk 格式到 notes |
| M5-3 | `stream:true` 时网关把上游的 SSE **边收边转**，逐块 flush 给客户端 | 客户端逐字出现，不是等几秒后一次性出现 |
| M5-4 | 正确处理 `data: [DONE]` 与结束 | 客户端能正常识别结束，连接被释放 |
| M5-5 | 客户端中途断开 → 网关也要掐断上游请求 | 客户端 Ctrl+C 后，上游连接不再挂着 |
| M5-6 | 流式下的错误：上游 429/500 时如何告诉已经开始的流 | 记录你选的方案与理由 |
| M5-7 | 重试策略：区分**能不能重试**（连不上 vs 已开始返回） | 不做"流已经吐了一半再重试"这种蠢事 |
| M5-8 | 超时分层：TTFT 超时 vs 总超时（流式不能用总超时打死） | 长回答不会被中途掐断 |
| M5-9 | 初始版熔断：连续 N 次失败 → 短暂摘除该 provider | 上游挂掉后请求快速失败，不再傻等 |
| M5-10 | `usage` 统计：prompt/completion token 记录到日志 | 每条请求都有 token 数 |

## 关键讲解点

### ① SSE 到底是什么（M5-1）

SSE = Server-Sent Events，就是"服务器在一个**不关闭的 HTTP 响应**里，持续往里写文本"。格式极简：

```text
data: {"choices":[{"delta":{"content":"你"}}]}

data: {"choices":[{"delta":{"content":"好"}}]}

data: [DONE]

```

规则：**一行以 `data: ` 开头，一个空行表示一条消息结束**；`\n\n` 是分隔符。

**所以"流式"不是什么高级魔法，就是"响应体没写完、连接不关、每来一块就 flush"。**

### ② 最大的技术难点：flush 与背压（M5-3）

新手必踩的坑：把上游所有 chunk 读完，再一次性返回。**这样结果是"假的流式"** ——你测的时候只有几十毫秒延迟，看起来对；一旦模型输出慢，用户体验和阻塞式一模一样。

要做到真流式，必须：

1. 上游响应用**流式读取**（`BodyHandlers.ofInputStream()` 或 `BodyHandlers.ofLines()`）
2. 网关向客户端输出也用**流式响应**（Spring MVC 里可以用 `SseEmitter` 或 `StreamingResponseBody`）
3. **每读一块就 flush 一次**（不 flush 会卡在缓冲区里）
4. 处理**背压**：客户端读得慢怎么办？（先"不管它"，但你要知道这是个问题——这是 Phase 3 讨论 WebFlux 的引子）

**"假流式"是这个阶段最经典的错误，没有之一。** 你的验收标准是：**`curl -N` 时字符要肉眼可见地一块块出现**，而不是"停顿 3 秒然后刷一下全出来"。

### ③ 什么能重试，什么不能（M5-7）

```text
连接失败 / 连接阶段超时     → 可以重试（可能换实例）
上游返回 429                → 可以重试（退避 + 换 key/实例）
上游返回 500                → 谨慎重试（可能上游真崩了，重试会雪上加霜）
上游返回 400/401/403        → 绝不重试（是你的请求/key 有问题，重试没用）
流式已经开始吐字            → 绝不能重试（除非你有办法告诉客户端"重来"）
```

**重试必须有退避（backoff）+ 上限 + 熔断**，否则上游一慢，你的重试风暴会把它彻底打死。这叫**重试放大**，是分布式系统的经典事故。

### ④ 流式下的超时（M5-8）

流式请求可能合法地持续好几分钟（生成 4000 个 token）。你 M2 学的"总超时 3 秒"在这里**必须换掉**，换成：

- **TTFT 超时**（Time To First Token）：多久没收到第一个字就放弃 → 比如 30s
- **空闲超时**：两个 chunk 之间最长间隔 → 比如 30s
- **总超时**：设一个比较大的兜底 → 比如 10 分钟

**"超时不是一个数，是一组数"** ——这个认知只有做过流式才会长出来。

---

## M5 验收清单

- [ ] Apifox/ChatBox 里配置你的网关为 OpenAI 兼容端点，能正常对话
- [ ] `stream:true` 时，`curl -N` 能**逐块**看到输出（不是"假流式"）
- [ ] `stream:false` 时行为正常
- [ ] 切换 `model=qwen` / `model=deepseek` / `model=fake-1` 都正常
- [ ] 关掉一个 provider 的 Key（配错）→ 返回规范错误，且**不重试**
- [ ] 上游 429 → 有退避重试，日志能看出重试了几次
- [ ] 客户端中断 → 日志显示上游连接被取消
- [ ] 每条请求日志里有 `prompt_tokens` / `completion_tokens` / 上游耗时 / TTFT
- [ ] commit：`feat(M5): 支持 SSE 流式转发与重试熔断`

---

## M3~M5 复盘题

1. 画出 `stream:true` 的完整数据流：上游 chunk → 网关 → 客户端，标出 flush 发生的位置
2. 为什么要区分 hop-by-hop 头在流式场景下更危险？
3. 客户端不认 OpenAI 协议的话，你要额外做什么？（提示：协议转换层放哪一层）
4. 现在你的网关能同时处理多少并发流式请求？瓶颈是线程还是内存？
5. 如果两个客户端发完全相同的请求（同 model、同 messages、`temperature=0`），你能直接返回缓存吗？有什么风险？
6. 你的网关现在有"状态"吗？重启会丢什么？（这个问题的答案决定了 Phase 3 要往 Redis 放什么）

**答完第 6 题，你就自然知道 Phase 3 该做什么了。**
