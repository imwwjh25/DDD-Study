### 一、先搞懂核心定义和区别

为了让你直观理解，我先分别定义这两个概念，再对比它们的核心差异：

#### 1. 普通的领域对象（Domain Object）

通常指我们日常开发中写的 “贫血模型” 实体类，比如：

```
// 普通的领域对象（贫血模型）
public class Order {
    // 仅包含数据，无业务逻辑
    private String orderId;
    private BigDecimal amount;
    private String userId;
    private String status;

    // 只有getter/setter，无任何业务行为
    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }
    // 其他getter/setter...
}
```

它的特点：

- 只承载数据，没有封装业务规则 / 校验逻辑；
- 字段暴露在外，外部代码可以随意修改（比如直接 setAmount 设置负数金额）；
- 业务逻辑分散在 Service、Util 等外部类中，导致代码冗余且易出错。

#### 2. 领域原生体（Domain Primitive）

是**封装了单一领域概念、包含完整业务规则和校验逻辑的不可变对象**，聚焦 “一个原子化的领域概念”，比如订单金额、手机号、订单号等：


```
// 领域原生体：订单金额（单一领域概念）
public class OrderAmount {
    private final BigDecimal value; // 不可变（final）

    // 私有化构造器，强制通过静态方法创建
    private OrderAmount(BigDecimal value) {
        this.value = value;
    }

    // 工厂方法：封装校验逻辑，确保金额合法
    public static OrderAmount of(BigDecimal value) {
        if (value == null) {
            throw new IllegalArgumentException("订单金额不能为空");
        }
        if (value.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("订单金额必须大于0");
        }
        return new OrderAmount(value);
    }

    // 只暴露getter，无setter，保证不可变
    public BigDecimal getValue() {
        return value;
    }

    // 封装领域行为：比如金额相加
    public OrderAmount add(OrderAmount other) {
        return OrderAmount.of(this.value.add(other.getValue()));
    }
}

// 改造后的Order（使用DP）
public class Order {
    private OrderId orderId; // 用DP替代字符串
    private OrderAmount amount; // 用DP替代BigDecimal
    private UserId userId;
    private OrderStatus status;

    // 业务行为内聚在领域对象中
    public void pay() {
        if (this.status != OrderStatus.UNPAID) {
            throw new IllegalStateException("只有未支付订单可支付");
        }
        this.status = OrderStatus.PAID;
    }
}
```

它的特点：

- **单一职责**：只封装一个原子化的领域概念（比如 “订单金额” 而非 “整个订单”）；
- **自校验**：创建时就保证数据合法，杜绝非法值；
- **不可变**：一旦创建就不能修改，避免并发问题和意外修改；
- **行为内聚**：相关的业务操作（比如金额相加）封装在内部，而非外部 Service。

#### 核心区别对比




|    维度    |         普通领域对象          |             领域原生体（DP）             |
| :--------: | :---------------------------: | :--------------------------------------: |
|    职责    |      承载多维度领域数据       |          封装单一原子化领域概念          |
|  业务逻辑  |      无，逻辑分散在外部       |        自包含校验、计算等业务逻辑        |
| 数据安全性 |    低（可随意 set 非法值）    |         高（创建时校验，不可变）         |
|   复用性   |       低（仅数据载体）        |           高（可在全领域复用）           |
|   可读性   | 低（String / 数字无业务语义） | 高（OrderAmount 比 BigDecimal 更易理解） |

### 二、为什么要提出 Domain Primitive 这个概念？

核心目的是**解决普通领域对象 “数据和逻辑分离” 导致的代码坏味道**，具体价值体现在：

#### 1. 杜绝非法数据，提升代码健壮性

普通领域对象中，你可能在 10 个 Service 里都写 “校验订单金额大于 0” 的逻辑，一旦漏写就会产生非法数据；而 DP 在创建时就强制校验，所有使用 DP 的地方都能保证数据合法，从源头避免 bug。

#### 2. 消除重复代码，提升复用性

比如 “手机号格式校验”“身份证合法性校验” 这类通用逻辑，不用在 Controller、Service、Mapper 里反复写，封装到`PhoneNumber`、`IdCard`这些 DP 中，全项目复用。

#### 3. 增强代码的业务语义，提升可读性

`OrderAmount amount` 比 `BigDecimal amount` 更能体现 “订单金额” 的业务含义，新人看代码时不用猜 “这个 BigDecimal 到底代表什么”；同时，`amount.add(otherAmount)` 比 `amount.add(otherAmount).compareTo(0) > 0` 更贴合业务表达。

#### 4. 简化业务逻辑，降低维护成本

原本分散在 Service 中的校验、计算逻辑，内聚到 DP 中后，业务代码会更简洁。比如支付逻辑：

- 用普通对象：需要先校验金额 > 0、订单状态是未支付，再修改状态；
- 用 DP：直接调用`order.pay()`，内部已封装所有校验，代码更简洁且不易出错。

#### 5. 适配 DDD 的 “领域建模” 思想

DDD 的核心是 “用代码还原业务领域”，而普通的贫血模型只是 “数据容器”，无法体现业务规则；DP 通过封装原子化的领域概念，让代码更贴近业务语言，是实现 “富领域模型” 的基础。

### 总结

1. **核心区别**：普通领域对象是 “无逻辑的数据载体”，DP 是 “封装单一领域概念、自带校验和业务行为的不可变对象”；
2. **提出 DP 的核心价值**：从源头保证数据合法性，消除重复代码，增强业务语义，让领域模型更贴合真实业务，最终降低系统维护成本；
3. **使用场景**：任何有业务规则的原子化领域概念（如金额、手机号、订单号、状态等），都适合封装为 DP。