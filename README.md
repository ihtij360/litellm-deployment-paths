# litellm 部署走哪条路：单机 Docker、Helm on Kubernetes、Terraform 各要准备什么？Postgres、密钥与扩缩容的坑一次讲清

部署 LiteLLM 有三条路：单机 Docker 最快，Helm on Kubernetes 是官方文档里的生产路径，AWS 和 GCP 上还能用 Terraform 直接拉起整套栈。Azure 没有 Terraform 模块，只能走 AKS 加 Helm。

真正要准备的往往不是 LiteLLM 本身。它是无状态容器，删掉重建不丢数据；麻烦在它背后的 Postgres、Redis，以及两个一旦设错就要重装的密钥。下面按“要多快跑起来”和“要不要对外服务”分开讲，最后给三个多数教程不提的故障。

> **多数人的选择**
>
> 只在本机验证虚拟密钥、预算和花费统计：单机 Docker 加一个本机 Postgres，半小时能通。
>
> 要给团队或自己的应用长期提供 OpenAI 兼容接口：Kubernetes 加 Helm，Postgres 和 Redis 用托管服务，网关至少 2 个副本。
>
> 已经在 AWS 或 GCP 上且不想维护 K8s：用官方 Terraform 模块。
>
> 机器侧，网关本身不吃 CPU，吃的是内存和数据库连接。2 核 2G 能跑起来，但 Postgres 的花费日志会和容器抢内存，4 核 4G 是更稳的起点。
>
> 👉 [查看 SixtyNet 当前在售 VPS 的配置与年付价](https://sixtynet.net/aff.php?aff=39)

## SixtyNet 在售方案与 LiteLLM 的匹配度

先把机器这件事说清楚。SixtyNet 目前在洛杉矶机房提供几条产品线，其中 LAX 系列带 DDOS 防护，价格按年付展示，官网标明比原价低 17%。下面是官网当前展示的四档：

| 方案 | vCPU / 内存 | NVMe | 月流量 | 年付价格 | 购买 |
| --- | --- | --- | --- | --- | --- |
| LAX-STARTER | 2 核 / 2G | 40G | 5TB | $200（原价 $240） | [选这款](https://sixtynet.net/aff.php?aff=39) |
| LAX-MICRO | 4 核 / 4G | 80G | 10TB | $400（原价 $480） | [选这款](https://sixtynet.net/aff.php?aff=39) |
| LAX-MEDIUM | 8 核 / 8G | 160G | 20TB | $800（原价 $960） | [选这款](https://sixtynet.net/aff.php?aff=39) |
| LAX-LARGE | 8 核 / 16G | 320G | 40TB | $1,600（原价 $1,920） | [选这款](https://sixtynet.net/aff.php?aff=39) |

官网还有两条线没放进这张表，因为报价按配置分档展示，而且限制会直接影响部署方式：一条是住宅 IP 系列，另一条是洛杉矶中国联通 Premium，走 AS9929、支持免费建立 BGP 会话、按 95 计费，适合需要优质回国线路的场景。

住宅 IP 系列有个必须先知道的限制：**官方页面明确写了，出于 IP 风控原因，住宅 IP 产品不开放 80 和 443 端口**。LiteLLM 网关默认监听 4000，本身不占用 80/443，但你如果要给它套域名、上 HTTPS、走标准端口的反向代理，这条路就堵死了。可行做法是用非标准端口直连，或者用 Cloudflare Tunnel 之类的隧道把流量导进去。如果你的目标是让公司内部或客户的 SDK 稳定指向一个 `https://llm.xxx.com/v1`，选 LAX 系列而不是住宅 IP 系列。

流量方面不用太焦虑。网关转发的是文本 token，不做图片、音频转发的话，5TB 月流量对绝大多数团队都是过剩的。

👉 [对比 SixtyNet 各条产品线的端口与线路限制](https://sixtynet.net/aff.php?aff=39)

## 三条部署路径，各自要准备什么

### 单机 Docker：最快，但 Postgres 要自己补

LiteLLM 官方镜像发布在 `ghcr.io/berriai`，并在 `docker.litellm.ai/berriai` 有镜像。单机跑要用 `ghcr.io/berriai/litellm` 这个仓库，因为它打包了 Prisma 工具链，带 Postgres 的部署缺了它跑不起来。

启动时把配置文件挂进容器、用 `-e` 传 `LITELLM_MASTER_KEY` 和 `DATABASE_URL`、把 4000 端口映射出来，容器就能提供 OpenAI 兼容接口。tag 一定要写具体版本号，别用 `latest`——官方文档专门强调过这一点，浮动 tag 会让回滚变成猜谜。

这条路的短板是它只有一层：没有副本、没有优雅退出、Postgres 挂了网关跟着挂。适合本地验证、CI 里跑集成测试、或者小团队内部自用。

### Helm on Kubernetes：官方生产路径

Helm 路径有两种架构。**单体**用一个 `litellm` 镜像同时提供 LLM 流量、管理 API 和 UI，也就是 `litellm-helm` chart 的跑法，运维最简单。**微服务**拆成 gateway（LLM 流量，4000 端口）、backend（管理 API 与 UI 后端，4001）、ui（3000）三个组件，各自独立扩缩容，对应组件化 chart 和两个 Terraform 模块。

不论哪种，支撑设施是一样的：网关无状态，负载均衡后面跑 2 个以上副本；Postgres 存密钥、团队、用户、花费日志和配置；Redis 负责限流、路由状态和跨实例缓存，只要副本超过一个就必须有；migrations job 每次升级跑一次，对 Postgres 执行 schema 迁移，而 proxy 实例统一设 `DISABLE_SCHEMA_UPDATE=true`，永远不自己跑迁移。

集群外的东西也要提前备：EKS 上配 AWS Load Balancer Controller，GKE 用原生 Ingress，AKS 用 AGIC，健康检查打 `/health/readiness`，然后把 DNS 指到生成的负载均衡器上。ingress 有个细节容易卡住人，ingress-nginx 的 admission webhook 会拒绝 chart 里带点的路径（`/favicon.ico`、`/eu.assemblyai`），除非把 `controller` 显式设成 `nginx`。

### Terraform：AWS 和 GCP 有，Azure 没有

AWS 和 GCP 各有官方 Terraform 模块，能把整套栈一次拉起来。Azure 没有对应模块，官方给的路径就是 AKS 加 Helm，自己在 Terraform 里包一层也可以，但那是自己维护的封装。

用托管数据库还有个分支：如果数据库不用密码而用短期 token（RDS/Aurora 的 IAM 数据库认证、Azure PostgreSQL 的 Entra ID），要设 `db.useIAMAuth: true` 或 `db.useAzureEntraAuth: true`，去掉 `secret` 块，Pod 还需要 IRSA、Pod Identity 或 workload identity 之类的云凭证。Azure Cache for Redis 走 TLS 时端口是 6380，`router_settings` 里要用 `redis_url: "rediss://..."` 而不是 `redis_host`/`redis_port`，`rediss://` 前缀本身就代表开 TLS。

## 部署前必须准备好的四样东西

第一样是 Postgres。它不只是个日志库，虚拟密钥、团队、用户、预算、Admin UI、MCP 与 agent registry 的数据都在这儿，proxy 的鉴权和花费追踪离了它直接不可用。

第二样是 Redis。单副本跑可以不配，但只要你想扩到两个副本，限流和路由状态就需要一个共享存储，否则每个副本各算各的。

第三样是 `LITELLM_MASTER_KEY`，网关的根凭证，必须以 `sk-` 开头。用 `openssl rand -hex 16` 之类的方式生成，别用能猜的字符串。它还兼任管理后台密码。

第四样是 `LITELLM_SALT_KEY`，加密存在数据库里的提供商凭证。这个密钥**设一次之后永远不要改**。改了以后数据库里那些 API key 全部不可读，只能一条条重新录入。官方建议把这两个 key 都放进云厂商的 secret manager，不要躺在 values.yaml 里。

## 版本和镜像：chart 版本和应用版本是两件事

这是多数部署教程没讲清的一点。Helm 部署里有两个独立的版本号：chart 版本决定渲染出什么样的 Kubernetes manifest，`image.tag` 决定实际跑哪个 LiteLLM 二进制。它们可以分开固定。

官方文档说已发布的 chart 版本会携带 LiteLLM release 号（比如 `1.90.2`），组件化 chart 要求固定到 `1.89.0` 或更新。社区实操里出现过另一种写法——固定 `oci://ghcr.io/berriai/litellm-helm --version 0.1.100`，同时让应用版本浮动到最新。两种描述可能对应的是不同 chart，但背后的逻辑一致：manifest 的变更风险比应用版本大，所以 chart 该钉死；应用版本只影响跑哪个二进制，可以跟进。

顺带一提，chart 默认会把 image tag 设成 `latest`，GitHub 上 LiteLLM 的 release candidate 又被标为 pre-release、不会进 `releases/latest`，所以“最新稳定版”需要你自己解析，别指望默认值。

`helm show values oci://ghcr.io/berriai/litellm-helm` 能列出所有可配置项，比翻文档快。

## 三个容易踩的坑

**Bitnami 的镜像 tag 被删了。** 2025 年 Bitnami 把所有带版本号的 tag 迁到了只读的 `bitnamilegacy` 命名空间，而 LiteLLM 的 chart 里有个 init container 硬编码引用了 `docker.io/bitnami/postgresql:16.1.0-debian-11-r20`，这个 tag 在 Docker Hub 上已经不存在，且写死在模板里，任何 Helm value 都改不到。直接 `helm install` 会看到 Postgres Pod 卡在 `ImagePullBackOff`。绕过办法是在节点侧把 legacy 镜像拉下来重打成 chart 期望的名字，靠节点的 `IfNotPresent` 拉取策略生效。主 Postgres 镜像有同样的问题，但那个可以通过 values 覆盖。

**`helm upgrade` 不带 `--reuse-values` 会换掉 master key。** 不显式传 `--set masterkey=...` 时 chart 会重新生成一个随机密钥，升级完所有还在用旧 key 的客户端立刻 401，而且报错信息不会告诉你原因。更麻烦的是，如果提供商凭证是通过 UI 保存的、安装时又没设独立 salt key，这些凭证是用 master key 加密的，所以轮换必须发生在添加凭证之前，不能之后。

**扩副本的上限其实是 Postgres 连接数。** chart 里 `maxReplicas` 默认给到 100，看着很自由，但连接池是按 worker 算的，默认池上限 10 的情况下，全量扩容意味着约 1000 个 Postgres 连接。数据库先撑不住，Pod 才刚开始扩。

## 扩缩容：CPU 指标对 LLM 流量不灵

一个 Pod 同时在流式返回四十个响应时，大部分时间在等上游提供商，CPU 很低，但它的并发容量已经用完了。按 CPU 扩缩容在这种场景下会一直滞后。

两个 chart 都支持按 proxy 已经导出的两个计数器扩：`litellm_proxy_total_requests_metric_total` 和 `litellm_total_tokens_metric_total`，折算成每 Pod 的 RPS 和 TPS。算例很直观：10 个网关 Pod 合计 1000 rps，即每 Pod 100 rps，目标设 90，HPA 会要 `ceil(10 × 100 / 90) = 12` 个副本。

走 HPA 这条路需要两件前置：chart 的 ServiceMonitor 逐 Pod 抓取，样本才带 `pod` 标签；以及开启独立的 metrics listener，因为主端口的 `/metrics/` 在虚拟密钥鉴权后面，未认证抓取只会拿到 401。然后用 Prometheus Adapter 在 `custom.metrics.k8s.io` 上按 Pod 提供这两个指标名。`rate()` 返回的已经是每秒值，不需要再乘 60。

不想装 adapter 的话，`litellm-helm` 上可以用 KEDA，`keda.prometheus.requestsPerSecond` 和 `keda.prometheus.tokensPerSecond` 填单副本应承载的负载，效果和 HPA 路径一致。两种机制互斥，同时开会只渲染 HPA。

TPS 有个盲区值得记一笔：token 是在响应结束时才计数的，几分钟的长流在跑的过程中不会出现在 TPS 里，RPS 先反应，TPS 后追上。如果你们的流量以长流式为主，别只设 TPS 目标，RPS 目标要一起配上。

## 怎么选

只想在本机或一台测试机上把网关跑起来，验证虚拟密钥、预算和花费统计长什么样：单机 Docker 加本机 Postgres，镜像 tag 写死版本号，别用 `latest`。

要给团队或外部应用提供长期接口：Kubernetes 加 Helm，Postgres 和 Redis 用托管服务，网关至少 2 个副本，还要把扩缩容的上限和数据库连接数对齐。

已经在 AWS 或 GCP 上、又不想自己维护 K8s：直接上官方 Terraform 模块。在 Azure 上就只有 AKS 加 Helm 一条路。

下单机器之前确认两件事：一，你要不要用标准 80/443 端口对外提供 HTTPS，要的话避开住宅 IP 系列；二，你打算跑几个副本，副本数乘以每 worker 的连接池大小，就是 Postgres 需要承受的连接数，单机 all-in-one 的部署方式扩不了副本，这一点在买机器时就该想清楚。

👉 [按你的部署方式挑一台 SixtyNet 的洛杉矶 VPS](https://sixtynet.net/aff.php?aff=39)
