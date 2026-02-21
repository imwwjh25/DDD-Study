## 一、核心架构思想（先搞懂底层设计哲学）

### 1. 第一性原则：依赖永远向内

图中蓝色箭头明确标注了 **Dependencies go inwards（依赖永远向内）**，这是整个架构的灵魂：

- 最内层的**业务核心**是整个系统的绝对中心，所有外层代码都必须依赖内层，内层绝对不依赖外层；
- 彻底反转了传统三层架构（Controller→Service→DAO，业务层依赖数据库）的依赖方向，实现了**业务逻辑和技术细节的完全解耦**；
- 业务核心不会被 Spring、MyBatis、MySQL、阿里云 SDK 等外部技术框架绑架，换数据库、换短信服务商、换接口协议，核心业务代码一行都不用改。

### 2. 核心概念：端口 + 适配器的解耦模型

六边形架构的本质，是用「端口」和「适配器」把系统拆成**不可变的业务核心**和**可变的外部交互**两部分：


|                 概念                 |                             定义                             |                           核心职责                           |
| :----------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|             端口（Port）             | 应用核心定义的**业务契约（接口）**，是核心与外界交互的唯一入口 / 出口，分为两类 |             定义「能做什么」，不关心「怎么实现」             |
|    主端口（Primary/Driving Port）    |     核心对外暴露的能力接口，比如「下单用例」「支付用例」     |                 给外部调用，驱动系统执行业务                 |
|   次端口（Secondary/Driven Port）    |      核心对外部的依赖接口，比如「订单存储」「短信发送」      |              核心需要外部提供的能力，被核心调用              |
|          适配器（Adapter）           |            端口的具体实现，是协议转换层，分为两类            |         实现「怎么做」，处理技术细节，不包含业务逻辑         |
| 主适配器（Primary/Driving Adapter）  |     主端口的调用方，把外部输入协议转换成核心能识别的调用     |    比如 HTTP Controller、CLI、RPC 消费者，是系统的输入层     |
| 次适配器（Secondary/Driven Adapter） |  次端口的实现方，把核心的调用转换成外部基础设施能识别的协议  | 比如数据库 DAO 实现、短信 SDK 封装、MQ 生产者，是系统的输出层 |

### 3. 分层逻辑：从内到外的四层结构（严格遵守依赖规则）

图中从内到外的同心圆，就是架构的分层，**内层对外层完全无感知，外层只能向内依赖**：

1. **Domain Layer（领域层，最核心）**：系统的心脏，所有业务规则、业务约束、业务语义都在这里，是绝对不可变的核心，不依赖任何外部框架和技术。
2. **Application Layer（应用层）**：用例编排层，不包含任何业务规则，只负责协调业务流程，相当于「业务流程的指挥官」，只依赖领域层。
3. **Ports 层（端口层，八边形边界）**：核心与外界的隔离墙，所有进出核心的请求必须通过端口，八边形的每一条边都是一组端口契约。
4. **Adapters 层（适配器层，最外层）**：分左右两侧，左侧是主适配器（系统输入），右侧是次适配器（系统输出），负责所有和外部系统的技术交互。

### 4. 融合的进阶架构思想

这张图不是纯六边形架构，还融合了 3 个企业级开发的最佳实践：

- **CQRS（命令查询分离）**：图中的 Command Query BUS，把写操作（Command，比如创建订单）和读操作（Query，比如查询订单）完全分离，写操作走完整的业务规则校验，读操作直接走查询优化，兼顾数据一致性和查询性能。
- **事件驱动架构**：图中的 Event BUS，用领域事件解耦业务流程，比如订单创建后，发短信、通知仓储、扣积分等后续操作，都通过事件异步执行，避免主流程臃肿，符合开闭原则。
- **DDD 领域建模**：图中的 Domain Model、Component（聚合），完全遵循 DDD 的设计思想，把业务语义落地到代码中，避免贫血模型。

------

## 二、逐模块拆解图中所有元素（从内到外）

### 1. 最核心：Application Core（应用核心）

这是整个系统唯一包含业务逻辑的部分，其他所有层都只做技术交互，不写业务代码。

#### （1）Domain Layer（领域层，最中心）

对应图中最内层的橙色区域，包含：

- **Domain Model（领域模型）**：聚合根、实体、值对象、Domain Primitive（领域原生体），是业务规则的载体，比如订单聚合根、商品聚合根，所有「订单状态不能随意修改」「库存不能为负」的核心业务规则，都必须封装在这里。
- **Domain Services（领域服务）**：跨多个聚合的业务逻辑，比如下单时同时校验商品、扣减库存、计算价格，跨了订单、商品、库存三个聚合，就放在领域服务中。
- 核心约束：**绝对不能依赖任何外部技术框架**，不能出现 Spring、MyBatis、MySQL 相关的代码，只写纯 Java 的业务逻辑。

#### （2）Application Layer（应用层）

对应图中中间的黄色区域，包含：

- **App Services（应用服务）/ C/Q Handlers（命令 / 查询处理器）**：用例的入口，负责编排业务流程，比如「下单用例」的步骤：校验参数→调用领域服务创建订单→保存订单→发布事件，只负责步骤编排，不写业务规则，所有业务规则都交给领域层处理。
- **Event Listeners（事件监听器）**：监听领域事件，做后续的流程编排，比如监听到订单创建事件，调用短信端口给用户发通知。
- 核心约束：**只能依赖领域层，不能依赖任何外部技术框架**，不能写 SQL、不能调用第三方 SDK，所有外部依赖都通过次端口接口注入。

#### （3）Ports（端口层，八边形红色边界）

对应图中红色八边形的边界，是核心与外界的唯一契约：

- 左侧的 Commands、Queries、Services：是**主端口**，应用核心对外暴露的能力，外部只能通过这些端口调用核心。
- 右侧的 Persistence、Search、Notifications、Event Listeners：是**次端口**，应用核心定义的依赖接口，需要外部适配器实现。

### 2. 左侧：Primary/Driving Adapters（主 / 驱动适配器）

对应图中左侧浅绿色区域，是**系统的所有输入源**，用户、外部系统通过这些适配器触发系统的业务行为。

- 图中包含的主适配器：Admin GUI/Controllers（管理后台）、API Controllers（OpenAPI / 前端接口）、Consumer GUI/Controllers（用户端界面）、Console Commands（CLI 命令行 / 定时任务）。
- 核心职责：**只做协议转换和基础参数校验，绝对不写业务逻辑**。比如 Controller 接收 HTTP 的 JSON 请求，把参数转换成 Command 对象，发送到 Command Bus，交给应用层处理，仅此而已。
- 图中注释说明：`Primary adapters wrap around a use case and adapt its input/output to a port, ie. HTTP/HTML, HTTP/JSON or CLI.` 主适配器的唯一作用，就是把外部的输入协议，适配成核心端口能识别的格式。

### 3. 右侧：Secondary/Driven Adapters（次 / 被驱动适配器）

对应图中右侧浅橙色区域，是**系统的所有输出源**，应用核心通过次端口调用这些适配器，和外部基础设施交互。

- 图中包含的次适配器：ORM Adapter（数据库）、Search Adapter（搜索引擎）、Email Adapter（邮件）、SMS Adapter（短信）、Message Queue Adapter（消息队列）、Event BUS Adapter（事件总线）。
- 核心职责：**实现次端口接口，只做技术适配，绝对不写业务逻辑**。比如 OrderRepository 的 MySQL 实现，把核心定义的`save(Order order)`方法，转换成 JDBC 的 insert 语句，业务规则已经在领域层保证了，适配器只做数据存储。
- 图中注释说明：`Secondary adapters wrap around a tool and adapt its input/output to a port, which fits the application core needs` 次适配器的唯一作用，就是把外部工具的能力，适配成核心定义的端口契约。

### 4. 中间件骨架：C/Q BUS、Event BUS

对应图中紫色的总线区域，是解耦适配器和应用层的中间层：

- **Command Query BUS**：路由命令和查询，Controller 不用直接调用 Handler，把 Command 发给 Bus，Bus 自动路由到对应的 Handler，方便统一处理事务、权限、日志、限流等切面逻辑。
- **Event BUS**：实现事件的发布和订阅，解耦主流程和副流程，比如下单主流程只需要创建订单，后续的短信通知、仓储通知都通过事件异步处理，提升系统吞吐量。

------

## 三、和传统三层架构的核心区别


|     维度     |                        传统三层架构                         |                      六边形架构（本图）                      |
| :----------: | :---------------------------------------------------------: | :----------------------------------------------------------: |
|   依赖方向   |     业务层（Service）依赖数据层（DAO），业务被技术绑架      | 业务核心不依赖任何外部，外部适配器依赖核心，业务与技术完全解耦 |
| 业务规则位置 | 分散在 Service、Controller、Util 中，容易出现重复逻辑和 bug |    全部内聚在领域层，唯一且不可修改，保证业务规则的一致性    |
|   可测试性   |     单元测试必须依赖数据库、中间件，测试成本高、速度慢      | 单元测试可以 mock 所有次端口，完全不依赖外部环境，测试速度快、覆盖全 |
|   可维护性   |         换数据库、换接口协议，需要修改大量业务代码          |     换技术实现，只需要替换适配器，核心业务代码完全不用改     |
|   代码语义   |      贫血模型，只有数据没有行为，代码无法体现业务语义       |    富领域模型，代码和业务语言完全一致，新人能快速理解业务    |

------

## 四、电商场景完整落地（下单核心场景，全链路对应）

我们用电商最核心的「用户提交订单」场景，把架构的每个环节落地，技术栈用 Java（符合你之前的技术背景），严格遵守依赖向内的规则。

### 1. 第一步：模块划分（锁定依赖规则）

先做 Maven 模块划分，**严格禁止内层依赖外层**，从根源上保证架构规范：


|             模块              | 对应架构层 |                           依赖范围                           |                           核心内容                           |
| :---------------------------: | :--------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|      `ecommerce-domain`       |   领域层   | 仅依赖 JDK、lombok 等基础工具，不依赖任何其他业务模块和框架  |  领域模型、领域服务、次端口接口、领域事件、Domain Primitive  |
|    `ecommerce-application`    |   应用层   | 仅依赖`ecommerce-domain`，不依赖任何外部框架（Spring、MyBatis 等） | Command/Query、Command/Query Handler、事件监听器、主端口接口 |
|  `ecommerce-adapter-primary`  | 主适配器层 |    依赖`ecommerce-application`，可依赖 Spring Web 等框架     |            Controller、CLI、RPC 消费者等主适配器             |
| `ecommerce-adapter-secondary` | 次适配器层 |    依赖`ecommerce-domain`，可依赖 MyBatis、第三方 SDK 等     |         Repository 实现、短信实现、MQ 实现等次适配器         |
|     `ecommerce-bootstrap`     |   启动层   |                         依赖所有模块                         |            Spring Boot 启动类、配置类、Bean 装配             |

### 2. 第二步：全链路流程映射（下单场景）

我们先把用户下单的完整流程，对应到架构的每个环节：

1. 用户在 APP 点击「提交订单」，发送 HTTP 请求到后端（主适配器：`OrderController`）

2. Controller 把 HTTP 参数转换成`CreateOrderCommand`，发送到 Command Bus

3. Command Bus 路由到`CreateOrderCommandHandler`（应用层）

4. Handler 编排下单流程：

    - 调用领域层`OrderDomainService`，校验商品合法性、计算订单金额、扣减库存、创建订单聚合根
    - 调用次端口`OrderRepository`，保存订单
    - 调用次端口`EventPublisher`，发布`OrderCreatedEvent`领域事件



5. 事件监听器（应用层）监听到

   ```
   OrderCreatedEvent
   ```

   ，调用：

    - 次端口`SmsPort`，给用户发送下单成功短信
    - 次端口`EventBus`，把事件发送到 MQ，通知仓储系统备货



6. 所有次端口的实现，都在`adapter-secondary`模块，核心完全不感知具体的技术实现。

### 3. 第三步：分模块代码落地

#### （1）`ecommerce-domain`模块（领域层，最核心）

先定义 Domain Primitive（之前你问过的概念，这里落地）：

```
// 订单ID（Domain Primitive）
public final class OrderId {
    private final String value;

    private OrderId(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("订单ID不能为空");
        }
        this.value = value;
    }

    public static OrderId of(String value) {
        return new OrderId(value);
    }

    public String getValue() {
        return value;
    }
}

// 订单金额（Domain Primitive）
public final class OrderAmount {
    private final BigDecimal value;

    private OrderAmount(BigDecimal value) {
        if (value == null) {
            throw new IllegalArgumentException("订单金额不能为空");
        }
        if (value.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("订单金额必须大于0");
        }
        this.value = value;
    }

    public static OrderAmount of(BigDecimal value) {
        return new OrderAmount(value);
    }

    public BigDecimal getValue() {
        return value;
    }
}
```

定义订单聚合根（领域模型，封装业务规则）：


```
// 订单状态枚举
public enum OrderStatus {
    UNPAID, PAID, SHIPPED, COMPLETED, CANCELLED
}

// 订单聚合根（核心业务规则都在这里）
public class Order {
    private final OrderId orderId;
    private final UserId userId;
    private final List<OrderItem> items;
    private final OrderAmount totalAmount;
    private OrderStatus status;
    private final Address shippingAddress;
    private final LocalDateTime createTime;

    // 私有化构造器，强制通过工厂方法创建，保证订单合法性
    private Order(OrderId orderId, UserId userId, List<OrderItem> items, OrderAmount totalAmount, Address shippingAddress) {
        this.orderId = orderId;
        this.userId = userId;
        this.items = items;
        this.totalAmount = totalAmount;
        this.shippingAddress = shippingAddress;
        this.status = OrderStatus.UNPAID;
        this.createTime = LocalDateTime.now();
    }

    // 工厂方法，封装订单创建的业务规则
    public static Order create(OrderId orderId, UserId userId, List<OrderItem> items, Address shippingAddress) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("订单商品不能为空");
        }
        // 计算订单总金额
        BigDecimal total = items.stream()
                .map(item -> item.getAmount().getValue())
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        return new Order(orderId, userId, items, OrderAmount.of(total), shippingAddress);
    }

    // 订单支付的业务规则，封装在聚合根内部
    public void pay() {
        if (this.status != OrderStatus.UNPAID) {
            throw new IllegalStateException("只有未支付订单可以支付");
        }可以支付");
        }
        this.status = OrderStatus.PAID;
    }

    // 取消订单的业务规则
    public void cancel() {
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.COMPLETED) {
            throw new IllegalStateException("已发货/已完成订单不能取消");
        }
        this.status = OrderStatus.CANCELLED;
    }

    // 只暴露getter，不暴露setter，保证不可变，避免外部随意修改状态
    public OrderId getOrderId() { return orderId; }
    public OrderStatus getStatus() { return status; }
    // 其他getter省略
}
```

定义次端口接口（领域层定义，适配器层实现）：

```
// 订单仓储次端口（领域层定义，不依赖MyBatis）
public interface OrderRepository {
    void save(Order order);
    Order findById(OrderId orderId);
}

// 短信发送次端口
public interface SmsPort {
    sendOrderSuccessSms(PhoneNumber phoneNumber, OrderId orderId);
}

// 事件发布次端口
public interface EventPublisher {
    void publish(Object event);
}
```

定义领域事件：


```
// 订单创建领域事件
public class OrderCreatedEvent {
    private final OrderId orderId;
    private final UserId userId;
    private final LocalDateTime eventTime;

    public OrderCreatedEvent(OrderId orderId, UserId userId) {
        this.orderId = orderId;
        this.userId = userId;
        this.eventTime = LocalDateTime.now();
    }

    // getter省略
}
```

#### （2）`ecommerce-application`模块（应用层，编排层）

定义 Command：

```
// 创建订单命令
public class CreateOrderCommand {
    private final String userId;
    private final List<OrderItemCommand> items;
    private final AddressCommand shippingAddress;

    // 构造器、getter省略
}
```

定义 Command Handler（应用服务，编排流程）：

```
// 下单用例处理器，对应主端口
@Component
public class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand, OrderId> {

    // 依赖领域层的次端口，只依赖接口，不依赖实现
    private final OrderDomainService orderDomainService;
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;

    // 构造器注入
    public CreateOrderCommandHandler(OrderDomainService orderDomainService, OrderRepository orderRepository, EventPublisher eventPublisher) {
        this.orderDomainService = orderDomainService;
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public OrderId handle(CreateOrderCommand command) {
        // 1. 调用领域服务，执行下单的核心业务逻辑
        Order order = orderDomainService.createOrder(
                UserId.of(command.getUserId()),
                command.getItems(),
                command.getShippingAddress()
        );

        // 2. 调用次端口，保存订单
        orderRepository.save(order);

        // 3. 调用次端口，发布领域事件
        eventPublisher.publish(new OrderCreatedEvent(order.getOrderId(), order.getUserId()));

        // 4. 返回订单ID
        return order.getOrderId();
    }
}
```

定义事件监听器：

```
// 订单创建事件监听器
@Component
public class OrderCreatedEventListener {

    private final SmsPort smsPort;
    private final UserRepository userRepository;

    public OrderCreatedEventListener(SmsPort smsPort, UserRepository userRepository) {
        this.smsPort = smsPort;
        this.userRepository = userRepository;
    }

    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 查询用户手机号
        User user = userRepository.findById(event.getUserId());
        // 调用次端口，发送短信
        smsPort.sendOrderSuccessSms(user.getPhoneNumber(), event.getOrderId());
    }
}
```

#### （3）`ecommerce-adapter-primary`模块（主适配器，输入层）

定义 Controller（主适配器，HTTP 协议转换）：


```
// 订单Controller，主适配器，只做协议转换，不写业务逻辑
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // 注入Command Bus，不直接调用Handler
    private final CommandBus commandBus;

    public OrderController(CommandBus commandBus) {
        this.commandBus = commandBus;
    }

    @PostMapping
    public ResponseEntity<OrderIdVO> createOrder(@RequestBody CreateOrderRequest request) {
        // 1. 把HTTP请求参数转换成Command
        CreateOrderCommand command = ConvertUtils.convert(request, CreateOrderCommand.class);
        // 2. 发送Command到Bus，交给应用层处理
        OrderId orderId = commandBus.send(command);
        // 3. 把返回结果转换成VO，返回给前端
        return ResponseEntity.ok(new OrderIdVO(orderId.getValue()));
    }
}
```

#### （4）`ecommerce-adapter-secondary`模块（次适配器，输出层）

实现订单仓储次端口（MySQL+MyBatis 实现）：




```
// 订单仓储实现，次适配器，实现领域层定义的接口
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;

    public OrderRepositoryImpl(OrderMapper orderMapper, OrderItemMapper orderItemMapper) {
        this.orderMapper = orderMapper;
        this.orderItemMapper = orderItemMapper;
    }

    @Override
    public void save(Order order) {
        // 把领域对象转换成数据库DO，做技术适配
        OrderDO orderDO = ConvertUtils.convert(order, OrderDO.class);
        orderMapper.insert(orderDO);

        List<OrderItemDO> itemDOList = order.getItems().stream()
                .map(item -> ConvertUtils.convert(item, OrderItemDO.class))
                .toList();
        orderItemMapper.batchInsert(itemDOList);
    }

    @Override
    public Order findById(OrderId orderId) {
        OrderDO orderDO = orderMapper.selectById(orderId.getValue());
        List<OrderItemDO> itemDOList = orderItemMapper.selectByOrderId(orderId.getValue());
        // 把数据库DO转换成领域对象，返回给核心
        return ConvertUtils.convertToOrder(orderDO, itemDOList);
    }
}
```

实现短信次端口（阿里云短信实现）：



```
// 短信实现，次适配器
@Component
public class SmsPortImpl implements SmsPort {

    // 注入阿里云短信SDK，只在适配器层依赖，核心完全不感知
    private final AliSmsClient aliSmsClient;

    public SmsPortImpl(AliSmsClient aliSmsClient) {
        this.aliSmsClient = aliSmsClient;
    }

    @Override
    public void sendOrderSuccessSms(PhoneNumber phoneNumber, OrderId orderId) {
        // 适配阿里云短信的协议，发送短信
        SmsRequest request = SmsRequest.builder()
                .phoneNumber(phoneNumber.getValue())
                .templateCode("ORDER_SUCCESS")
                .templateParam(Map.of("orderId", orderId.getValue()))
                .build();
        aliSmsClient.sendSms(request);
    }
}
```

### 4. CQRS 查询场景落地

图中的 CQRS 思想，查询场景完全分离，不走领域层，直接在应用层定义 Query Handler，调用查询适配器，提升性能：




```
// 订单查询Query
public class GetOrderListQuery {
    private final String userId;
    private final Integer pageNum;
    private final Integer pageSize;
    // 构造器、getter省略
}

// 查询处理器，直接调用查询端口，不走领域层
@Component
public class GetOrderListQueryHandler implements QueryHandler<GetOrderListQuery, Page<OrderVO>> {

    private final OrderQueryPort orderQueryPort;

    public GetOrderListQueryHandler(OrderQueryPort orderQueryPort) {
        this.orderQueryPort = orderQueryPort;
    }

    @Override
    public Page<OrderVO> handle(GetOrderListQuery query) {
        // 直接调用查询次端口，返回VO，不需要业务规则
        return orderQueryPort.queryOrderList(query.getUserId(), query.getPageNum(), query.getPageSize());
    }
}
```

------

## 五、从 0 到 1 的落地执行步骤（渐进式落地，避免重构风险）

### 1. 第一步：业务梳理（先做 DDD 领域建模）

- 梳理电商核心业务场景，划分限界上下文（订单上下文、商品上下文、库存上下文、用户上下文）；
- 每个限界上下文内，划分聚合根、实体、值对象，定义核心业务规则；
- 输出领域模型图，统一团队的业务语言，避免代码和业务脱节。

### 2. 第二步：搭建模块结构，锁定依赖规则

- 按照上面的模块划分，创建 Maven 模块，在 maven 中配置依赖禁令，禁止 domain 模块依赖 adapter 模块，从工具层面保证架构规范；
- 引入基础依赖，比如 lombok、Spring Boot，只在 bootstrap 和 adapter 模块引入框架依赖，domain 和 application 模块绝对不引入外部框架。

### 3. 第三步：实现领域层，定义端口契约

- 先实现领域模型、Domain Primitive、领域服务，把所有核心业务规则封装在领域层；
- 定义主端口和次端口接口，明确核心与外界的契约，先定接口，再写实现。

### 4. 第四步：实现应用层，编排用例流程

- 针对每个业务用例，定义 Command/Query，实现对应的 Handler，编排业务流程；
- 实现事件监听器，处理异步流程，保证主流程的简洁。

### 5. 第五步：实现适配器层，对接外部系统

- 实现主适配器：Controller、CLI 等，对接前端和外部系统的输入；
- 实现次适配器：Repository、短信、MQ 等，对接数据库和第三方服务。

### 6. 第六步：测试与渐进式替换

- 先写单元测试，mock 所有次端口，测试核心业务逻辑，保证业务规则的正确性；
- 再写集成测试，验证适配器和核心的对接；
- 对于旧系统，采用「绞杀者模式」，先把新业务用这个架构实现，再逐步把旧业务迁移过来，避免一次性重构的风险。

------

## 六、落地避坑指南

1. **不要过度设计**：图底部的注释明确说明「Understand all of this, but use only what you need」，简单的 CRUD 场景不需要完整的六边形架构，只有复杂业务系统才需要；
2. **严格遵守依赖规则**：绝对不能在 domain 和 application 模块里引入外部框架的代码，否则架构就会腐化，回到传统三层的老路；
3. **不要把业务逻辑写在适配器里**：主适配器和次适配器只能做协议转换，绝对不能写业务规则，所有业务规则必须内聚在领域层；
4. **不要滥用领域服务**：能放在聚合根里的业务逻辑，就不要放在领域服务里，领域服务只处理跨聚合的逻辑；
5. **保证聚合根的不变性**：聚合根的状态只能通过自身的方法修改，不能暴露 setter，避免外部随意修改，保证业务规则的一致性。