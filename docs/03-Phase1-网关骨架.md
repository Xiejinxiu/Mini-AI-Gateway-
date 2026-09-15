# 03 · Phase 1：网关骨架（M1 + M2）

> **这一阶段完全不碰 AI。** 目标只有一个：让一个 HTTP 请求真的从你的客户端，穿过你的网关，落到一个后端，再回来。
>
> 这一阶段结束后，你会开始主动想学「计算机网络」——因为你亲眼看见了那些字段。

---

## 一、这一阶段为什么这么排

大多数人学网关是「打开 Spring Cloud Gateway，抄配置」。那样你学到的只有配置语法。

我们的路子是：**先用手写一个会漏 header 的破网关，再一条条把它补成生产级。** 每条补丁对应一个 HTTP 知识点。等你补完，HTTP 就不再是"背诵题"了。

```text
Client ──── HTTP ────▶ Gateway (:8080) ──── HTTP ────▶ mock-backend (:8081)
                            │
                     你手写的转发逻辑
                   （JDK HttpClient，不用框架）
```

---

# M1：最小转发（1 天）

## M1-1 先做后端（约 40 分钟）

**为什么先做后端**：你要转发的目标得先存在。而且亲手写一次后端，你才知道 `HttpServletRequest` 里能拿到什么——这个视角在写网关时极其值钱。

在 `mock-backend` 里写一个 `EchoController`，暴露 `GET/POST /echo`，返回一个 JSON：

```json
{
  "method": "POST",
  "path": "/echo",
  "query": "a=1&b=2",
  "headers": { "content-type": "...", "x-request-id": "...", "host": "..." },
  "body": "原样回显的请求体",
  "receivedAt": "2025-01-01T12:00:00"
}
```

要求（这些要求就是后面网关的验收依据）：
- 把**收到的所有 header** 原样列出来（不要只挑几个）——后面你要靠它验证透传
- `body` 原样回显，别做 JSON 解析（Phase 2 再解析）
- 打一行日志：`收到请求 method=... path=... headerCount=...`

**验收**：

```powershell
curl.exe -v "http://localhost:8081/echo?a=1&b=2" -H "X-Test: hello"
# 期望：返回的 headers 里能看到 x-test: hello
# 期望：-v 输出里能看到 < HTTP/1.1 200 和完整响应头
```

> 💡 **用 `curl.exe`，不要在 PowerShell 里用 `curl`**——PowerShell 里 `curl` 是 `Invoke-WebRequest` 的别名，参数不一样，会浪费你半小时。这个坑现在告诉你，省你一次暴躁。

**M1-1 要理解的**：
- `@RestController` / `@GetMapping` 背后 Spring MVC 帮你做了什么
- 请求头是**大小写不敏感**的（HTTP/1.1 规范），但 Map 里的 Key 可能是小写，为什么
- `-v` 的输出里，`>` 开头是**我发出去的**，`<` 开头是**我收到的**——记住这个，后面全靠它

---

## M1-2 网关的最简转发（约 90 分钟）

在 `gateway` 里：

1. 写一个 `ProxyController`，映射 `@RequestMapping("/api/**")`
2. 用 **JDK 的 `java.net.http.HttpClient`**（不是 RestTemplate，不是 WebClient）把请求转发到 `http://localhost:8081`
3. 把上游的响应原样写回给客户端

**先把"最傻的版本"写出来，允许它漏掉一半东西**：

```java
// 伪代码，别照抄，把这个意思用自己的代码实现
@GetMapping("/api/echo")
public ResponseEntity<String> proxy() {
    HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create("http://localhost:8081/echo"))
            .GET()
            .build();
    HttpResponse<String> resp = httpClient.send(req, BodyHandlers.ofString());
    return ResponseEntity.status(resp.statusCode()).body(resp.body());
}
```

跑通：

```powershell
curl.exe "http://localhost:8080/api/echo?a=1&b=2"
# 期望：看到 backend 的 JSON
```

**跑通了就是胜利。** 现在你亲眼看到：请求从 8080 进来，从 8081 出去又回来了。

---

## M1-3 然后自己找茬（约 50 分钟，这是最重要的一步）

现在拿出刚才的 echo JSON，**逐条对照**，自己列出问题清单。你会发现至少这些不对：

1. `query` 里的 `a=1&b=2` 丢了（我只转发了路径）
2. `headers` 里没有 `x-test`（我一个 header 都没传）
3. 我的 `POST` 请求进来，网关却用 `GET` 转出去了
4. `method` 是 `GET`，method 完全没透传
5. 请求体（body）根本没读、没转
6. 我写死了 `localhost:8081`，以后怎么配置

**这就是 M2 的任务清单——而且是你自己发现的，不是我给的。**
发现问题的能力，比解决问题的能力值钱。把你的清单写进 `notes/01-http与网络.md`。

---

# M2：生产级转发（2 天）

## 任务卡

| 编号 | 任务 | 敲什么 | 验收命令 | 知识点 |
|---|---|---|---|---|
| **M2-1** | method 透传 | 从 `HttpServletRequest` 取 method，动态构造 `HttpRequest` | `curl -X POST .../api/echo` 返回 `"method":"POST"` | HTTP 方法语义、幂等性 |
| **M2-2** | path + query 透传 | `getRequestURI()` + `getQueryString()`，注意中文/特殊字符编码 | `curl ".../api/echo?name=%E5%BC%A0%E4%B8%89"` 能正确回显 | URL 编码、`%` 转义、`URI` 构造 |
| **M2-3** | header 透传（**含黑名单**） | 遍历 `getHeaderNames()`，过滤 hop-by-hop 头后转发 | echo 里能看到你自定义的头，但看不到 `Connection` | **hop-by-hop vs end-to-end 头** |
| **M2-4** | body 透传 | `getInputStream().readAllBytes()` 转字节数组 | `curl -X POST -d '{"a":1}' -H 'Content-Type: application/json'` 原样回显 | Content-Length、字节 vs 字符、编码 |
| **M2-5** | status + 响应头透传 | 上游状态码和响应头回写（同样过滤黑名单） | 让 backend 返回 418，网关也返回 418 | 状态码语义（1xx~5xx） |
| **M2-6** | 可配置上游地址 | 抽到 `application.yml`，用 `@ConfigurationProperties` 读 | 改 yml 后不用改代码 | 配置外部化、`@ConfigurationProperties` |
| **M2-7** | 三种超时 | 设置 connectTimeout / 请求超时，超时转 504 | 见下面「故障实验」 | **超时的三个层次** |
| **M2-8** | 错误处理 | 后端挂了 → 502；超时 → 504；返回统一错误 JSON | 关掉 backend 再 curl，拿到 502 | 网关错误 vs 上游错误、错误码归属 |
| **M2-9** | 请求 ID + 结构化日志 | 生成/透传 `X-Request-Id`，前后端日志都带上 | 一次 curl，两端日志能串起来 | 链路追踪思想（Phase 3 的伏笔） |

---

## 关键讲解点（我讲，你敲）

### ① hop-by-hop 头为什么不能透传（M2-3）

HTTP 头分两类：

- **end-to-end**：`Content-Type`、`Authorization`、`Accept`…… 描述的是"最终接收方"的事，**必须透传**
- **hop-by-hop**：`Connection`、`Keep-Alive`、`TE`、`Trailer`、`Transfer-Encoding`、`Upgrade`、`Proxy-Authenticate`、`Proxy-Authorization`…… 描述的是"**这一跳**"的连接，**绝不能透传**

原因：`Connection: close` 意思是"我这条 TCP 连接用完就关"，只对客户端↔网关这一跳有意义。你把它转给后端，等于命令后端关连接，链路语义就被污染了。

**这个坑几乎所有手写网关的人都会踩，而且症状极其诡异**（连接复用异常、偶发挂起）。踩一次，你就永远记住了。

### ② 超时的三个层次（M2-7）

| 类型 | 含义 | 对应故障 |
|---|---|---|
| **连接超时** | TCP 三次握手没成功 | 后端没启动、端口不通、防火墙 |
| **读超时 / 响应超时** | 请求发出去了，对方迟迟不返回 | 后端卡住、死锁、GC 停顿 |
| **总超时** | 整个请求端到端的最长时限 | 组合场景，是**客户端**感知到的那个 |

**网关和客户端都得设超时**，且 `网关总超时 < 客户端总超时`，否则客户端先超时放弃，网关还在傻等——这是真实生产事故的常见形态。

### ③ 502 / 504 该由谁返回（M2-8）

- 上游返回 500 → **透传 500**（错误是上游的，网关只负责转达，不能改）
- 连不上上游 → 网关返回 **502 Bad Gateway**
- 上游超时 → 网关返回 **504 Gateway Timeout**

**为什么不能统一成 500**：调用方需要区分"我的请求有问题"、"服务挂了"、"太慢了"，这三种的处置策略完全不同（重试？降级？报警？）。**错误码是接口契约的一部分。**

---

## 故障实验（必须做，每个 5 分钟）

**故意搞坏它，看它怎么坏。** 这比让它正常工作学到的多。

| # | 怎么搞 | 期望现象 | 你要解释 |
|---|---|---|---|
| E1 | 关掉 `mock-backend`，curl 网关 | 502 + 你的错误 JSON | 异常从哪一层抛出来的？ |
| E2 | 在 backend 的 echo 里 `Thread.sleep(5000)`，网关超时设 2s | 504 | 是网关超时还是 HttpClient 超时？ |
| E3 | 发一个 `Content-Type: application/json; charset=iso-8859-1` 的请求 | 观察回显的字节 | 中文会乱码吗？为什么？ |
| E4 | 让 backend 返回一个 10MB 的 body | 正常返回还是 OOM？ | `ofString()` 意味着什么？ |
| E5 | 用 PowerShell 并发打 100 个请求 | 全部成功？耗时？ | 网关线程数、上游连接池 |
| E6 | Wireshark 抓 `lo` 上的流量，curl 一次 | 看到三次握手 | **这一步别跳过**，你会第一次"看见"TCP |
| E7 | 连续 curl 两次，看 TCP 连接是否复用 | 第一个请求后连接没断 | keep-alive 是谁在维持？ |

> E6 建议用 **Wireshark + Npcap**（loopback 抓包）。看到 SYN / SYN-ACK / ACK 的那一瞬间，你对"网络"的感觉会永久改变。这就是《简介.md》里说的"你自己产生了问题"。

---

## M2 验收清单

- [ ] `curl -X POST -d '{"a":1}' .../api/echo?x=中文` → method/path/query/body 全对
- [ ] 自定义 header 能到后端；`Connection`、`Host` 等**没**被透传
- [ ] backend 返回 418/500 时，网关原样透传状态码
- [ ] backend 关掉 → 502；超时 → 504；错误响应体是统一的 JSON 格式
- [ ] 上游地址可在 `application.yml` 改，不用改代码
- [ ] 一次请求，网关和 backend 的日志能用同一个 `X-Request-Id` 串起来
- [ ] E1~E7 全部跑过，每个都在 `notes/` 里写了一句结论
- [ ] commit：`feat(M2): 完成生产级 HTTP 转发（method/path/header/body/status/超时/错误）`

---

## 常见坑（先看一遍，能省你 2 小时）

1. **`curl` vs `curl.exe`**（PowerShell 别名坑）
2. **`getInputStream()` 只能读一次**——读两次拿到的第二次是空的
3. **`Content-Length` 不要手动设置**，让 HttpClient 自己算；手动设错会挂起
4. **`Transfer-Encoding: chunked` 和 `Content-Length` 互斥**，两个都在会 400
5. **中文 query 不编码**直接塞进 `URI.create()` 会抛异常 → 用 `URLEncoder`（注意空格是 `%20` 不是 `+`）
6. **代理/本机环境变量**：`HTTP_PROXY` 会影响 `HttpClient`，本机调试时可能莫名失败
7. **IDEA 改完 yml 要重启**（除非开了 devtools）
8. **`HttpClient` 是线程安全的、应该复用一个实例**，别每次请求 new 一个——这是个很好的思考题：为什么？

---

## M1+M2 复盘题（先自己答，再来找我）

1. 画一张时序图：一个 `POST /api/echo?x=1` 从 curl 到 backend 再回来，中间发生了什么？（要标出两次 TCP 连接）
2. 网关在这里是"代理"还是"中间人"？它对客户端来说像什么？
3. 如果网关和 backend 之间是 `Connection: close`，QPS 会受什么影响？为什么？
4. 现在这个网关，能同时处理两个请求吗？瓶颈在哪？
5. 为什么说"网关不应该理解业务"？

答完这 5 题，你已经具备做 Phase 2 的资格了。
