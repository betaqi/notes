# 全栈设计模式深度学习指南
> 作者视角：10 年全栈架构经验 · React/Vue · Go · Java · Node.js  
> 核心：GoF 经典 23 种模式 · 框架无关 · 前后端双视角

---

## 目录

- [全局学习路径](#全局学习路径)
- [一、创建型模式（Creational）](#一创建型模式creational)
  - [1. 单例模式 Singleton](#1-单例模式-singleton)
  - [2. 工厂方法模式 Factory Method](#2-工厂方法模式-factory-method)
  - [3. 抽象工厂模式 Abstract Factory](#3-抽象工厂模式-abstract-factory)
  - [4. 建造者模式 Builder](#4-建造者模式-builder)
  - [5. 原型模式 Prototype](#5-原型模式-prototype)
- [二、结构型模式（Structural）](#二结构型模式structural)
  - [6. 适配器模式 Adapter](#6-适配器模式-adapter)
  - [7. 桥接模式 Bridge](#7-桥接模式-bridge)
  - [8. 组合模式 Composite](#8-组合模式-composite)
  - [9. 装饰器模式 Decorator](#9-装饰器模式-decorator)
  - [10. 外观模式 Facade](#10-外观模式-facade)
  - [11. 享元模式 Flyweight](#11-享元模式-flyweight)
  - [12. 代理模式 Proxy](#12-代理模式-proxy)
- [三、行为型模式（Behavioral）](#三行为型模式behavioral)
  - [13. 责任链模式 Chain of Responsibility](#13-责任链模式-chain-of-responsibility)
  - [14. 命令模式 Command](#14-命令模式-command)
  - [15. 迭代器模式 Iterator](#15-迭代器模式-iterator)
  - [16. 中介者模式 Mediator](#16-中介者模式-mediator)
  - [17. 备忘录模式 Memento](#17-备忘录模式-memento)
  - [18. 观察者模式 Observer](#18-观察者模式-observer)
  - [19. 状态模式 State](#19-状态模式-state)
  - [20. 策略模式 Strategy](#20-策略模式-strategy)
  - [21. 模板方法模式 Template Method](#21-模板方法模式-template-method)
  - [22. 访问者模式 Visitor](#22-访问者模式-visitor)
  - [23. 解释器模式 Interpreter](#23-解释器模式-interpreter)
- [四、实战重构案例](#四实战重构案例)
- [五、模式选型速查表](#五模式选型速查表)

---

## 全局学习路径

```
阶段一：入门级（1～2 个月）        阶段二：进阶级（2～4 个月）        阶段三：架构级（持续）
──────────────────────────        ──────────────────────────        ──────────────────────
目标：消灭大量 if-else，           目标：解决模块间耦合，             目标：系统级演进，
      让单个类职责清晰。                  降低循环依赖。                     支撑微服务拆分。

重点模式：                         重点模式：                         重点模式：
• 策略模式（替换 if-else）         • 观察者模式（事件解耦）            • 门面模式（服务边界）
• 工厂方法（对象创建统一）          • 中介者模式（组件通信）            • 代理模式（网关、BFF）
• 单例模式（全局资源管理）          • 装饰器模式（功能增强）            • 组合模式（树形结构）
• 模板方法（流程标准化）            • 适配器模式（接口适配）            • 责任链（拦截器管道）
• 建造者模式（复杂对象构建）        • 命令模式（操作封装/撤销）         • 访问者（AST、报表引擎）
```

---

## 一、创建型模式（Creational）

> 核心关注点：**如何创建对象**，将创建逻辑与使用逻辑解耦。

---

### 1. 单例模式 Singleton

#### 意图

保证一个类仅有一个实例，并提供一个全局访问点。

---

#### 🖥️ 前端视角

单例在前端最典型的场景是**全局状态容器**与**服务实例**。

以 React 生态为例，Redux Store 本质上是一个单例——整个应用共享同一棵状态树。Vue 的 `app.config.globalProperties` 注册的服务也遵循单例语义。在非框架场景下，EventBus、WebSocket 连接管理器、主题管理器都是典型的单例需求。

**前端单例的陷阱**：SSR（服务端渲染）场景下，服务端处理多个请求是并发的，如果把单例挂在模块级别（Node.js 的模块缓存），会导致**跨请求状态污染**。正确做法是将单例的作用域绑定到请求上下文，而非进程级别。

```typescript
// 前端：惰性单例 + TypeScript
class ThemeManager {
  private static instance: ThemeManager | null = null;
  private theme: 'light' | 'dark' = 'light';

  private constructor() {
    // 读取 localStorage 初始化
    this.theme = (localStorage.getItem('theme') as any) ?? 'light';
  }

  static getInstance(): ThemeManager {
    if (!ThemeManager.instance) {
      ThemeManager.instance = new ThemeManager();
    }
    return ThemeManager.instance;
  }

  setTheme(theme: 'light' | 'dark') {
    this.theme = theme;
    localStorage.setItem('theme', theme);
    document.documentElement.setAttribute('data-theme', theme);
  }

  getTheme() { return this.theme; }
}

// 使用
const tm = ThemeManager.getInstance();
tm.setTheme('dark');
```

---

#### 🗄️ 后端视角

后端单例最常见于**数据库连接池**、**配置中心客户端**、**缓存客户端**等全局共享资源。并发环境下必须考虑线程安全。

**Go：sync.Once 保证并发安全**

```go
package db

import (
    "database/sql"
    "sync"
    _ "github.com/lib/pq"
)

type Database struct {
    pool *sql.DB
}

var (
    instance *Database
    once     sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        pool, err := sql.Open("postgres", "postgres://...")
        if err != nil {
            panic(err)
        }
        pool.SetMaxOpenConns(25)
        pool.SetMaxIdleConns(5)
        instance = &Database{pool: pool}
    })
    return instance
}
```

**Java：双重检查锁定（DCL）**

```java
public class ConfigCenter {
    private volatile static ConfigCenter instance;
    private Map<String, String> configs;

    private ConfigCenter() {
        // 从配置服务拉取配置
        this.configs = loadFromRemote();
    }

    public static ConfigCenter getInstance() {
        if (instance == null) {                    // 第一次检查，避免加锁开销
            synchronized (ConfigCenter.class) {
                if (instance == null) {            // 第二次检查，保证唯一
                    instance = new ConfigCenter();
                }
            }
        }
        return instance;
    }
}
```

**Node.js：利用模块缓存天然单例**

```javascript
// redis-client.js
const { createClient } = require('redis');

const client = createClient({ url: process.env.REDIS_URL });
client.connect();

module.exports = client; // Node.js 模块首次加载后会被缓存，天然单例
```

---

#### ⚖️ 对比总结

| 维度 | 前端 | 后端 |
|------|------|------|
| 典型场景 | Store、EventBus、WebSocket | 连接池、配置中心、缓存客户端 |
| 主要风险 | SSR 跨请求污染 | 多线程竞争 |
| 解决方案 | 请求级 DI 容器 | sync.Once / volatile DCL |
| 测试难点 | 全局状态难以隔离 | 单例难以 mock，需注入接口 |

---

### 2. 工厂方法模式 Factory Method

#### 意图

定义一个创建对象的接口，让子类决定实例化哪个类。工厂方法使类的实例化延迟到子类。

---

#### 🖥️ 前端视角

前端的工厂方法集中体现在**组件动态渲染**与**渲染器抽象**上。

当一个列表需要根据数据类型渲染不同组件时（文字卡片、图片卡片、视频卡片），使用工厂方法可以将"决定渲染哪个组件"的逻辑从父组件中分离出去。React 中常见的 `renderItem`、Vue 中的动态组件 `<component :is>` 都是工厂方法思想的体现。

```typescript
// 前端：组件工厂 (React 风格伪代码)
type CardType = 'text' | 'image' | 'video';

interface CardProps { data: any; }

// 抽象：每种 Card 组件都实现相同 props 接口
const TextCard  = ({ data }: CardProps) => <div>{data.content}</div>;
const ImageCard = ({ data }: CardProps) => <img src={data.url} />;
const VideoCard = ({ data }: CardProps) => <video src={data.url} />;

// 工厂方法
function createCard(type: CardType): React.FC<CardProps> {
  const registry: Record<CardType, React.FC<CardProps>> = {
    text:  TextCard,
    image: ImageCard,
    video: VideoCard,
  };
  const Component = registry[type];
  if (!Component) throw new Error(`Unknown card type: ${type}`);
  return Component;
}

// 消费方
function Feed({ items }: { items: { type: CardType; data: any }[] }) {
  return (
    <>
      {items.map((item, i) => {
        const Card = createCard(item.type);
        return <Card key={i} data={item.data} />;
      })}
    </>
  );
}
```

**状态管理中的工厂方法**：Redux Toolkit 的 `createSlice` 就是一个工厂——传入配置，产出 reducer + actions，屏蔽了手写 action creator 的细节。

---

#### 🗄️ 后端视角

后端工厂方法的核心价值是**隔离对象创建与业务逻辑**，在多策略、多渠道、多协议场景中尤为关键。

**Go：接口 + 工厂函数**

```go
// 支付渠道抽象
type PaymentGateway interface {
    Charge(amount int64, currency string) (*Receipt, error)
    Refund(txID string, amount int64) error
}

type AlipayGateway  struct { appID string }
type WechatGateway  struct { mchID string }
type StripeGateway  struct { apiKey string }

func (a *AlipayGateway) Charge(amount int64, currency string) (*Receipt, error)  { /* ... */ }
func (w *WechatGateway) Charge(amount int64, currency string) (*Receipt, error)  { /* ... */ }
func (s *StripeGateway) Charge(amount int64, currency string) (*Receipt, error)  { /* ... */ }

// 工厂函数
func NewPaymentGateway(provider string, cfg Config) (PaymentGateway, error) {
    switch provider {
    case "alipay":
        return &AlipayGateway{appID: cfg.AlipayAppID}, nil
    case "wechat":
        return &WechatGateway{mchID: cfg.WechatMchID}, nil
    case "stripe":
        return &StripeGateway{apiKey: cfg.StripeKey}, nil
    default:
        return nil, fmt.Errorf("unsupported payment provider: %s", provider)
    }
}

// 业务层完全不感知具体实现
func (s *OrderService) Pay(orderID, provider string) error {
    gw, _ := NewPaymentGateway(provider, s.cfg)
    _, err := gw.Charge(s.getAmount(orderID), "CNY")
    return err
}
```

**Java：Spring 结合工厂方法**

```java
public interface NotificationSender {
    void send(String target, String message);
}

@Component("email")  public class EmailSender  implements NotificationSender { /* ... */ }
@Component("sms")    public class SmsSender    implements NotificationSender { /* ... */ }
@Component("wechat") public class WechatSender implements NotificationSender { /* ... */ }

@Service
public class NotificationFactory {
    @Autowired
    private Map<String, NotificationSender> senders; // Spring 自动注入所有实现

    public NotificationSender getSender(String channel) {
        NotificationSender sender = senders.get(channel);
        if (sender == null) throw new IllegalArgumentException("Unknown channel: " + channel);
        return sender;
    }
}
```

---

#### ⚖️ 对比总结

| 维度 | 前端 | 后端 |
|------|------|------|
| 典型场景 | 动态组件渲染、渲染器选择 | 支付/通知/存储渠道切换 |
| 扩展方式 | 注册表 Map 新增 entry | 实现接口 + 工厂 switch 新增 case |
| 框架支持 | React.createElement、Vue resolveComponent | Spring @Component Map 注入 |
| 测试优势 | mock 工厂返回 stub 组件 | mock 接口隔离外部依赖 |

---

### 3. 抽象工厂模式 Abstract Factory

#### 意图

提供一个创建**一系列相关或互相依赖对象**的接口，而无需指定具体类。

---

#### 🖥️ 前端视角

抽象工厂在前端的绝佳案例是**UI 主题系统**。一套"暗黑主题"需要同时提供 Button、Input、Modal、Toast 等一整族组件，它们共享同一视觉语言。抽象工厂保证了这一族组件的内聚性——不会出现"暗黑按钮配亮色输入框"的混乱。

```typescript
// 抽象产品族
interface Button  { render(): JSX.Element; }
interface Input   { render(): JSX.Element; }
interface Dialog  { render(): JSX.Element; }

// 抽象工厂
interface UIFactory {
  createButton(): Button;
  createInput():  Input;
  createDialog(): Dialog;
}

// 具体工厂：亮色主题
class LightThemeFactory implements UIFactory {
  createButton() { return new LightButton(); }
  createInput()  { return new LightInput();  }
  createDialog() { return new LightDialog(); }
}

// 具体工厂：暗色主题
class DarkThemeFactory implements UIFactory {
  createButton() { return new DarkButton(); }
  createInput()  { return new DarkInput();  }
  createDialog() { return new DarkDialog(); }
}

// 消费：页面组件只依赖抽象工厂
function buildPage(factory: UIFactory) {
  const btn    = factory.createButton();
  const input  = factory.createInput();
  const dialog = factory.createDialog();
  // 整套组件风格一致
}

// 切换主题只需切换工厂
const factory = userPreference === 'dark'
  ? new DarkThemeFactory()
  : new LightThemeFactory();
buildPage(factory);
```

跨平台场景同理：Web 工厂 vs 移动端工厂，各自产出适配平台的组件族。

---

#### 🗄️ 后端视角

后端典型场景是**多云/多存储抽象**。一套系统需要在 AWS、阿里云、私有化部署三种环境下运行，存储（对象存储）、消息队列、邮件服务的具体实现各不同，但接口一致。抽象工厂可以保证同一套业务代码在不同环境下无缝切换。

```go
// Go: 多云基础设施工厂

type ObjectStorage interface {
    Upload(key string, data []byte) error
    Download(key string) ([]byte, error)
}

type MessageQueue interface {
    Publish(topic string, msg []byte) error
    Subscribe(topic string, handler func([]byte)) error
}

// 抽象工厂接口
type CloudFactory interface {
    NewObjectStorage() ObjectStorage
    NewMessageQueue()  MessageQueue
}

// AWS 工厂
type AWSFactory struct { region string }
func (f *AWSFactory) NewObjectStorage() ObjectStorage { return &S3Storage{region: f.region} }
func (f *AWSFactory) NewMessageQueue()  MessageQueue  { return &SQSQueue{region: f.region}  }

// 阿里云工厂
type AliyunFactory struct { endpoint string }
func (f *AliyunFactory) NewObjectStorage() ObjectStorage { return &OSSStorage{endpoint: f.endpoint} }
func (f *AliyunFactory) NewMessageQueue()  MessageQueue  { return &MNSQueue{endpoint: f.endpoint}  }

// 业务服务：只依赖工厂接口
type UploadService struct {
    storage ObjectStorage
    queue   MessageQueue
}

func NewUploadService(factory CloudFactory) *UploadService {
    return &UploadService{
        storage: factory.NewObjectStorage(),
        queue:   factory.NewMessageQueue(),
    }
}
```

---

#### ⚖️ 对比总结

| 维度 | 前端 | 后端 |
|------|------|------|
| 典型场景 | 主题系统、跨平台 UI 组件族 | 多云基础设施、多租户环境隔离 |
| 核心价值 | 保证同族组件视觉/行为一致性 | 保证同族服务接口一致性 |
| 扩展成本 | 新主题 = 新工厂类（实现全族接口）| 新云厂商 = 新工厂类 |
| vs 工厂方法 | 工厂方法创建单一产品，抽象工厂创建产品族 | 同左 |

---

### 4. 建造者模式 Builder

#### 意图

将一个复杂对象的**构建**与它的**表示**分离，使同样的构建过程可以创建不同的表示。

---

#### 🖥️ 前端视角

前端中建造者模式最直接的应用是**复杂表单配置**与**查询条件构造**。当一个搜索请求有十几个可选参数时，链式 Builder 比逐一传入 options 对象可读性高得多，且能做参数校验。

```typescript
// 前端：搜索查询 Builder
class SearchQueryBuilder {
  private query: Record<string, any> = {};

  keyword(kw: string): this {
    this.query.keyword = kw.trim();
    return this;
  }

  dateRange(from: Date, to: Date): this {
    if (from > to) throw new Error('from must be before to');
    this.query.dateFrom = from.toISOString();
    this.query.dateTo   = to.toISOString();
    return this;
  }

  category(id: number): this {
    this.query.categoryId = id;
    return this;
  }

  page(num: number, size = 20): this {
    this.query.page     = num;
    this.query.pageSize = size;
    return this;
  }

  sort(field: string, order: 'asc' | 'desc' = 'desc'): this {
    this.query.sortField = field;
    this.query.sortOrder = order;
    return this;
  }

  build(): URLSearchParams {
    return new URLSearchParams(
      Object.entries(this.query).map(([k, v]) => [k, String(v)])
    );
  }
}

// 使用：声明式、可读
const params = new SearchQueryBuilder()
  .keyword('设计模式')
  .category(42)
  .dateRange(new Date('2024-01-01'), new Date())
  .sort('createdAt')
  .page(1, 10)
  .build();

fetch(`/api/search?${params}`);
```

---

#### 🗄️ 后端视角

后端建造者最典型的场景是**SQL 查询构造器**、**HTTP 客户端配置**、**复杂领域对象初始化**。

**Go：HTTP 客户端 Builder**

```go
type HTTPClientBuilder struct {
    timeout     time.Duration
    retries     int
    baseURL     string
    headers     map[string]string
    rateLimiter *rate.Limiter
}

func NewHTTPClientBuilder() *HTTPClientBuilder {
    return &HTTPClientBuilder{
        timeout: 30 * time.Second,
        headers: make(map[string]string),
    }
}

func (b *HTTPClientBuilder) WithTimeout(d time.Duration) *HTTPClientBuilder {
    b.timeout = d; return b
}
func (b *HTTPClientBuilder) WithRetries(n int) *HTTPClientBuilder {
    b.retries = n; return b
}
func (b *HTTPClientBuilder) WithBaseURL(url string) *HTTPClientBuilder {
    b.baseURL = url; return b
}
func (b *HTTPClientBuilder) WithHeader(k, v string) *HTTPClientBuilder {
    b.headers[k] = v; return b
}
func (b *HTTPClientBuilder) WithRateLimit(rps float64) *HTTPClientBuilder {
    b.rateLimiter = rate.NewLimiter(rate.Limit(rps), int(rps)); return b
}

func (b *HTTPClientBuilder) Build() (*MyHTTPClient, error) {
    if b.baseURL == "" {
        return nil, errors.New("baseURL is required")
    }
    return &MyHTTPClient{
        client:      &http.Client{Timeout: b.timeout},
        retries:     b.retries,
        baseURL:     b.baseURL,
        headers:     b.headers,
        rateLimiter: b.rateLimiter,
    }, nil
}

// 使用
client, _ := NewHTTPClientBuilder().
    WithBaseURL("https://api.example.com").
    WithTimeout(10 * time.Second).
    WithRetries(3).
    WithHeader("Authorization", "Bearer "+token).
    WithRateLimit(100).
    Build()
```

**Java：lombok @Builder 简化**

```java
@Builder
@Data
public class EmailMessage {
    @NonNull private String to;
    @NonNull private String subject;
    private String body;
    private List<String> cc;
    private List<Attachment> attachments;
    @Builder.Default private Priority priority = Priority.NORMAL;
}

// 使用
EmailMessage msg = EmailMessage.builder()
    .to("user@example.com")
    .subject("您的订单已发货")
    .body("订单 #12345 已于今日发出...")
    .priority(Priority.HIGH)
    .build();
```

---

#### ⚖️ 对比总结

| 维度 | 前端 | 后端 |
|------|------|------|
| 典型场景 | 查询参数、表单配置、动画配置 | SQL 构造器、HTTP 客户端、领域对象 |
| 核心价值 | 链式调用 + 参数校验 + 可读性 | 分步骤构建复杂对象，隔离校验逻辑 |
| 常见变体 | Fluent Interface | Director + Builder（固定构建流程）|
| 测试优势 | 易于构造测试数据 | 同左，Builder 可作为 Test Fixture 工具 |

---

### 5. 原型模式 Prototype

#### 意图

用原型实例指定创建对象的种类，并通过**拷贝**这些原型创建新的对象。

---

#### 🖥️ 前端视角

前端状态管理中不可变更新（Immutable Update）就是原型思想的应用。Redux 的 reducer、Immer.js 的 produce，本质上都是"基于当前状态克隆出新状态再修改"。

```typescript
// 前端：深度克隆与不可变更新
interface FormState {
  user: { name: string; email: string; };
  preferences: { theme: string; notifications: boolean; };
}

// 浅克隆（结构共享）
const updateUserName = (state: FormState, name: string): FormState => ({
  ...state,                        // 原型：保留其他字段
  user: { ...state.user, name }    // 只更新变化的部分
});

// 深克隆场景（配置模板复制）
const cloneTemplate = <T>(template: T): T =>
  JSON.parse(JSON.stringify(template));   // 简单场景

// 生产环境推荐 structuredClone（现代浏览器原生支持）
const cloneDeep = <T>(obj: T): T => structuredClone(obj);

// 图表配置的原型复用
const baseChartConfig = {
  type: 'line',
  options: { responsive: true, animation: { duration: 300 } },
  plugins: ['tooltip', 'legend']
};

const revenueChart = {
  ...cloneDeep(baseChartConfig),    // 克隆原型
  data: revenueData,                 // 差异化填充
  options: { ...baseChartConfig.options, title: '月度营收' }
};
```

---

#### 🗄️ 后端视角

后端场景：**对象池**、**配置模板克隆**、**文档/报表模板复制**。

```go
// Go: 原型接口 + 克隆
type Cloneable interface {
    Clone() Cloneable
}

type ReportTemplate struct {
    Title       string
    Sections    []Section
    Styles      map[string]Style
    Footer      string
}

func (r *ReportTemplate) Clone() Cloneable {
    // 深拷贝 Sections 切片
    sections := make([]Section, len(r.Sections))
    copy(sections, r.Sections)

    styles := make(map[string]Style, len(r.Styles))
    for k, v := range r.Styles {
        styles[k] = v
    }

    return &ReportTemplate{
        Title:    r.Title,
        Sections: sections,
        Styles:   styles,
        Footer:   r.Footer,
    }
}

// 使用：从模板克隆，填充差异数据
baseTemplate := &ReportTemplate{
    Title:    "月度报告",
    Sections: defaultSections(),
    Styles:   defaultStyles(),
    Footer:   "© 2025 Company",
}

juneReport := baseTemplate.Clone().(*ReportTemplate)
juneReport.Title = "2025 年 6 月月度报告"
juneReport.Sections = append(juneReport.Sections, juneSpecificSection)
```

---

## 二、结构型模式（Structural）

> 核心关注点：**如何组合对象/类**，形成更大的结构，同时保持结构的灵活和高效。

---

### 6. 适配器模式 Adapter

#### 意图

将一个类的接口**转换**成客户希望的另外一个接口。Adapter 模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

---

#### 🖥️ 前端视角

前端最频繁使用适配器的场景是**接口数据转换**。后端返回的数据结构未必符合前端组件的 props 结构，适配层将两者解耦，避免 UI 组件直接依赖后端数据格式。

```typescript
// 后端返回的格式（不受控）
interface ApiUser {
  user_id: number;
  full_name: string;
  email_address: string;
  created_at: string;    // ISO 字符串
  is_active: 0 | 1;
}

// 前端组件期望的格式
interface UIUser {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
  active: boolean;
  avatarUrl: string;
}

// 适配器函数
function adaptApiUser(apiUser: ApiUser): UIUser {
  return {
    id:        apiUser.user_id,
    name:      apiUser.full_name,
    email:     apiUser.email_address,
    createdAt: new Date(apiUser.created_at),
    active:    apiUser.is_active === 1,
    avatarUrl: `https://avatar.service.com/${apiUser.user_id}`,
  };
}

// 在 API 层统一转换，UI 组件不感知后端格式
async function fetchUsers(): Promise<UIUser[]> {
  const raw: ApiUser[] = await api.get('/users');
  return raw.map(adaptApiUser);
}
```

第三方 SDK 集成也是适配器的典型场景。当从地图 SDK A 切换到 SDK B 时，只需替换适配器，业务代码零改动。

---

#### 🗄️ 后端视角

后端适配器常见于**外部服务集成**和**遗留系统对接**。微服务中，上游系统的数据格式往往不受控，适配器层是防腐层（Anti-Corruption Layer，DDD 概念）的核心实现。

```go
// Go: 集成旧版支付系统

// 旧版系统的接口（不可修改）
type LegacyPaymentSystem struct{}
func (l *LegacyPaymentSystem) MakePayment(cardNo string, expiry string, amountCents int) (string, error) {
    // 旧系统逻辑...
    return "TXN_12345", nil
}

// 新系统期望的接口
type PaymentProcessor interface {
    Process(req PaymentRequest) (*PaymentResult, error)
}

type PaymentRequest struct {
    CardNumber string
    ExpiryDate string    // MM/YY 格式
    Amount     float64   // 元为单位
}

type PaymentResult struct {
    TransactionID string
    Success       bool
}

// 适配器：将旧接口包装成新接口
type LegacyPaymentAdapter struct {
    legacy *LegacyPaymentSystem
}

func (a *LegacyPaymentAdapter) Process(req PaymentRequest) (*PaymentResult, error) {
    // 数据转换
    amountCents := int(req.Amount * 100)
    txID, err := a.legacy.MakePayment(req.CardNumber, req.ExpiryDate, amountCents)
    if err != nil {
        return &PaymentResult{Success: false}, err
    }
    return &PaymentResult{TransactionID: txID, Success: true}, nil
}

// 业务层只依赖 PaymentProcessor 接口，可无缝切换新旧系统
type OrderService struct {
    payment PaymentProcessor
}
```

---

### 7. 桥接模式 Bridge

#### 意图

将**抽象部分**与它的**实现部分**分离，使它们都可以独立地变化。

---

#### 🖥️ 前端视角

桥接模式在前端体现为**渲染引擎与业务逻辑分离**。以图表库为例，图表的数据处理（排序、过滤、聚合）是抽象部分，而渲染（Canvas / SVG / WebGL）是实现部分，两者可以独立扩展。

```typescript
// 实现部分（渲染器抽象）
interface Renderer {
  drawLine(points: Point[]): void;
  drawBar(x: number, y: number, w: number, h: number): void;
  clear(): void;
}

class SVGRenderer   implements Renderer { /* SVG 实现 */    }
class CanvasRenderer implements Renderer { /* Canvas 实现 */ }

// 抽象部分（图表抽象）
abstract class Chart {
  constructor(protected renderer: Renderer) {}   // 桥接：持有实现的引用
  abstract draw(data: number[]): void;
}

// 精化抽象
class LineChart extends Chart {
  draw(data: number[]) {
    this.renderer.clear();
    const points = data.map((v, i) => ({ x: i * 50, y: 200 - v }));
    this.renderer.drawLine(points);
  }
}

class BarChart extends Chart {
  draw(data: number[]) {
    this.renderer.clear();
    data.forEach((v, i) => this.renderer.drawBar(i * 60, 200 - v, 40, v));
  }
}

// 使用：抽象和实现自由组合
const lineWithSVG    = new LineChart(new SVGRenderer());
const barWithCanvas  = new BarChart(new CanvasRenderer());
const lineWithCanvas = new LineChart(new CanvasRenderer());  // 轻松新增组合
```

---

#### 🗄️ 后端视角

后端桥接模式体现在**消息发送渠道与消息内容解耦**、**数据库驱动与业务逻辑解耦**等场景。

```java
// Java: 通知内容（抽象）与发送渠道（实现）的桥接

// 实现层：发送渠道
public interface MessageChannel {
    void send(String recipient, String content);
}

@Component public class SMSChannel   implements MessageChannel { /* 短信 */ }
@Component public class EmailChannel implements MessageChannel { /* 邮件 */ }
@Component public class PushChannel  implements MessageChannel { /* 推送 */ }

// 抽象层：通知类型
public abstract class Notification {
    protected MessageChannel channel;

    public Notification(MessageChannel channel) {
        this.channel = channel;
    }

    public abstract void send(String recipient, NotificationPayload payload);
}

// 精化：订单通知
public class OrderNotification extends Notification {
    public OrderNotification(MessageChannel channel) { super(channel); }

    @Override
    public void send(String recipient, NotificationPayload payload) {
        String content = String.format("您的订单 %s 状态已更新为：%s",
            payload.getOrderId(), payload.getStatus());
        channel.send(recipient, content);   // 委托给具体渠道
    }
}

// 自由组合：订单通知 × 短信渠道、订单通知 × 邮件渠道
new OrderNotification(new SMSChannel()).send(phone, payload);
new OrderNotification(new EmailChannel()).send(email, payload);
```

---

### 8. 组合模式 Composite

#### 意图

将对象组合成**树形结构**以表示"部分-整体"的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。

---

#### 🖥️ 前端视角

组合模式是前端**组件树**的底层逻辑。React / Vue 的组件体系就是组合模式的最佳实践——叶子组件（Button、Input）与容器组件（Form、Page）都实现相同的"组件接口"（render/mount/unmount）。

另一个典型场景是**菜单/权限树**的渲染：

```typescript
interface MenuItem {
  id: string;
  label: string;
  icon?: string;
  children?: MenuItem[];   // 叶子节点没有 children
  action?: () => void;
}

// 统一渲染函数，叶子和容器处理逻辑一致
function renderMenu(items: MenuItem[], depth = 0): JSX.Element {
  return (
    <ul className={`menu depth-${depth}`}>
      {items.map(item => (
        <li key={item.id}>
          <span onClick={item.action}>{item.icon} {item.label}</span>
          {item.children && renderMenu(item.children, depth + 1)}
        </li>
      ))}
    </ul>
  );
}

// 数据：叶子和节点混合，消费方无需区分
const menu: MenuItem[] = [
  { id: '1', label: '首页', action: () => navigate('/') },
  {
    id: '2', label: '系统管理', children: [
      { id: '2-1', label: '用户管理', action: () => navigate('/users') },
      { id: '2-2', label: '角色管理', action: () => navigate('/roles') },
    ]
  },
];
```

---

#### 🗄️ 后端视角

后端组合模式典型场景：**规则引擎**（条件组合）、**权限树**、**文件系统抽象**、**组织架构**。

```go
// Go: 权限规则引擎（组合条件）

type Rule interface {
    Evaluate(ctx Context) bool
}

// 叶子节点：单个条件
type RoleRule struct { requiredRole string }
func (r *RoleRule) Evaluate(ctx Context) bool {
    return ctx.User.HasRole(r.requiredRole)
}

type IPRule struct { allowedCIDR string }
func (r *IPRule) Evaluate(ctx Context) bool {
    return cidr.Contains(r.allowedCIDR, ctx.RemoteIP)
}

// 组合节点：AND
type AndRule struct { rules []Rule }
func (a *AndRule) Evaluate(ctx Context) bool {
    for _, r := range a.rules {
        if !r.Evaluate(ctx) { return false }
    }
    return true
}

// 组合节点：OR
type OrRule struct { rules []Rule }
func (o *OrRule) Evaluate(ctx Context) bool {
    for _, r := range o.rules {
        if r.Evaluate(ctx) { return true }
    }
    return false
}

// 构建复杂规则：(role=admin OR role=superuser) AND ip in 内网
rule := &AndRule{rules: []Rule{
    &OrRule{rules: []Rule{
        &RoleRule{requiredRole: "admin"},
        &RoleRule{requiredRole: "superuser"},
    }},
    &IPRule{allowedCIDR: "10.0.0.0/8"},
}}

if !rule.Evaluate(ctx) {
    return errors.New("access denied")
}
```

---

### 9. 装饰器模式 Decorator

#### 意图

动态地给一个对象**添加额外的职责**。就增加功能来说，装饰器比生成子类更为灵活。

---

#### 🖥️ 前端视角

前端装饰器最典型的场景是**高阶组件（HOC）**和 **React Hooks 的封装链**。TypeScript/ES2022 的 `@Decorator` 语法则用于类组件的属性增强。

```typescript
// 高阶组件：装饰器风格
// withAuth：给任意页面组件加上鉴权能力
function withAuth<P extends {}>(Component: React.ComponentType<P>) {
  return function AuthGuard(props: P) {
    const { isAuthenticated, loading } = useAuth();
    if (loading) return <Spinner />;
    if (!isAuthenticated) return <Navigate to="/login" />;
    return <Component {...props} />;   // 透传所有 props，不影响原组件
  };
}

// withLogging：给任意组件加上渲染日志
function withLogging<P extends {}>(Component: React.ComponentType<P>, name: string) {
  return function LoggedComponent(props: P) {
    useEffect(() => {
      console.log(`[Render] ${name}`, props);
      return () => console.log(`[Unmount] ${name}`);
    });
    return <Component {...props} />;
  };
}

// 叠加装饰：先鉴权，再加日志
const ProfilePage = withLogging(withAuth(RawProfilePage), 'ProfilePage');
```

---

#### 🗄️ 后端视角

后端装饰器最强大的场景是**中间件链**和**服务层能力增强**（缓存、限流、日志、事务）。

**Go：HTTP Handler 装饰链**

```go
type Middleware func(http.Handler) http.Handler

// 日志装饰器
func WithLogging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)                              // 调用被装饰的 Handler
        log.Printf("%s %s %v", r.Method, r.URL, time.Since(start))
    })
}

// 认证装饰器
func WithAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if !validateToken(token) {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// 限流装饰器
func WithRateLimit(limit int) Middleware {
    limiter := rate.NewLimiter(rate.Limit(limit), limit)
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// 组合：从外到内依次包裹
handler := WithLogging(WithAuth(WithRateLimit(100)(coreHandler)))
```

**Java：装饰器 + 缓存**

```java
public interface UserRepository {
    Optional<User> findById(Long id);
    List<User> findAll();
}

// 缓存装饰器
public class CachingUserRepository implements UserRepository {
    private final UserRepository delegate;    // 被装饰的对象
    private final Cache<Long, User> cache;

    @Override
    public Optional<User> findById(Long id) {
        User cached = cache.getIfPresent(id);
        if (cached != null) return Optional.of(cached);

        Optional<User> user = delegate.findById(id);
        user.ifPresent(u -> cache.put(id, u));
        return user;
    }
}

// 审计日志装饰器（可继续叠加）
public class AuditingUserRepository implements UserRepository {
    private final UserRepository delegate;

    @Override
    public Optional<User> findById(Long id) {
        log.info("Fetching user {}", id);
        Optional<User> user = delegate.findById(id);
        log.info("Fetched user: {}", user.isPresent());
        return user;
    }
}

// 组合：DB → 缓存 → 审计
UserRepository repo = new AuditingUserRepository(
    new CachingUserRepository(
        new DatabaseUserRepository(dataSource)
    )
);
```

---

### 10. 外观模式 Facade

#### 意图

为子系统中的一组接口提供一个**一致的界面**，Facade 模式定义了一个高层接口，这个接口使得子系统更加容易使用。

---

#### 🖥️ 前端视角

前端外观模式常见于**API 层封装**和**第三方 SDK 封装**。将底层 `fetch`、错误处理、token 刷新、序列化等细节隐藏在一个简洁的 `api` 对象后面。

```typescript
// 外观：封装 fetch 的所有复杂性
class ApiClient {
  private baseURL = '/api/v1';
  private token: string | null = null;

  private async request<T>(method: string, path: string, data?: any): Promise<T> {
    const res = await fetch(`${this.baseURL}${path}`, {
      method,
      headers: {
        'Content-Type': 'application/json',
        ...(this.token && { Authorization: `Bearer ${this.token}` }),
      },
      body: data ? JSON.stringify(data) : undefined,
    });

    if (res.status === 401) {
      await this.refreshToken();      // 自动刷新 token
      return this.request(method, path, data);
    }

    if (!res.ok) throw new ApiError(res.status, await res.json());
    return res.json();
  }

  // 简洁的门面方法
  get<T>(path: string)              { return this.request<T>('GET', path);         }
  post<T>(path: string, data: any)  { return this.request<T>('POST', path, data);  }
  put<T>(path: string, data: any)   { return this.request<T>('PUT', path, data);   }
  delete<T>(path: string)           { return this.request<T>('DELETE', path);       }

  private async refreshToken() { /* token 刷新逻辑 */ }
}

export const api = new ApiClient();

// 业务代码极其简洁
const users = await api.get<User[]>('/users');
await api.post('/orders', { productId: 1, qty: 2 });
```

---

#### 🗄️ 后端视角

后端外观模式最重要的应用是**BFF（Backend for Frontend）**和**服务聚合层**。微服务架构中，前端往往需要聚合多个下游服务的数据，BFF 就是这个外观层。

```go
// Go: BFF 聚合订单详情页所需数据

type OrderDetailFacade struct {
    orderSvc   OrderService
    userSvc    UserService
    productSvc ProductService
    logisticSvc LogisticService
}

type OrderDetailResponse struct {
    Order    *Order
    User     *User
    Products []*Product
    Tracking *TrackingInfo
}

func (f *OrderDetailFacade) GetOrderDetail(ctx context.Context, orderID string) (*OrderDetailResponse, error) {
    // 并发调用多个下游服务
    var (
        order    *Order
        user     *User
        products []*Product
        tracking *TrackingInfo
        eg       errgroup.Group
    )

    eg.Go(func() error {
        var err error
        order, err = f.orderSvc.Get(ctx, orderID)
        return err
    })

    eg.Go(func() error {
        var err error
        user, err = f.userSvc.GetByOrderID(ctx, orderID)
        return err
    })

    // 注意：products 依赖 order，需要串行
    if err := eg.Wait(); err != nil {
        return nil, err
    }

    var eg2 errgroup.Group
    eg2.Go(func() error {
        var err error
        products, err = f.productSvc.GetByIDs(ctx, order.ProductIDs)
        return err
    })
    eg2.Go(func() error {
        var err error
        tracking, err = f.logisticSvc.GetTracking(ctx, order.LogisticNo)
        return err
    })

    if err := eg2.Wait(); err != nil {
        return nil, err
    }

    return &OrderDetailResponse{Order: order, User: user, Products: products, Tracking: tracking}, nil
}
```

前端只需一次请求 `GET /order-detail/{id}`，无需了解背后的微服务拓扑。

---

### 11. 享元模式 Flyweight

#### 意图

运用共享技术有效地支持**大量细粒度的对象**，将对象状态分为**内部状态**（共享）和**外部状态**（不共享）。

---

#### 🖥️ 前端视角

前端享元模式的核心应用是**虚拟列表**（Virtual Scrolling）。渲染 10 万行数据时，只创建可视区域内的 DOM 节点（享元），滚动时复用这些节点并更新其外部状态（显示的数据）。

```typescript
// 享元思想：DOM 节点池
class ListItemPool {
  private pool: HTMLElement[] = [];

  acquire(): HTMLElement {
    return this.pool.pop() || this.createElement();   // 复用或新建
  }

  release(el: HTMLElement) {
    el.style.display = 'none';
    this.pool.push(el);   // 归还到池中
  }

  private createElement(): HTMLElement {
    const el = document.createElement('div');
    el.className = 'list-item';    // 内部状态：样式、结构（共享）
    return el;
  }
}

// 外部状态（每行不同的数据）由调用方注入
function renderVisibleItems(pool: ListItemPool, data: any[], visibleRange: [number, number]) {
  const [start, end] = visibleRange;
  data.slice(start, end).forEach((item, i) => {
    const el = pool.acquire();
    el.style.display = '';
    el.style.top = `${(start + i) * 40}px`;    // 外部状态：位置
    el.textContent = item.name;                  // 外部状态：数据
    container.appendChild(el);
  });
}
```

Canvas 游戏中的**粒子系统**、图标字体（多处使用同一字形数据）都是前端享元的典型应用。

---

#### 🗄️ 后端视角

后端享元最常见于**连接池**（数据库连接、HTTP 连接）和**对象池**（线程池本质也是享元）。共享昂贵的连接对象，通过外部状态区分每次使用的上下文。

```go
// Go: 通用对象池（享元工厂）
type ByteBufferPool struct {
    pool sync.Pool
}

func NewByteBufferPool(defaultSize int) *ByteBufferPool {
    return &ByteBufferPool{
        pool: sync.Pool{
            New: func() interface{} {
                return bytes.NewBuffer(make([]byte, 0, defaultSize))
            },
        },
    }
}

func (p *ByteBufferPool) Get() *bytes.Buffer {
    buf := p.pool.Get().(*bytes.Buffer)
    buf.Reset()   // 清除外部状态（上次的内容）
    return buf
}

func (p *ByteBufferPool) Put(buf *bytes.Buffer) {
    p.pool.Put(buf)
}

// 使用：避免频繁分配大缓冲区
var bufPool = NewByteBufferPool(4096)

func buildJSONResponse(data interface{}) []byte {
    buf := bufPool.Get()
    defer bufPool.Put(buf)

    json.NewEncoder(buf).Encode(data)
    result := make([]byte, buf.Len())
    copy(result, buf.Bytes())
    return result
}
```

---

### 12. 代理模式 Proxy

#### 意图

为其他对象提供一种代理以**控制对这个对象的访问**。

---

#### 🖥️ 前端视角

前端代理模式最典型的应用是 **ES6 `Proxy`** 对象。Vue 3 的响应式系统（Reactive）就是基于 `Proxy` 实现的——对数据对象的读写被拦截，从而驱动视图更新。

```typescript
// 表单验证代理
function createValidatedForm<T extends object>(target: T, rules: Partial<Record<keyof T, (v: any) => string | null>>) {
  return new Proxy(target, {
    set(obj, prop, value) {
      const rule = rules[prop as keyof T];
      if (rule) {
        const error = rule(value);
        if (error) throw new ValidationError(String(prop), error);
      }
      return Reflect.set(obj, prop, value);
    }
  });
}

const form = createValidatedForm(
  { email: '', age: 0 },
  {
    email: v => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) ? null : '邮箱格式不正确',
    age:   v => (v >= 0 && v <= 150) ? null : '年龄不合法',
  }
);

form.email = 'user@example.com';  // ✅ 通过
form.email = 'not-an-email';      // ❌ 抛出 ValidationError
```

---

#### 🗄️ 后端视角

后端代理模式覆盖：**缓存代理**（加速读取）、**保护代理**（权限控制）、**远程代理**（gRPC/REST 客户端）、**日志代理**。

```java
// Java: 动态代理实现 AOP 式缓存
public class CachingProxy implements InvocationHandler {
    private final Object target;
    private final Cache cache;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 只缓存带有 @Cacheable 注解的方法
        if (!method.isAnnotationPresent(Cacheable.class)) {
            return method.invoke(target, args);
        }

        String cacheKey = method.getName() + Arrays.toString(args);
        Object cached = cache.get(cacheKey);
        if (cached != null) return cached;

        Object result = method.invoke(target, args);
        cache.put(cacheKey, result);
        return result;
    }

    @SuppressWarnings("unchecked")
    public static <T> T wrap(T target, Cache cache) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new CachingProxy(target, cache)
        );
    }
}

// 使用：为任意 Service 接口透明加上缓存
UserService cached = CachingProxy.wrap(new UserServiceImpl(), redisCache);
```

---

## 三、行为型模式（Behavioral）

> 核心关注点：**对象间的通信**，职责如何分配，算法如何封装。

---

### 13. 责任链模式 Chain of Responsibility

#### 意图

使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递请求，直到有一个对象处理它为止。

---

#### 🖥️ 前端视角

前端责任链模式体现在**事件冒泡**机制（浏览器原生支持）以及**axios 拦截器链**、**Redux 中间件管道**。

```typescript
// axios 风格的请求拦截器链
type Handler = (req: Request, next: () => Promise<Response>) => Promise<Response>;

class RequestPipeline {
  private handlers: Handler[] = [];

  use(handler: Handler): this {
    this.handlers.push(handler);
    return this;
  }

  async execute(req: Request): Promise<Response> {
    const run = (index: number): Promise<Response> => {
      if (index >= this.handlers.length) {
        return fetch(req);   // 终点：真实请求
      }
      return this.handlers[index](req, () => run(index + 1));
    };
    return run(0);
  }
}

// 组装管道
const pipeline = new RequestPipeline()
  .use(async (req, next) => {                         // 认证
    req.headers.set('Authorization', `Bearer ${getToken()}`);
    return next();
  })
  .use(async (req, next) => {                         // 日志
    console.log('[Request]', req.url);
    const res = await next();
    console.log('[Response]', res.status);
    return res;
  })
  .use(async (req, next) => {                         // 错误重试
    try { return await next(); }
    catch { return next(); }                           // 重试一次
  });
```

---

#### 🗄️ 后端视角

后端责任链是**Middleware 管道**的核心实现，也是框架（Gin、Spring Security、Express）的设计基础。在业务层，审批流、风控规则引擎也是责任链的典型应用。

```go
// Go: Gin 风格中间件链
type HandlerFunc func(ctx *Context)

type Context struct {
    Request  *http.Request
    Writer   http.ResponseWriter
    handlers []HandlerFunc
    index    int
    Keys     map[string]interface{}
}

func (c *Context) Next() {
    c.index++
    for c.index < len(c.handlers) {
        c.handlers[c.index](c)
        c.index++
    }
}

func (c *Context) Abort() { c.index = len(c.handlers) }

// 中间件示例
func AuthMiddleware() HandlerFunc {
    return func(c *Context) {
        token := c.Request.Header.Get("Authorization")
        user, err := validateToken(token)
        if err != nil {
            http.Error(c.Writer, "Unauthorized", 401)
            c.Abort()    // 中断后续处理
            return
        }
        c.Keys["user"] = user
        c.Next()         // 传递给下一个处理器
    }
}

func RateLimitMiddleware(limit int) HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(limit), limit)
    return func(c *Context) {
        if !limiter.Allow() {
            http.Error(c.Writer, "Too Many Requests", 429)
            c.Abort()
            return
        }
        c.Next()
    }
}
```

---

### 14. 命令模式 Command

#### 意图

将一个**请求封装为一个对象**，从而使你可用不同的请求对客户进行参数化；对请求排队或记录请求日志，以及支持可撤销的操作。

---

#### 🖥️ 前端视角

前端命令模式最直接的应用是**撤销/重做（Undo/Redo）**功能，以及**宏录制**。富文本编辑器、图形编辑器（如 Figma）的核心就是命令栈。

```typescript
interface Command {
  execute(): void;
  undo(): void;
}

// 具体命令
class AddTextCommand implements Command {
  constructor(
    private editor: TextEditor,
    private text: string,
    private position: number
  ) {}

  execute() { this.editor.insertText(this.position, this.text); }
  undo()    { this.editor.deleteText(this.position, this.text.length); }
}

// 命令历史管理器（调用者）
class CommandHistory {
  private history: Command[] = [];
  private cursor = -1;

  execute(cmd: Command) {
    // 清除 cursor 之后的历史（分支丢弃）
    this.history = this.history.slice(0, this.cursor + 1);
    cmd.execute();
    this.history.push(cmd);
    this.cursor++;
  }

  undo() {
    if (this.cursor < 0) return;
    this.history[this.cursor--].undo();
  }

  redo() {
    if (this.cursor >= this.history.length - 1) return;
    this.history[++this.cursor].execute();
  }
}
```

---

#### 🗄️ 后端视角

后端命令模式应用于**任务队列**、**事件溯源（Event Sourcing）**、**批量操作事务回滚**。

```go
// Go: 任务队列 + 可回滚命令
type Command interface {
    Execute(ctx context.Context) error
    Rollback(ctx context.Context) error
    Name() string
}

// 数据库迁移命令
type CreateTableCommand struct {
    db        *sql.DB
    tableName string
    schema    string
}

func (c *CreateTableCommand) Execute(ctx context.Context) error {
    _, err := c.db.ExecContext(ctx, fmt.Sprintf("CREATE TABLE %s (%s)", c.tableName, c.schema))
    return err
}

func (c *CreateTableCommand) Rollback(ctx context.Context) error {
    _, err := c.db.ExecContext(ctx, fmt.Sprintf("DROP TABLE IF EXISTS %s", c.tableName))
    return err
}

func (c *CreateTableCommand) Name() string { return "CreateTable:" + c.tableName }

// 命令执行器：支持顺序执行 + 失败回滚
type Executor struct{ commands []Command }

func (e *Executor) Run(ctx context.Context) error {
    executed := make([]Command, 0, len(e.commands))
    for _, cmd := range e.commands {
        if err := cmd.Execute(ctx); err != nil {
            // 逆序回滚已执行的命令
            for i := len(executed) - 1; i >= 0; i-- {
                executed[i].Rollback(ctx)
            }
            return fmt.Errorf("command %s failed: %w", cmd.Name(), err)
        }
        executed = append(executed, cmd)
    }
    return nil
}
```

---

### 15. 迭代器模式 Iterator

#### 意图

提供一种方法顺序访问一个聚合对象中各个元素，而又不暴露该对象的内部表示。

---

#### 🖥️ 前端视角

JavaScript 的 `Symbol.iterator` 协议和 `for...of` 语法就是迭代器模式的语言级实现。自定义迭代器可以让任意数据结构支持惰性遍历。

```typescript
// 分页数据的惰性迭代器
async function* paginate<T>(
  fetcher: (page: number, size: number) => Promise<{ data: T[]; hasMore: boolean }>,
  pageSize = 20
): AsyncGenerator<T> {
  let page = 0;
  while (true) {
    const { data, hasMore } = await fetcher(page++, pageSize);
    for (const item of data) yield item;
    if (!hasMore) break;
  }
}

// 使用：调用方无需关心分页逻辑
for await (const user of paginate(fetchUsers)) {
  processUser(user);
}
```

---

#### 🗄️ 后端视角

后端迭代器用于**流式处理大数据集**，避免一次性加载全部数据到内存。数据库游标、消息队列消费者都是迭代器的体现。

```go
// Go: 数据库游标迭代器
type UserIterator struct {
    db     *sql.DB
    rows   *sql.Rows
    current *User
    err     error
}

func NewUserIterator(db *sql.DB, filter string) *UserIterator {
    rows, err := db.Query("SELECT id, name, email FROM users WHERE " + filter)
    return &UserIterator{db: db, rows: rows, err: err}
}

func (it *UserIterator) Next() bool {
    if it.err != nil || !it.rows.Next() { return false }
    it.current = &User{}
    it.err = it.rows.Scan(&it.current.ID, &it.current.Name, &it.current.Email)
    return it.err == nil
}

func (it *UserIterator) Value() *User { return it.current }
func (it *UserIterator) Close()       { it.rows.Close() }

// 使用：流式处理，内存只保留一行
iter := NewUserIterator(db, "status='active'")
defer iter.Close()
for iter.Next() {
    sendEmail(iter.Value())
}
```

---

### 16. 中介者模式 Mediator

#### 意图

用一个中介对象来封装一系列的对象交互，使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

---

#### 🖥️ 前端视角

前端中介者模式最典型的体现是**EventBus** 和 **Vuex/Redux 的 Store**。组件之间不直接通信，而是通过中介者（Store 或 EventBus）传递消息。

```typescript
// EventBus：简单中介者
type Listener<T = any> = (data: T) => void;

class EventBus {
  private listeners: Map<string, Set<Listener>> = new Map();

  on<T>(event: string, listener: Listener<T>) {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener as Listener);
  }

  off(event: string, listener: Listener) {
    this.listeners.get(event)?.delete(listener);
  }

  emit<T>(event: string, data?: T) {
    this.listeners.get(event)?.forEach(l => l(data));
  }
}

export const bus = new EventBus();

// 组件 A 发布（不知道谁在监听）
bus.emit('cart:updated', { count: 3 });

// 组件 B 订阅（不知道谁在发布）
bus.on('cart:updated', ({ count }) => updateBadge(count));
```

---

#### 🗄️ 后端视角

微服务架构中，**消息中间件**（Kafka、RabbitMQ）就是中介者模式的工程级实现。服务间不直接调用，通过消息总线解耦。

```go
// Go: 内存级事件总线（单体服务内部）
type EventHandler func(event Event)

type EventBus struct {
    mu       sync.RWMutex
    handlers map[string][]EventHandler
}

func (b *EventBus) Subscribe(eventType string, handler EventHandler) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.handlers[eventType] = append(b.handlers[eventType], handler)
}

func (b *EventBus) Publish(event Event) {
    b.mu.RLock()
    handlers := b.handlers[event.Type]
    b.mu.RUnlock()

    for _, h := range handlers {
        go h(event)   // 异步处理，不阻塞发布者
    }
}

// 订单服务发布事件
bus.Publish(Event{Type: "order.paid", Payload: order})

// 库存服务、通知服务分别订阅（彼此不感知）
bus.Subscribe("order.paid", inventoryService.OnOrderPaid)
bus.Subscribe("order.paid", notificationService.OnOrderPaid)
bus.Subscribe("order.paid", loyaltyService.OnOrderPaid)
```

---

### 17. 备忘录模式 Memento

#### 意图

在不破坏封装性的前提下，捕获一个对象的**内部状态**，并在该对象之外保存这个状态。这样以后就可以将该对象恢复到原先保存的状态。

---

#### 🖥️ 前端视角

前端备忘录模式就是**状态快照与时间旅行调试**。Redux DevTools 的时间旅行功能，本质上就是将每次 dispatch 后的 state 存储为备忘录，可以任意恢复。

```typescript
// 简易时间旅行状态管理
class StateMachine<S> {
  private snapshots: S[] = [];
  private cursor = -1;

  constructor(private state: S) {
    this.save();
  }

  getState(): S { return this.state; }

  update(updater: (s: S) => S) {
    this.state = updater(structuredClone(this.state));   // 不可变更新
    // 清除 cursor 之后的历史
    this.snapshots = this.snapshots.slice(0, this.cursor + 1);
    this.snapshots.push(structuredClone(this.state));
    this.cursor++;
  }

  undo(): boolean {
    if (this.cursor <= 0) return false;
    this.state = structuredClone(this.snapshots[--this.cursor]);
    return true;
  }

  redo(): boolean {
    if (this.cursor >= this.snapshots.length - 1) return false;
    this.state = structuredClone(this.snapshots[++this.cursor]);
    return true;
  }

  private save() {
    this.snapshots.push(structuredClone(this.state));
    this.cursor++;
  }
}
```

---

#### 🗄️ 后端视角

后端备忘录对应**数据库事务保存点**、**工作流状态快照**、**游戏存档系统**。

```go
// Go: 工作流状态快照
type WorkflowState struct {
    Step     string
    Data     map[string]interface{}
    Metadata map[string]string
}

type WorkflowEngine struct {
    current   WorkflowState
    snapshots []WorkflowState
    store     SnapshotStore
}

func (w *WorkflowEngine) Snapshot() error {
    snapshot := deepCopy(w.current)
    w.snapshots = append(w.snapshots, snapshot)
    return w.store.Save(snapshot)   // 持久化，支持重启恢复
}

func (w *WorkflowEngine) Restore(snapshotID string) error {
    snapshot, err := w.store.Load(snapshotID)
    if err != nil { return err }
    w.current = snapshot
    return nil
}
```

---

### 18. 观察者模式 Observer

#### 意图

定义对象间的一种**一对多**的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

---

#### 🖥️ 前端视角

观察者模式是前端响应式系统的基石。Vue 的响应式数据、React 的 `useEffect` 依赖追踪、RxJS 的 Observable，都是观察者模式的实现。

```typescript
// 自实现响应式（Vue 3 reactivity 简化版）
type Effect = () => void;
let activeEffect: Effect | null = null;

class Reactive<T extends object> {
  private deps = new Map<keyof T, Set<Effect>>();

  constructor(private data: T) {
    return new Proxy(data, {
      get: (target, key: keyof T) => {
        this.track(key);
        return target[key];
      },
      set: (target, key: keyof T, value) => {
        target[key] = value;
        this.trigger(key);
        return true;
      }
    }) as any;
  }

  private track(key: keyof T) {
    if (!activeEffect) return;
    if (!this.deps.has(key)) this.deps.set(key, new Set());
    this.deps.get(key)!.add(activeEffect);
  }

  private trigger(key: keyof T) {
    this.deps.get(key)?.forEach(effect => effect());
  }
}

function watchEffect(effect: Effect) {
  activeEffect = effect;
  effect();    // 首次运行，触发依赖收集
  activeEffect = null;
}

// 使用
const state = new Reactive({ count: 0, name: 'Alice' });

watchEffect(() => {
  console.log(`Count is: ${state.count}`);    // 自动追踪 count 依赖
});

state.count++;   // 自动触发上面的 effect
```

---

#### 🗄️ 后端视角

后端观察者对应**领域事件（Domain Events）**、**数据库变更监听（CDC）**、**WebSocket 推送**。

```java
// Java: 领域事件 + 观察者
public class DomainEventPublisher {
    private final Map<Class<?>, List<DomainEventHandler<?>>> handlers = new ConcurrentHashMap<>();

    @SuppressWarnings("unchecked")
    public <T extends DomainEvent> void subscribe(Class<T> type, DomainEventHandler<T> handler) {
        handlers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>()).add(handler);
    }

    @SuppressWarnings("unchecked")
    public <T extends DomainEvent> void publish(T event) {
        List<DomainEventHandler<?>> eventHandlers = handlers.getOrDefault(event.getClass(), emptyList());
        eventHandlers.forEach(h -> ((DomainEventHandler<T>) h).handle(event));
    }
}

// 订单支付成功事件
public class OrderPaidEvent extends DomainEvent {
    public final String orderId;
    public final BigDecimal amount;
}

// 多个观察者独立响应
publisher.subscribe(OrderPaidEvent.class, e -> inventoryService.deduct(e.orderId));
publisher.subscribe(OrderPaidEvent.class, e -> emailService.sendConfirmation(e.orderId));
publisher.subscribe(OrderPaidEvent.class, e -> analyticsService.trackRevenue(e.amount));

// 触发：业务逻辑与副作用解耦
publisher.publish(new OrderPaidEvent(orderId, amount));
```

---

### 19. 状态模式 State

#### 意图

允许一个对象在其**内部状态改变**时改变它的行为，对象看起来似乎修改了它的类。

---

#### 🖥️ 前端视角

前端状态模式适用于**UI 复杂状态机**，如视频播放器（playing/paused/buffering/ended）、上传组件（idle/uploading/success/error）。

```typescript
// 上传组件状态机
interface UploadState {
  onUpload(ctx: Uploader): void;
  onProgress(ctx: Uploader, progress: number): void;
  onSuccess(ctx: Uploader): void;
  onError(ctx: Uploader, err: Error): void;
  getLabel(): string;
}

class IdleState implements UploadState {
  onUpload(ctx: Uploader) { ctx.setState(new UploadingState()); ctx.startUpload(); }
  onProgress() {}
  onSuccess()  {}
  onError()    {}
  getLabel() { return '点击上传'; }
}

class UploadingState implements UploadState {
  onUpload()  {}   // 上传中不允许再次点击
  onProgress(ctx: Uploader, progress: number) { ctx.setProgress(progress); }
  onSuccess(ctx: Uploader) { ctx.setState(new SuccessState()); }
  onError(ctx: Uploader, err: Error) { ctx.setState(new ErrorState(err)); }
  getLabel() { return '上传中...'; }
}

class SuccessState implements UploadState {
  onUpload(ctx: Uploader) { ctx.setState(new IdleState()); }   // 允许重新上传
  onProgress() {} onSuccess() {} onError() {}
  getLabel() { return '✅ 上传成功，点击替换'; }
}

class ErrorState implements UploadState {
  constructor(private err: Error) {}
  onUpload(ctx: Uploader) { ctx.setState(new UploadingState()); ctx.startUpload(); }  // 重试
  onProgress() {} onSuccess() {} onError() {}
  getLabel() { return `❌ ${this.err.message}，点击重试`; }
}
```

---

#### 🗄️ 后端视角

后端状态模式用于**订单状态机**、**工作流引擎**、**协议解析**。

```go
// Go: 订单状态机
type OrderState interface {
    Pay(o *Order) error
    Ship(o *Order) error
    Deliver(o *Order) error
    Cancel(o *Order) error
    String() string
}

type PendingState   struct{}
type PaidState      struct{}
type ShippedState   struct{}
type DeliveredState struct{}
type CancelledState struct{}

func (s *PendingState) Pay(o *Order) error {
    o.SetState(&PaidState{})
    o.PaidAt = time.Now()
    return nil
}
func (s *PendingState) Ship(o *Order) error   { return errors.New("未支付，不能发货") }
func (s *PendingState) Cancel(o *Order) error { o.SetState(&CancelledState{}); return nil }

func (s *PaidState) Pay(o *Order) error    { return errors.New("已支付") }
func (s *PaidState) Ship(o *Order) error   { o.SetState(&ShippedState{}); return nil }
func (s *PaidState) Cancel(o *Order) error { o.SetState(&CancelledState{}); return refund(o) }

// 订单对象委托给当前状态处理
type Order struct {
    state OrderState
    // ...
}
func (o *Order) Pay()    error { return o.state.Pay(o) }
func (o *Order) Ship()   error { return o.state.Ship(o) }
func (o *Order) Cancel() error { return o.state.Cancel(o) }
func (o *Order) SetState(s OrderState) { o.state = s }
```

---

### 20. 策略模式 Strategy

#### 意图

定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。

---

#### 🖥️ 前端视角

策略模式是消除前端 `if-else` 的最有效工具。表单验证规则、排序算法、图表数据格式化都适用。

```typescript
// 价格计算策略
interface PricingStrategy {
  calculate(basePrice: number, quantity: number): number;
  label: string;
}

const strategies: Record<string, PricingStrategy> = {
  standard: {
    label: '标准价',
    calculate: (price, qty) => price * qty,
  },
  bulk: {
    label: '批量折扣',
    calculate: (price, qty) => {
      if (qty >= 100) return price * qty * 0.7;
      if (qty >= 50)  return price * qty * 0.85;
      return price * qty;
    },
  },
  member: {
    label: '会员价',
    calculate: (price, qty) => price * qty * 0.9,
  },
  flash: {
    label: '限时特价',
    calculate: (price, qty) => Math.floor(price * 0.5) * qty,
  },
};

// Context：不知道具体策略
function calculateTotal(strategyKey: string, price: number, qty: number) {
  const strategy = strategies[strategyKey];
  if (!strategy) throw new Error(`Unknown pricing strategy: ${strategyKey}`);
  return strategy.calculate(price, qty);
}

// 扩展新策略只需在 strategies 对象里添加一个 entry，零修改已有代码
```

---

#### 🗄️ 后端视角

后端策略模式用于**算法热切换**（排序、推荐、风控规则）、**多渠道路由**、**A/B 测试**。

```go
// Go: 文件压缩策略

type CompressStrategy interface {
    Compress(data []byte) ([]byte, error)
    Extension() string
}

type GzipStrategy  struct{}
type ZstdStrategy  struct{}
type LZ4Strategy   struct{}

func (g *GzipStrategy) Compress(data []byte) ([]byte, error) { /* gzip */ return nil, nil }
func (g *GzipStrategy) Extension() string { return ".gz" }

func (z *ZstdStrategy) Compress(data []byte) ([]byte, error) { /* zstd */ return nil, nil }
func (z *ZstdStrategy) Extension() string { return ".zst" }

// Context：策略在运行时注入
type FileArchiver struct {
    strategy CompressStrategy
}

func (a *FileArchiver) SetStrategy(s CompressStrategy) { a.strategy = s }

func (a *FileArchiver) Archive(filename string, data []byte) error {
    compressed, err := a.strategy.Compress(data)
    if err != nil { return err }
    return os.WriteFile(filename+a.strategy.Extension(), compressed, 0644)
}

// 根据配置或文件大小动态选择策略
archiver := &FileArchiver{}
if fileSize > 100*MB {
    archiver.SetStrategy(&ZstdStrategy{})   // 大文件用 zstd（速度快）
} else {
    archiver.SetStrategy(&GzipStrategy{})   // 小文件用 gzip（兼容性好）
}
```

---

### 21. 模板方法模式 Template Method

#### 意图

定义一个操作中的算法的**骨架**，而将一些步骤延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

---

#### 🖥️ 前端视角

前端模板方法常见于**页面生命周期**和**通用列表页**。React 类组件的生命周期（constructor → render → componentDidMount → …）本身就是模板方法。

```typescript
// 通用列表页模板（React Hooks 实现）
function useListPage<T, F>(
  fetcher: (filters: F, page: number) => Promise<{ data: T[]; total: number }>,
  defaultFilters: F
) {
  const [data, setData] = useState<T[]>([]);
  const [filters, setFilters] = useState(defaultFilters);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [total, setTotal] = useState(0);

  // 模板：固定的加载流程骨架
  useEffect(() => {
    setLoading(true);
    fetcher(filters, page)               // 步骤1：可替换的数据获取
      .then(({ data, total }) => {
        setData(data);                   // 步骤2：固定的状态更新
        setTotal(total);
      })
      .finally(() => setLoading(false)); // 步骤3：固定的收尾
  }, [filters, page]);

  return { data, loading, total, filters, setFilters, page, setPage };
}

// 用户列表（只需提供差异化的 fetcher）
const UserListPage = () => {
  const { data, loading } = useListPage(
    (f, p) => api.get(`/users?page=${p}&role=${f.role}`),
    { role: 'all' }
  );
  // ...
};
```

---

#### 🗄️ 后端视角

后端模板方法用于**数据导出管道**、**ETL 流程**、**报表生成**。

```go
// Go: ETL 管道模板
type ETLPipeline interface {
    Extract(ctx context.Context) (<-chan Record, error)   // 抽象：各自实现
    Transform(record Record) (Record, error)               // 抽象：各自实现
    Load(ctx context.Context, record Record) error         // 抽象：各自实现
}

// 模板方法：固定的执行骨架
func RunPipeline(ctx context.Context, pipeline ETLPipeline) error {
    records, err := pipeline.Extract(ctx)   // 步骤1
    if err != nil { return err }

    var wg sync.WaitGroup
    errCh := make(chan error, 1)

    for record := range records {
        wg.Add(1)
        go func(r Record) {
            defer wg.Done()
            transformed, err := pipeline.Transform(r)    // 步骤2
            if err != nil { select { case errCh <- err: default: }; return }
            if err := pipeline.Load(ctx, transformed); err != nil {   // 步骤3
                select { case errCh <- err: default: }
            }
        }(record)
    }
    wg.Wait()
    select { case err := <-errCh: return err; default: return nil }
}

// 具体实现只需关注差异化逻辑
type MySQLToElasticPipeline struct { /* ... */ }
func (p *MySQLToElasticPipeline) Extract(ctx context.Context) (<-chan Record, error) { /* MySQL 读 */ }
func (p *MySQLToElasticPipeline) Transform(r Record) (Record, error) { /* 字段映射 */ }
func (p *MySQLToElasticPipeline) Load(ctx context.Context, r Record) error { /* ES 写 */ }
```

---

### 22. 访问者模式 Visitor

#### 意图

表示一个作用于某对象结构中各元素的操作，它使你可以在不改变各元素类的前提下定义作用于这些元素的新操作。

---

#### 🖥️ 前端视角

前端访问者模式用于**AST 遍历**（Babel 插件、ESLint 规则）和**DOM 树分析**。Babel 的 `visitor` 对象就是标准的访问者实现。

```typescript
// Babel 插件风格的 AST 访问者
interface ASTNode { type: string; }
interface CallExpression extends ASTNode { callee: ASTNode; arguments: ASTNode[]; }
interface Identifier extends ASTNode { name: string; }

interface Visitor {
  CallExpression?(node: CallExpression): void;
  Identifier?(node: Identifier): void;
  // ...其他节点类型
}

// 遍历器（Element 角色）
function traverse(node: ASTNode, visitor: Visitor) {
  const handler = visitor[node.type as keyof Visitor] as any;
  if (handler) handler(node);
  // 递归遍历子节点...
}

// 具体访问者：收集所有 console.log 调用
const consoleLogVisitor: Visitor = {
  CallExpression(node) {
    if (
      node.callee.type === 'MemberExpression' &&
      (node.callee as any).object.name === 'console'
    ) {
      console.warn('Found console call, consider removing for production');
    }
  }
};

traverse(ast, consoleLogVisitor);
```

---

#### 🗄️ 后端视角

后端访问者用于**报表引擎**（对同一数据结构输出不同格式）、**编译器/解释器**。

```java
// Java: 费用报表访问者（输出不同格式）
public interface ExpenseVisitor {
    void visit(TravelExpense e);
    void visit(MealExpense e);
    void visit(EquipmentExpense e);
}

// JSON 导出访问者
public class JSONExportVisitor implements ExpenseVisitor {
    private JsonArray array = new JsonArray();

    @Override public void visit(TravelExpense e) {
        array.add(Map.of("type","travel","amount",e.getAmount(),"date",e.getDate()));
    }
    @Override public void visit(MealExpense e) { /* ... */ }
    @Override public void visit(EquipmentExpense e) { /* ... */ }

    public String getResult() { return array.toJsonString(); }
}

// PDF 导出访问者（完全不同的输出逻辑，无需修改 Expense 类）
public class PDFExportVisitor implements ExpenseVisitor {
    private PdfDocument doc = new PdfDocument();
    @Override public void visit(TravelExpense e) { doc.addRow("交通", e.getAmount()); }
    // ...
}

// 元素
public interface Expense {
    void accept(ExpenseVisitor visitor);
}
public class TravelExpense implements Expense {
    @Override public void accept(ExpenseVisitor v) { v.visit(this); }
}
```

---

### 23. 解释器模式 Interpreter

#### 意图

给定一个语言，定义它的文法的一种表示，并定义一个解释器，该解释器使用该表示来解释语言中的句子。

---

#### 🖥️ 前端视角

前端解释器用于**模板引擎**、**表达式求值**、**搜索 DSL**（如 `status:active AND role:admin`）。

```typescript
// 简单搜索 DSL 解释器
// 语法：field:value AND/OR field:value

type Expression = { field: string; value: string } | { op: 'AND' | 'OR'; left: Expression; right: Expression };

function interpret(expr: Expression, record: Record<string, string>): boolean {
  if ('field' in expr) {
    return record[expr.field]?.includes(expr.value) ?? false;
  }
  if (expr.op === 'AND') return interpret(expr.left, record) && interpret(expr.right, record);
  if (expr.op === 'OR')  return interpret(expr.left, record) || interpret(expr.right, record);
  return false;
}

// 解析：简化示意
const expr: Expression = {
  op: 'AND',
  left:  { field: 'status', value: 'active' },
  right: { field: 'role',   value: 'admin'  },
};

const users = allUsers.filter(u => interpret(expr, u));
```

---

#### 🗄️ 后端视角

后端解释器用于**规则引擎**、**权限表达式**、**配置 DSL**、**SQL 方言解析**。

```go
// Go: 简单条件规则解释器
type Expr interface {
    Eval(ctx map[string]interface{}) bool
}

type EqExpr  struct{ Field string; Value interface{} }
type AndExpr struct{ Left, Right Expr }
type OrExpr  struct{ Left, Right Expr }
type NotExpr struct{ Inner Expr }

func (e *EqExpr)  Eval(ctx map[string]interface{}) bool { return ctx[e.Field] == e.Value }
func (e *AndExpr) Eval(ctx map[string]interface{}) bool { return e.Left.Eval(ctx) && e.Right.Eval(ctx) }
func (e *OrExpr)  Eval(ctx map[string]interface{}) bool { return e.Left.Eval(ctx) || e.Right.Eval(ctx) }
func (e *NotExpr) Eval(ctx map[string]interface{}) bool { return !e.Inner.Eval(ctx) }

// 规则：(country == "CN" OR country == "HK") AND age >= 18
rule := &AndExpr{
    Left: &OrExpr{
        Left:  &EqExpr{Field: "country", Value: "CN"},
        Right: &EqExpr{Field: "country", Value: "HK"},
    },
    Right: &EqExpr{Field: "adult", Value: true},
}

if rule.Eval(map[string]interface{}{"country": "CN", "adult": true}) {
    // 符合规则
}
```

---

## 四、实战重构案例

### 阶段一：入门级重构

**案例：支付方式选择（策略模式消灭 if-else）**

#### 重构前代码

```typescript
// 问题：每增加一种支付方式，就要修改这个函数
async function processPayment(amount: number, method: string, userInfo: any) {
  let result: any;

  if (method === 'alipay') {
    const sign = md5(userInfo.uid + amount + 'alipay_secret');
    result = await fetch('https://alipay.api/pay', {
      method: 'POST',
      body: JSON.stringify({ uid: userInfo.uid, amount, sign })
    }).then(r => r.json());
    if (result.code !== 0) throw new Error('支付宝支付失败: ' + result.msg);
    return { txId: result.data.trade_no, method: 'alipay' };

  } else if (method === 'wechat') {
    const openId = await fetchWechatOpenId(userInfo.token);
    result = await fetch('https://wechat.api/pay', {
      method: 'POST',
      body: JSON.stringify({ openId, totalFee: amount * 100 })
    }).then(r => r.json());
    if (result.errcode !== undefined) throw new Error('微信支付失败');
    return { txId: result.prepayId, method: 'wechat' };

  } else if (method === 'stripe') {
    result = await stripe.paymentIntents.create({ amount: amount * 100, currency: 'usd' });
    if (result.status !== 'succeeded') throw new Error('Stripe 支付失败');
    return { txId: result.id, method: 'stripe' };

  } else if (method === 'paypal') {
    // ... 又是一段硬编码逻辑
  } else {
    throw new Error('不支持的支付方式: ' + method);
  }
}
```

#### 问题分析

1. **违反开闭原则**：新增支付方式必须修改核心函数，风险高。
2. **职责混乱**：签名、请求、错误处理、结果映射全部混在一起。
3. **可测试性差**：无法单独测试某个支付方式的逻辑。
4. **长函数**：随着支付方式增加，函数会无限膨胀。

#### 重构后代码

```typescript
// 1. 定义统一的策略接口
interface PaymentStrategy {
  charge(amount: number, userInfo: UserInfo): Promise<PaymentResult>;
}

interface PaymentResult { txId: string; method: string; }

// 2. 各策略独立实现（可分文件、可独立测试）
class AlipayStrategy implements PaymentStrategy {
  async charge(amount: number, user: UserInfo): Promise<PaymentResult> {
    const sign = md5(user.uid + amount + 'alipay_secret');
    const res  = await alipayClient.pay({ uid: user.uid, amount, sign });
    if (res.code !== 0) throw new PaymentError('alipay', res.msg);
    return { txId: res.data.trade_no, method: 'alipay' };
  }
}

class WechatPayStrategy implements PaymentStrategy {
  async charge(amount: number, user: UserInfo): Promise<PaymentResult> {
    const openId = await fetchWechatOpenId(user.token);
    const res    = await wechatClient.pay({ openId, totalFee: amount * 100 });
    if (res.errcode !== undefined) throw new PaymentError('wechat', res.errmsg);
    return { txId: res.prepayId, method: 'wechat' };
  }
}

class StripeStrategy implements PaymentStrategy {
  async charge(amount: number, _user: UserInfo): Promise<PaymentResult> {
    const intent = await stripe.paymentIntents.create({ amount: amount * 100, currency: 'usd' });
    if (intent.status !== 'succeeded') throw new PaymentError('stripe', 'Payment failed');
    return { txId: intent.id, method: 'stripe' };
  }
}

// 3. 注册表：新增支付方式只需在此添加，零修改已有代码
const paymentStrategies: Record<string, PaymentStrategy> = {
  alipay:  new AlipayStrategy(),
  wechat:  new WechatPayStrategy(),
  stripe:  new StripeStrategy(),
};

// 4. Context：极简，无任何业务细节
async function processPayment(amount: number, method: string, userInfo: UserInfo) {
  const strategy = paymentStrategies[method];
  if (!strategy) throw new Error(`Unsupported payment method: ${method}`);
  return strategy.charge(amount, userInfo);
}
```

---

### 阶段二：进阶级重构

**案例：组件间通信耦合（观察者模式解耦）**

#### 重构前代码

```typescript
// 问题：Header、Cart、Notification 三个组件相互直接引用，形成蜘蛛网
class Header {
  private cart: Cart;         // 直接依赖
  private notification: Notification;  // 直接依赖

  onUserLogin(user: User) {
    this.cart.loadUserCart(user.id);          // 直接调用兄弟组件
    this.notification.showWelcome(user.name); // 直接调用兄弟组件
    this.updateAvatar(user.avatarUrl);
  }
}

class Cart {
  private header: Header;     // 反向依赖，形成循环
  private notification: Notification;

  onItemAdded(item: Item) {
    this.header.updateCartBadge(this.items.length);  // 直接调用
    this.notification.showAddedToast(item.name);      // 直接调用
  }
}
// 任何一个组件变更接口，其他所有依赖它的组件都要同步修改
```

#### 问题分析

1. **循环依赖**：Header → Cart，Cart → Header，无法独立实例化。
2. **高耦合**：任一组件改动接口，级联修改所有依赖方。
3. **难以扩展**：新增 Analytics 组件监听登录事件，需要修改 Header。
4. **测试困难**：测试 Header 必须 mock Cart 和 Notification。

#### 重构后代码

```typescript
// 1. 引入 EventBus（中介者/观察者）
const bus = new EventBus();

// 2. 定义事件类型（类型安全）
interface AppEvents {
  'user:login':    { user: User };
  'cart:changed':  { count: number; lastItem?: Item };
  'order:placed':  { orderId: string; amount: number };
}

// 3. 各组件只依赖 EventBus，彼此不感知
class Header {
  constructor() {
    // 只订阅自己关心的事件
    bus.on('user:login',   ({ user }) => this.updateAvatar(user.avatarUrl));
    bus.on('cart:changed', ({ count }) => this.updateCartBadge(count));
  }

  private onLoginButtonClick() {
    const user = await authService.login();
    bus.emit('user:login', { user });   // 发布事件，不关心谁处理
  }
}

class Cart {
  constructor() {
    bus.on('user:login', ({ user }) => this.loadUserCart(user.id));
  }

  addItem(item: Item) {
    this.items.push(item);
    bus.emit('cart:changed', { count: this.items.length, lastItem: item });  // 发布，不关心谁处理
  }
}

class Notification {
  constructor() {
    bus.on('user:login',   ({ user }) => this.showWelcome(user.name));
    bus.on('cart:changed', ({ lastItem }) => lastItem && this.showAddedToast(lastItem.name));
    bus.on('order:placed', ({ orderId }) => this.showOrderSuccess(orderId));
  }
}

// 4. 新增 Analytics 组件：零修改已有组件
class Analytics {
  constructor() {
    bus.on('user:login',   ({ user }) => this.track('login', { userId: user.id }));
    bus.on('order:placed', ({ amount }) => this.track('purchase', { revenue: amount }));
  }
}
```

---

### 阶段三：架构级重构

**案例：单体服务拆分为微服务（门面模式 + 适配器模式）**

#### 重构前代码

```go
// 单体：所有逻辑在一个服务内，直接函数调用
func (s *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    json.NewDecoder(r.Body).Decode(&req)

    // 直接调用内部函数（紧耦合）
    user, _ := s.userRepo.FindByID(req.UserID)
    if user == nil { http.Error(w, "user not found", 404); return }

    products, _ := s.productRepo.FindByIDs(req.ProductIDs)
    for _, p := range products {
        if p.Stock < req.Quantities[p.ID] {
            http.Error(w, "insufficient stock", 400); return
        }
    }

    for _, p := range products {
        s.productRepo.DeductStock(p.ID, req.Quantities[p.ID])  // 直接操作 DB
    }

    order := s.orderRepo.Create(Order{ UserID: req.UserID, Products: products })

    s.emailService.SendOrderConfirmation(user.Email, order)  // 直接调用邮件服务
    s.smsService.SendOrderSMS(user.Phone, order.ID)          // 直接调用短信

    json.NewEncoder(w).Encode(order)
}
```

#### 问题分析

1. **无法独立扩展**：用户服务和商品服务不能独立部署扩容。
2. **技术栈锁定**：所有功能必须用同一语言/框架。
3. **故障传导**：库存服务故障导致整个下单链路崩溃。
4. **团队协作难**：多团队同时修改同一代码库，冲突频繁。

#### 重构后代码（微服务架构）

```go
// 1. 定义服务接口（防腐层抽象）
type UserService interface {
    GetUser(ctx context.Context, userID string) (*User, error)
}

type InventoryService interface {
    CheckAndReserve(ctx context.Context, items []ReserveItem) (*Reservation, error)
    Confirm(ctx context.Context, reservationID string) error
    Release(ctx context.Context, reservationID string) error
}

type NotificationService interface {
    SendOrderConfirmation(ctx context.Context, userID, orderID string) error
}

// 2. 适配器：封装 HTTP/gRPC 调用细节（防腐层实现）
type UserServiceAdapter struct {
    baseURL    string
    httpClient *http.Client
    breaker    *gobreaker.CircuitBreaker   // 熔断器
}

func (a *UserServiceAdapter) GetUser(ctx context.Context, userID string) (*User, error) {
    result, err := a.breaker.Execute(func() (interface{}, error) {
        req, _ := http.NewRequestWithContext(ctx, "GET",
            fmt.Sprintf("%s/users/%s", a.baseURL, userID), nil)
        resp, err := a.httpClient.Do(req)
        if err != nil { return nil, err }
        defer resp.Body.Close()

        if resp.StatusCode == 404 { return nil, ErrUserNotFound }
        if resp.StatusCode != 200 { return nil, fmt.Errorf("user service error: %d", resp.StatusCode) }

        var user User
        json.NewDecoder(resp.Body).Decode(&user)
        return &user, nil
    })
    if err != nil { return nil, err }
    return result.(*User), nil
}

// 3. 门面（BFF/API Gateway 层）：聚合多个微服务
type OrderFacade struct {
    userSvc      UserService
    inventorySvc InventoryService
    orderRepo    OrderRepository
    notifySvc    NotificationService
}

func (f *OrderFacade) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // 1. 验证用户（调用用户微服务）
    user, err := f.userSvc.GetUser(ctx, req.UserID)
    if err != nil { return nil, fmt.Errorf("validate user: %w", err) }

    // 2. 预留库存（Saga 模式：可补偿事务）
    reservation, err := f.inventorySvc.CheckAndReserve(ctx, toReserveItems(req))
    if err != nil { return nil, fmt.Errorf("reserve inventory: %w", err) }

    // 3. 创建订单（本地事务）
    order, err := f.orderRepo.Create(ctx, Order{
        UserID:        user.ID,
        ReservationID: reservation.ID,
        Items:         req.Items,
        Status:        OrderStatusPending,
    })
    if err != nil {
        f.inventorySvc.Release(ctx, reservation.ID)   // Saga 补偿
        return nil, fmt.Errorf("create order: %w", err)
    }

    // 4. 确认库存扣减
    if err := f.inventorySvc.Confirm(ctx, reservation.ID); err != nil {
        // 补偿：取消订单
        f.orderRepo.Cancel(ctx, order.ID)
        return nil, fmt.Errorf("confirm inventory: %w", err)
    }

    // 5. 异步通知（非关键路径，失败不回滚）
    go func() {
        notifyCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        f.notifySvc.SendOrderConfirmation(notifyCtx, user.ID, order.ID)
    }()

    return order, nil
}

// 4. HTTP Handler：极简，只做协议转换
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", 400); return
    }

    order, err := h.facade.CreateOrder(r.Context(), req)
    if err != nil {
        handleError(w, err); return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}
```

---

## 五、模式选型速查表

| 场景特征 | 推荐模式 | 避坑 |
|---------|---------|------|
| 大量 if-else 按类型分支 | 策略模式 | 分支 < 3 时不必引入 |
| 创建逻辑复杂、步骤多 | 建造者模式 | 参数 < 4 时用普通构造器即可 |
| 需要创建一族相关对象 | 抽象工厂 | 产品族固定时才值得引入 |
| 全局共享资源（连接池） | 单例模式 | SSR 场景注意请求隔离 |
| 旧接口 → 新接口适配 | 适配器模式 | 不要用来绕过设计缺陷 |
| 动态添加功能，不想继承 | 装饰器模式 | 装饰层数 > 5 时考虑重构 |
| 子系统复杂，需要简化调用 | 门面模式 | 不要让门面变成上帝类 |
| 一对多通知，松耦合 | 观察者模式 | 注意内存泄漏（及时 off）|
| 组件间通信，避免循环依赖 | 中介者模式 | 中介者本身不能变成上帝 |
| 对象行为随内部状态变化 | 状态模式 | 状态 < 3 时 if-else 即可 |
| 对不同对象执行不同操作 | 访问者模式 | 元素类型频繁变化时慎用 |
| 操作需要撤销/重做 | 命令模式 | 快照存储注意内存占用 |
| 大量对象，内存压力大 | 享元模式 | 外部状态复杂时不值得 |
| 流程固定，步骤可替换 | 模板方法 | 继承层次不超过 2 层 |
| 控制对象访问（缓存/权限）| 代理模式 | 不要过度代理（影响调试）|

---

> **最后一句话**：设计模式不是银弹，是**词汇表**。掌握它的目的是让团队沟通更高效——当你说"这里用策略模式"，所有人都知道意味着什么，而不必阅读整段代码。先写能跑的代码，再用模式重构；而不是一开始就把所有模式塞进去。
