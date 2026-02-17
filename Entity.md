### 一、DDD中Entity的核心特征


在 DDD 中，实体不是简单的 POJO，它的核心特征是：

1. **拥有唯一标识（Identity）**：区分两个实体的不是属性，而是唯一 ID（比如用户 ID、订单 ID），哪怕属性完全一样，ID 不同就是不同实体。
2. **具备业务行为**：实体不仅包含数据（属性），更要封装核心的业务逻辑和行为，体现 “充血模型”。
3. **生命周期可变**：实体的属性会随业务流程变化（比如订单状态从 “待支付” 变 “已支付”）。
4. **值相等基于 ID**：重写`equals()`和`hashCode()`时，只基于唯一 ID，而非所有属性。


### 二、Java 中实体的设计步骤与示例

下面以 “订单（Order）” 这个典型实体为例，一步步展示规范的设计方式。

#### 1. 基础结构：定义唯一标识 + 核心属性
```
import lombok.Getter;
import java.time.LocalDateTime;
import java.util.Objects;

// 订单实体（核心领域对象）
@Getter // 只暴露必要的读方法，写操作通过业务方法完成
public class Order {
    // 1. 唯一标识（核心）：通常用领域ID（如OrderId）而非基础类型，增强类型安全
    private final OrderId id;
    
    // 2. 核心业务属性（与订单强相关的属性）
    private final CustomerId customerId; // 关联客户（值对象/实体ID）
    private OrderStatus status; // 订单状态（枚举，体现业务规则）
    private LocalDateTime createTime;
    private LocalDateTime payTime;
    private BigDecimal totalAmount; // 订单总金额

    // 3. 构造方法：保证实体创建时的完整性（强制必填属性，避免无效状态）
    public Order(OrderId id, CustomerId customerId, BigDecimal totalAmount) {
        // 前置校验：确保实体创建时符合业务规则
        if (id == null) {
            throw new IllegalArgumentException("订单ID不能为空");
        }
        if (customerId == null) {
            throw new IllegalArgumentException("客户ID不能为空");
        }
        if (totalAmount == null || totalAmount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("订单金额必须大于0");
        }
        
        this.id = id;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.status = OrderStatus.PENDING_PAYMENT; // 初始状态：待支付
        this.createTime = LocalDateTime.now();
    }

    // 4. 业务行为（核心）：封装订单的状态变更和业务逻辑
    /**
     * 订单支付
     * 封装“支付”的业务规则，外部无法直接修改status和payTime
     */
    public void pay() {
        // 业务规则校验：只有待支付的订单才能支付
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new IllegalStateException("当前订单状态为" + status + "，无法支付");
        }
        this.status = OrderStatus.PAID;
        this.payTime = LocalDateTime.now();
    }

    /**
     * 取消订单
     */
    public void cancel() {
        // 业务规则：已支付的订单不能取消
        if (this.status == OrderStatus.PAID) {
            throw new IllegalStateException("已支付的订单无法取消");
        }
        this.status = OrderStatus.CANCELLED;
    }

    // 5. 重写equals和hashCode：仅基于唯一ID
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Order order = (Order) o;
        return Objects.equals(id, order.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    // 订单状态枚举（体现业务规则）
    public enum OrderStatus {
        PENDING_PAYMENT("待支付"),
        PAID("已支付"),
        CANCELLED("已取消");

        private final String desc;

        OrderStatus(String desc) {
            this.desc = desc;
        }
    }

    // 领域ID（类型安全，避免Long/String等基础类型混用）
    @Getter
    public static class OrderId {
        private final Long value;

        public OrderId(Long value) {
            if (value == null || value <= 0) {
                throw new IllegalArgumentException("订单ID必须为正整数");
            }
            this.value = value;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            OrderId orderId = (OrderId) o;
            return Objects.equals(value, orderId.value);
        }

        @Override
        public int hashCode() {
            return Objects.hash(value);
        }
    }

    // 客户ID（同理，领域ID）
    @Getter
    public static class CustomerId {
        private final Long value;

        public CustomerId(Long value) {
            if (value == null || value <= 0) {
                throw new IllegalArgumentException("客户ID必须为正整数");
            }
            this.value = value;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            CustomerId that = (CustomerId) o;
            return Objects.equals(value, that.value);
        }

        @Override
        public int hashCode() {
            return Objects.hash(value);
        }
    }
}
```

#### 2. 关键设计点解释

表格







|          设计点           |                             说明                             |
| :-----------------------: | :----------------------------------------------------------: |
| **领域 ID（如 OrderId）** | 不用 Long/String 直接作为 ID，而是封装成自定义类型，避免 “订单 ID” 和 “客户 ID” 混用（比如把客户 ID 传给订单 ID 参数），增强类型安全。 |
|  **私有属性 + 受控修改**  | 属性私有化，不提供 setter 方法，所有状态变更必须通过业务方法（如`pay()`、`cancel()`）完成，确保业务规则不被破坏。 |
|     **构造方法校验**      | 实体创建时就保证 “有效状态”，避免创建出 “金额为负”“无客户 ID” 的无效订单。 |
|     **业务行为封装**      | 把 “支付”“取消” 等业务逻辑写在实体内部，而非外部 Service，符合 “充血模型”，让实体体现领域规则。 |
|    **equals/hashCode**    | 只基于唯一 ID，比如两个订单属性完全一样但 ID 不同，就是两个不同的订单；ID 相同则是同一个订单（哪怕其他属性变了）。 |

#### 3. 实体的使用示例
```
public class OrderTest {
    public static void main(String[] args) {
        // 1. 创建有效订单
        Order.OrderId orderId = new Order.OrderId(1001L);
        Order.CustomerId customerId = new Order.CustomerId(2001L);
        Order order = new Order(orderId, customerId, new BigDecimal("99.9"));
        
        // 2. 执行支付行为
        order.pay();
        System.out.println("支付后状态：" + order.getStatus()); // 输出：PAID
        
        // 3. 尝试取消已支付的订单（会抛出异常）
        try {
            order.cancel();
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage()); // 输出：已支付的订单无法取消
        }
        
        // 4. 验证equals：ID相同则相等
        Order sameIdOrder = new Order(orderId, new Order.CustomerId(2002L), new BigDecimal("199.9"));
        System.out.println(order.equals(sameIdOrder)); // 输出：true
    }
}
```

### 三、实体设计的避坑点

1. **避免 “贫血模型”**：不要只做属性 + getter/setter，把所有业务逻辑都放到 Service 里，实体变成单纯的数据载体。
2. **ID 不要用自增主键（可选）**：优先用分布式 ID（如雪花算法），避免依赖数据库自增，符合 “领域模型与存储解耦”。
3. **属性不要过度冗余**：只保留实体的核心属性，关联其他对象时优先用 “对方的 ID” 而非完整对象（避免循环依赖）。
4. **不要暴露内部状态**：比如不要把`status`设为 public，也不要提供`setStatus()`方法，所有状态变更必须通过业务方法。

### 总结

1. DDD 实体的核心是**唯一 ID + 业务行为**，而非单纯的数据，要遵循 “充血模型” 封装业务规则。
2. Java 设计时，通过**私有属性 + 业务方法**控制状态变更，构造方法保证创建时的有效性，重写 equals/hashCode 仅基于 ID。
3. 用**领域 ID（自定义类型）** 替代基础类型 ID，增强类型安全，避免数据混用。