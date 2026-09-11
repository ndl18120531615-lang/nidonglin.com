---
title: AI 智能书店（一）：一个能自己动手帮我买书的网上书店
published: 2026-08-11
description: 一个把大模型接进真实交易链路的图书电商：Function Calling 让 AI 直接操作购物车与图书、Elasticsearch 向量检索做语义搜索与个性化推荐、RocketMQ 事务消息保证订单与库存一致、支付宝沙箱完成从下单到退款的支付闭环。
image: ''
tags: [Spring AI, Spring Boot, RocketMQ, Elasticsearch, 支付宝, 项目实战]
category: 技术
draft: false
lang: ''
---

## 为什么写这个项目

上一篇写的是智能客服（[Spring AI 实战：用 DeepSeek 搭一个会查天气、能记上下文的智能客服](/posts/spring-ai-smart-customer-service/)），那是一个纯粹的 AI 应用——所有能力都长在对话框上。

这次想换个方向试试：**如果 AI 不是主角，而是一条真实交易链路里的一个零件，会长什么样？**

于是有了这个项目：一个完整的图书电商。注册登录、浏览搜索、购物车、下单、支付宝支付，后面挂着一整套库存与订单状态的自动流转。而 AI 不是贴在右下角的浮窗——它能真的查我的购物车、能把书加进去、能用语义而不是关键词帮我找书。

<!-- TODO 配图 ai-bs-01-home.png：首页整屏（分类导航 + 轮播 + 新鲜好书 + 低价专区）
![首页：分类导航、轮播和图书楼层](./images/ai-bs-01-home.png)
-->

## 一句话概括

> 注册登录 → 浏览 / 分类 / 搜索 → 加入购物车 ↓ 确认订单与地址 → 支付宝支付 ↓ 后台自动完成：扣库存 / 改订单状态 / 清购物车 / 退款。
> 在此之上叠一层 AI：**能操作业务**（Function Calling）、**能理解语义**（ES 向量检索）、**能个性化推荐**（读购物车做检索）。

## 这个项目我觉得最值钱的七个地方

### 1. AI 从"会说话"变成"会干活"

绝大多数"AI + 电商"的做法是：页面右下角挂个对话框，问题发给模型，模型回一段话——它不知道你购物车里有什么，也改不了任何数据。

这个项目把**身份查询、购物车增删改查、日期、全量图书**四组后端能力注册成了工具：

```java
@Bean("openAiChatClient")
public ChatClient openAiChatClient() {
    return ChatClient.builder(openAiChatModel)
            .defaultTools(whoToolService, shoppingCatToolService, dateToolService, bookToolService)
            .build();
}
```

用户说"帮我把《活着》加两本"，模型不是回答"好的"，而是自己走完这条链：

```
findAllBook()                      → 拿全量图书，定位"活着"的 bookId
findShoppingCartByUserId(userId)   → 判断这本是否已在购物车
updateShoppingCartByUserId(...)    → 或 addShoppingCartByUserId(...)
                                   → Redis 里的购物车真的被改了
```

**这是我认为整个项目里最有含金量的一块**：模型全程没有数据库权限，它只是"申请调用"，真正执行的是我的 Java 代码——权限、事务、日志都还在自己手里。

### 2. 语义搜索：不看关键词，看"感觉"

用户想找的是「适合通勤路上看的、写小人物挣扎的小说」。这种需求 `LIKE '%通勤%'` 一条都搜不出来。

图书入库时被嵌入模型转成 4096 维向量存进 ES，搜索时把问题也转向量、按余弦相似度取 TopK：

```java
SearchRequest searchRequest = SearchRequest.builder()
        .query(question)            // 问题文本，内部自动转向量
        .topK(10)                   // 最相似的 10 条
        .similarityThreshold(0.4)   // 低于阈值的直接丢，避免污染上下文
        .build();
List<Document> documents = vectorStore.similaritySearch(searchRequest);
```

而且同一件事我写了三种实现对照着看：`search1` 靠模型读全量数据 + 手写正则洗 JSON；`search2` 是向量检索 + 结构化输出；`search3` 干脆不做二次生成，命中即反序列化。**三种写法的取舍我都踩过一遍**，这比只会抄一种写法扎实得多。

### 3. 个性化推荐：把购物车喂给向量库

`fenxi3` 是我最满意的接口，一条链把「用户身份 → 购物车 → 向量检索 → 生成推荐」串了起来：

```java
User user = whoToolService.who(token);                              // 1. 我是谁
List<ShoppingCart> carts = shoppingCatToolService.findShoppingCartByUserId(user.getId());
List<String> ids = carts.stream().map(c -> c.getBook().getId()).toList();
String q = "请根据图书ID为:" + ids + "，获取和它们风格相似的图书";      // 2. 用购物车构造查询
List<Document> docs = vectorStore.similaritySearch(SearchRequest.builder()
        .query(q).topK(5).similarityThreshold(0.4).build());
if (docs == null || docs.isEmpty()) {                                // 3. 检索为空要能降级
    return openAiChatClient.prompt().user("推荐五本图书").stream().content();
}
```

第 3 步的**降级**是我特意加的：向量库没结果时不能让接口 500，退化成"让模型随便推荐五本"至少还能用。**能被演示的 AI 功能，都是把异常路径想过的功能。**

### 4. 一条真正走得通的支付闭环

很多项目的"支付"只做到"能跳到支付宝"。这里是完整的一段：

```
下单 → 落库订单 + 订单明细 → 发半事务消息 → 返回支付宝收银台 HTML
     → 用户付款 → 同步跳回 /alipay/success（主动查一次交易状态，不盲信 URL 参数）
                → 异步通知 /alipay/notify（RSA2 验签，验签不过直接拒）
     → 事务回查发现已支付 → 发 order:success → 改订单状态 / 扣库存 / 清购物车
     → 库存不足时发 order:refund → 撤单 + 调支付宝退款
```

几个细节是我认为做对了的：

- **同步回跳页面不信任 URL 参数**：`return_url` 上的 `out_trade_no`、金额用户都能改，所以我的 `/alipay/success` 拿到订单号后**再主动查一次**支付宝的交易状态，按查询结果渲染页面；
- **异步通知必须验签**，否则任何人都能伪造一条"支付成功"的 POST 过来；
- **支付的每一步都有兜底**：查询失败、未支付、库存不足，分别有明确的状态流转，而不是抛异常了事。

### 5. 用事务消息解决"订单落库"和"扣库存"的一致性

订单落库和通知别人扣库存是两件事：先落库再发消息，消息可能丢；先发消息再落库，消费者可能处理一张不存在的订单。普通消息保证不了这个原子性，所以用了 **RocketMQ 半事务消息 + 本地事务回查**。

这里有一个我自己挺得意的设计：**本地事务成功的判定条件不是"数据库写成功了"，而是"用户已经付款了"**——因为支付是外部异步系统，用 `alipay.trade.query` 的结果作为提交/回滚的依据：

| 查询结果 | 返回给 Broker | 后果 |
| --- | --- | --- |
| 未支付 | `UNKNOW` | Broker 稍后回查，等到付款为止 |
| 已支付 + 库存充足 | `COMMIT_MESSAGE` | 消息可见 → 三个消费者各自干活 |
| 已支付 + 库存不足 | `ROLLBACK_MESSAGE` | 消息丢弃，并另发 `order:refund` 撤单退款 |

后续动作全部拆成独立消费者：改订单状态、扣库存、清购物车（订阅 `order:success`），撤单、退款（订阅 `order:refund`），用广播模式各收各的，互不干扰。

### 6. 全真环境，没有一处是"假"的

这一点我觉得比功能列表更能说明问题——项目不是用内存队列、H2 数据库糊出来的演示：

| 组件 | 实际用的东西 |
| --- | --- |
| 消息队列 | **RocketMQ 5.3.1**（proxy + 本地 broker 模式），真的走半事务消息与回查 |
| 向量库 | **Elasticsearch** 真实索引 `bookstore`，cosine 相似度，4096 维 |
| 缓存 | **Redis**，登录 token + 购物车 Hash |
| 数据库 | **MySQL 8** + Druid 连接池 + MyBatis-Plus 逻辑删除 |
| 模型 | 硅基流动 GLM-4.7（OpenAI 兼容协议）+ `Qwen3-Embedding-8B` |
| 支付 | 支付宝**沙箱网关**，真实的 RSA2 签名、交易查询、退款接口 |

也正因为全真，中间件的坑一个都躲不过：broker 注册地址写错一个字节，客户端就是 `No route info of this topic`；Redis 的快照落盘一失败，所有写命令直接被拒、登录接口 500。**能把这些环境问题一个个定位掉，本身就是这套项目的一部分收获。**

### 7. 细节上做了几处"防坑"设计

- **JWT + Redis 双校验**：JWT 负责"这张票是不是我发的、有没有被改过"，Redis 负责"这张票现在还作不作数"——所以登出、改密码、踢下线都能立刻生效，这是纯 JWT 做不到的；
- **订单号用雪花 ID 并且存成 String**：趋势递增、分布式唯一，转成字符串是为了避开 JS 的 `Number.MAX_SAFE_INTEGER`，否则前端拿到的订单号最后几位会变；
- **购物车用 Redis Hash**：`key = shopping_cart:{userId}`、`field = bookId`、`value = 数量`，加减是 `HINCRBY`、读全部是 `HGETALL`，天然贴合"商品 → 数量"的映射；
- **全局逻辑删除**：`is_deleted` 一配，所有查询自动带 `WHERE is_deleted = 0`，数据可恢复；
- **统一响应体**：`JsonResult{data, success, message, code}` 贯穿所有接口，前端只解析一种结构；
- **接口文档**：springdoc-openapi，所有接口在 `swagger-ui.html` 里可直接点着调，联调时省掉大量来回。

## 功能一览

### 交易链路

<!-- TODO 配图 ai-bs-02-login.png：登录页
![登录页](./images/ai-bs-02-login.png)
-->

<!-- TODO 配图 ai-bs-03-list.png：分类浏览或书名搜索结果列表
![分类与搜索列表](./images/ai-bs-03-list.png)
-->

<!-- TODO 配图 ai-bs-04-cart.png：购物车页面（数量增减、批量删除）
![购物车](./images/ai-bs-04-cart.png)
-->

<!-- TODO 配图 ai-bs-05-order.png：订单确认页 + 地址管理弹层
![订单确认与收货地址](./images/ai-bs-05-order.png)
-->

<!-- TODO 配图 ai-bs-06-alipay.png：支付宝沙箱收银台
![支付宝沙箱收银台](./images/ai-bs-06-alipay.png)
-->

### AI 部分：九个"能动手"的接口

所有对话类接口都是流式输出，前端可以做打字机效果。

| 能力 | 接口 | 说明 |
| --- | --- | --- |
| 带身份的对话 | `GET /user/ask` | 模型通过工具拿到当前用户，用用户名跟我说话 |
| 图书简介分析 | `GET /book/fenxi1` | 读简介输出带标签的简要介绍 |
| 购物车推荐 | `GET /book/fenxi2` | 按购物车里书的受众群体推荐相似的 |
| 向量语义推荐 | `GET /book/fenxi3` | 购物车 → 构造查询 → ES 向量检索 TopK → 模型生成 |
| AI 搜索（纯模型） | `GET /book/search1` | 模型读全量图书后返回 JSON 数组 |
| AI 搜索（RAG） | `GET /book/search2` | 向量检索 + 结构化输出直接映射成 `List<Book>` |
| AI 搜索（RAG 直解） | `GET /book/search3` | 命中后直接反序列化，不经过模型二次生成 |
| 查购物车 | `GET /shopping_cart/find` | 模型调工具查 Redis，返回结构化购物车 |
| 自然语言加购 | `POST /shopping_cart/add` `/add2` | 说一句"帮我把《XXX》加两本"，真的写进 Redis |

<!-- TODO 配图 ai-bs-07-chat.png：AI 对话界面，能体现"它知道我是谁"
![AI 对话：模型知道当前用户是谁](./images/ai-bs-07-chat.png)
-->

<!-- TODO 配图 ai-bs-08-ai-cart.png：自然语言加购的前后对比（对话 + 购物车里真的出现了这本书）
![自然语言加购](./images/ai-bs-08-ai-cart.png)
-->

<!-- TODO 配图 ai-bs-09-search.png：语义搜索，用一句描述而不是书名找到书
![语义搜索](./images/ai-bs-09-search.png)
-->

### 接口文档

集成 springdoc-openapi，启动后访问 `http://localhost:8000/swagger-ui.html` 就是全部接口，关键接口都带了 `@Operation` / `@Parameter` 注解。

<!-- TODO 配图 ai-bs-10-swagger.png：Swagger UI 接口列表
![Swagger UI](./images/ai-bs-10-swagger.png)
-->

## 技术选型

| 层次 | 选型 | 说明 |
| --- | --- | --- |
| 语言/框架 | Java 17 / Spring Boot 3.4.5 | Maven 多模块（commons + front），公共能力可被复用 |
| ORM/数据库 | MyBatis-Plus 3.5.12 / MySQL 8 / Druid | 逻辑删除、条件构造器、连接池 |
| 缓存 | Redis | 登录 token + 购物车 Hash |
| 消息队列 | RocketMQ | **事务消息**，订单后续动作异步解耦 |
| 支付 | 支付宝 SDK（沙箱 / RSA2） | 网页支付、交易查询、退款 |
| AI 编排 | Spring AI 1.0.0 | `ChatClient` + Function Calling + 结构化/流式输出 |
| 模型服务 | 硅基流动（OpenAI 兼容协议） | `Pro/zai-org/GLM-4.7` |
| 向量库 | Elasticsearch | 索引 `bookstore`，cosine 相似度 |
| 嵌入模型 | `Qwen/Qwen3-Embedding-8B` | 4096 维 |
| 安全 | JWT（jjwt）+ Redis 双校验 | 支持服务端主动让 token 失效 |
| 接口文档 | springdoc-openapi 2.6.0 | Swagger UI |
| 前端 | Vue 2 + axios + Element UI + jQuery + layer | 无构建工具的原生多页面 |

选型上有三个刻意的决定：

1. **AI 层用 Spring AI，不手写 HTTP**。硅基流动给的是 OpenAI 兼容接口，所以换模型只改 `application.yml` 里的 `base-url` 和 `model`，Java 代码一行不动——**面向接口编程在 AI 这一层同样适用**。
2. **多模块拆分**：`commons` 放实体、Mapper、JWT、工具类这些与业务无关的通用能力，`front` 是唯一可启动的 Web 模块。职责边界清楚，以后加新的业务模块可以直接复用 commons。
3. **前端不上构建工具**：每个页面一个独立 HTML，Vue 2 只做页面级数据绑定，改完刷新就能看效果。**代价是没有组件复用**，但我把省下来的时间全放在了后端和 AI 上。

## 怎么跑起来

需要提前准备四个服务：

| 依赖 | 地址 |
| --- | --- |
| MySQL | `localhost:3306`，库名 `bookstore` |
| Redis | `192.168.150.101:6379` |
| Elasticsearch | `192.168.150.101:9200` |
| RocketMQ NameServer | `192.168.150.101:9876` |

```bash
# commons 会被 front 依赖，必须整体安装
mvn clean install

# 启动后端，端口 8000
mvn spring-boot:run -pl ai-bookstore-front -am
```

:::warning
跑起来之前有三件事要做：设置环境变量 `siliconflow_key`（模型 API Key）、把 `application.yml` 里的支付宝沙箱和中间件地址换成自己的、以及**手动灌一次图书向量**——把全部图书转成 `Document` 调 `vectorStore.add(...)`，否则语义搜索接口会返回空。
:::

:::tip
两个我踩过、值得先检查的环境坑：
**① RocketMQ 起 broker 时 `-n` 的地址必须和客户端连的 namesrv 一致，且 `brokerIP1` 要写成本机对外的真实 IP**。否则 broker 注册不上去（或注册成一个客户端连不到的地址），发送消息时只会得到一句 `No route info of this topic`。
**② Redis 如果快照落盘失败（`rdb_last_bgsave_status: err`）且开着 `stop-writes-on-bgsave-error yes`，它会拒绝一切写命令**——表现出来就是登录接口 500（登录最后一步要把 token 写进 Redis），而读接口一切正常，很容易误判成"密码错了"。
:::

:::caution
`application.yml` 里有支付宝商户私钥、支付宝公钥、ES 密码这类敏感信息。上传到公开仓库前一定要换成环境变量或占位符——私钥泄露是可以被用来伪造支付的。
:::

## 结语

做完这个项目，我最想说的一句话是：**把 AI 接进一个已经开始讲一致性的系统里，难度和做一个聊天页面完全不是一个量级。**

聊天页面的失败只是"回答不好"；而接入业务之后，模型的每一次工具调用都在真实改数据——参数传错、被提示词注入诱导，后果都是真的。所以我在工具体系里做了 `who(token)` 这样"从身份推导参数"的设计，而不是盲信模型传进来的 `userId`。

另一个感受是：**这套系统真正难的部分，其实都不在"功能列表"上**。半消息什么时候提交、外部支付结果怎么变成事务的判定依据、库存不足时怎么优雅撤单——这些才是让它像一个"系统"而不是一个"作业"的东西。

实现细节、代码和踩坑复盘都写在配套的第二篇里：[AI 智能书店（二）：Function Calling、向量检索，和一笔订单的一致性](/posts/ai-bookstore-tech/)。