## 商业化落地与 AI 工程运维(重写版)

> 本阶段目标:把 AI 应用从 localhost 推到能服务真实付费用户的生产系统,具备抗压 / 弹性 / 容错 / 可观测 / 安全合规 / 成本可控,并建立"线上数据 → 评估 → 改进"的持续迭代闭环。

---

### 5-1 LLM Gateway 与多 Provider 路由【必学】

**学习重点:**

- LLM Gateway 主流产品:
    - **LiteLLM Proxy**(开源、自部署、最灵活)
    - **Portkey**(商业 + 开源)
    - **Helicone**
    - **Cloudflare AI Gateway**
    - **Kong AI Gateway**
- 核心能力:统一 OpenAI 兼容协议、多 Provider 路由、限流、缓存、日志、密钥代管、成本核算
- 路由策略:权重 / 延迟 / 成本 / 主备降级
- 在自研后端 vs 独立 Gateway 之间的取舍

**理解:**

- 为什么 2026 年标配 Gateway:统一接入、灰度、成本核算、Provider 切换零代码改动

---

### 5-2 容错、重试、限流、降级【必学】

**学习重点:**

- 重试:`tenacity`、指数退避 + 抖动、区分可重试/不可重试错误
- **Circuit Breaker**(断路器,`pybreaker`)
- 客户端限流:令牌桶(`aiolimiter`)、按用户 / 按 API key / 按租户分级
- **Provider 限流**:RPM / TPM 双维度
- 降级:主模型挂 → 兜底模型;贵模型超额 → 便宜模型
- 流式断线重连:`Last-Event-ID` + 服务端事件回放
- 请求队列与排队反馈(高峰期用户提示"前面还有 N 个请求")

**理解:**

- "重试 + 限流 + 降级" 是 AI 服务三件套,缺一个就有故障预案缺口
- 错误的重试会"成本爆炸"——失败也烧 token

---

### 5-3 缓存策略与成本优化【必学】

**学习重点:**

- 三层缓存:
    - **Provider Prompt Cache**(Anthropic `cache_control` / OpenAI 自动 / DeepSeek)
    - **应用层语义缓存**(`GPTCache` / `Redis` + embedding)
    - **HTTP / CDN 层**(对幂等 GET)
- Prompt 结构优化以命中 Cache(静态前 / 动态后)
- 模型选择经济学:输入贵 vs 输出贵差异、推理模型 reasoning_tokens 计费
- **Routing 到便宜模型**:简单查询走小模型(`RouteLLM` / 自训分类器)
- 批量 API(Batch API)适用场景(异步任务,价格半价)
- 输出长度控制 `max_tokens`、`stop_sequences`

**理解:**

- 一个能跑起来的 AI 应用 ≠ 一个能盈利的 AI 应用,成本设计在第一天就要做
- 缓存命中率每提升 10% 通常等于毛利率提升数个百分点

---

### 5-4 安全与合规【必学】

**学习重点:**

- **OWASP Top 10 for LLM Applications**(2025 版)逐项理解
- Prompt Injection 防御:输入分隔、指令隔离、输出校验、最小权限工具
- 间接注入(工具结果 / RAG 内容反向攻击)
- 输出安全:敏感词过滤、毒性检测(Llama Guard / ShieldGemma / 阿里安全 / 腾讯天御)
- **PII 检测与脱敏**(`Presidio`,身份证 / 手机号 / 银行卡)
- 输入输出审计日志(满足合规留存)
- API Key 管理:密钥轮换、最小权限、`HashiCorp Vault` / `AWS Secrets Manager` / `阿里云 KMS`
- 用户鉴权:JWT / OAuth 2.1 / API Key
- **国内合规**:生成式 AI 服务备案、内容审核接入(国内必备)
- **海外合规**:GDPR / EU AI Act / 数据出境
- **数据驻留 / 私有化部署**

**理解:**

- 任何"用户输入 → LLM → 工具执行"的链路都默认不安全,需层层防御
- 合规不是法务的事,是工程师的事

---

### 5-5 多租户、计费与配额【必学】

**学习重点:**

- 多租户隔离:数据隔离(行级权限 / schema 隔离 / 库隔离)
- 用户 / 租户级用量计量:token / 请求数 / 工具调用数
- 配额(Quota)与软硬限制
- 计费模式:按 token / 按请求 / 包月 / 积分制
- Stripe / Lemon Squeezy / 国内支付集成
- 成本归因:把单次 API 调用的成本归到用户 → 功能 → 团队
- 余额预扣 / 异步计费 / 防套利

**理解:**

- AI SaaS 的护城河之一是"知道每个用户花了多少 / 给你赚了多少"
- 计量必须前置设计,后补极难

---

### 5-6 依赖管理与容器化【必学】

**学习重点:**

- Python 依赖管理:`uv`(2025 主流,极快)/ `Poetry` / `pip-tools`
- `pyproject.toml` 与 lock 文件
- Dockerfile 多阶段构建、`slim` / `distroless` 基础镜像
- 镜像大小优化:模型权重不进镜像(运行时挂载或下载)
- `.dockerignore`、构建缓存层
- GPU 镜像基础:`nvidia/cuda` 基镜、`nvidia-container-toolkit`

**理解:**

- AI 服务镜像膨胀的真正原因:torch / cuda / 模型权重,需分层处理

---

### 5-7 编排与部署【必学】

**学习重点:**

- **本地 / 小规模**:`docker-compose`(FastAPI + Postgres + Redis + Qdrant + Langfuse)
- **PaaS 快速上线**:Railway / Render / Fly.io / 腾讯云 CloudBase / 阿里云 SAE
- **K8s 生产部署**:
    - Deployment / Service / Ingress 基础
    - **HPA**(水平伸缩)+ **KEDA**(基于 QPS / 队列长度伸缩)
    - **Ray Serve / KServe**(模型服务化)
    - **vLLM Production Stack**(2025 vLLM 官方 K8s 方案)
- GPU 调度:`nvidia-device-plugin`、节点选择、共享 GPU(MIG / MPS / vGPU)
- 蓝绿 / 金丝雀发布
- 配置中心:`Nacos` / `etcd` / K8s ConfigMap

**理解:**

- 个人项目用 PaaS,商业项目用 K8s,中间不要纠结
- LLM 服务的伸缩信号不是 CPU,而是队列深度 / TTFT

---

### 5-8 密钥与配置管理【必学】

**学习重点:**

- `.env` + `pydantic-settings` 本地开发
- 生产 Secrets:`Vault` / `AWS Secrets Manager` / `阿里云 KMS` / `Doppler` / `Infisical`
- 多环境配置分层:dev / staging / prod
- 密钥轮换流程
- 客户端 API Key vs 后端 Provider Key 严格分离
- Provider Key 通过 LLM Gateway 集中托管

---

### 5-9 可观测性(Logs / Metrics / Traces)【必学】

**学习重点:**

- **LLM 专属可观测平台**(必接其一):
    - **Langfuse**(开源 + 云,推荐)
    - **LangSmith**(LangChain 生态)
    - **Helicone** / **Arize Phoenix** / **Braintrust** / **W&B Weave**
- 结构化日志:`structlog` / `loguru`,trace_id 贯穿
- Metrics:Prometheus + Grafana
- Traces:OpenTelemetry → Tempo / Jaeger
- 错误告警:Sentry / 钉钉机器人 / 飞书机器人
- **LLM 专属指标**:
    - TTFT / TPOT / 端到端延迟
    - Token 用量(prompt / completion / cached / reasoning)
    - 单次请求成本
    - 工具调用成功率 / 重试次数
    - 用户反馈(👍/👎)、留痕评分
    - 幻觉率(基于 RAG faithfulness 抽样)
- 业务漏斗:发起 → 模型响应 → 工具完成 → 用户接受

**理解:**

- 传统 APM 不够用,必须有 LLM 维度的追踪
- "看 trace 改 prompt" 是 LLM 调优最快路径

---

### 5-10 线上评估与持续迭代【必学】

**学习重点:**

- **在线评估**:生产 trace 抽样 → LLM-as-Judge / 人工标注
- **离线评估集**:从生产 trace 中沉淀,持续扩充
- **A/B 测试**:模型版本、prompt 版本、检索策略
- **灰度发布**:按用户 / 租户 / 流量比例
- Feature Flag(`Unleash` / `Flagsmith` / `LaunchDarkly`)
- 用户反馈回路:👍/👎、修改回写、隐式信号(用户是否复制结果)
- **数据飞轮**:用户使用 → trace → 评估集 → 改进 → 上线 → 再使用

**理解:**

- AI 产品的核心壁垒不在模型,在"你比对手更快建立改进闭环"
- 没有评估集就没有改进方向,只有玄学

---

### 5-11 性能压测与容量规划【必学】

**学习重点:**

- 压测工具:`locust` / `k6` / `vllm bench` / `genai-perf`
- LLM 压测的特殊性:输入长度 / 输出长度 / 思考 token 分布
- 容量规划公式:`并发上限 ≈ Provider RPM / 平均请求时长` 或自部署的 `GPU 数 × 单卡吞吐`
- 排队 vs 拒绝策略
- 冷启动问题(自部署模型)与预热

---

### 阶段实战项目

**《商用级智能客服平台 SaaS 化部署》**

- 多租户架构,行级数据隔离
- LiteLLM Proxy 做 LLM Gateway,主备降级 + 语义缓存
- 知识库 RAG(承接阶段三)+ 退换货状态机 Agent(承接阶段四)
- 用 `uv` 管理依赖,多阶段 Dockerfile
- K8s 部署(HPA + KEDA 基于队列伸缩),GPU 节点跑自部署 embedding / rerank
- Vault 管理密钥,JWT 鉴权
- Langfuse 全链路追踪,接入用户👍/👎反馈
- Stripe 计费,按 token 用量出账
- OWASP LLM Top 10 安全自查,接入国内内容审核
- 建立离线评估集 + 在线 A/B,验证两种 prompt 哪个用户满意度更高
- 全套用 `k6` 压测,出容量规划报告