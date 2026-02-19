### 1. 单一职责原则 (SRP)

#### 业务背景

电商中「订单处理」是核心场景，违反 SRP 的典型问题是：一个类既处理订单创建，又计算优惠，还发送短信通知、记录日志，任何一个逻辑变更都要改这个类。

#### 反例（违反 SRP）：万能订单类

```
import java.util.Date;

// 一个类包揽：创建订单 + 计算优惠 + 发送通知 + 记录日志
public class OrderManager {
    // 创建订单（核心职责）
    public void createOrder(String orderId, double amount, String userId) {
        System.out.println("创建订单：" + orderId + "，金额：" + amount);
        
        // 职责1：计算优惠（不该属于此类）
        double discountAmount = calculateDiscount(amount, userId);
        System.out.println("优惠后金额：" + (amount - discountAmount));
        
        // 职责2：发送短信通知（不该属于此类）
        sendSmsNotification(userId, orderId);
        
        // 职责3：记录日志（不该属于此类）
        log("订单 " + orderId + " 于 " + new Date() + " 创建成功");
    }

    // 优惠计算
    private double calculateDiscount(double amount, String userId) {
        return userId.startsWith("VIP") ? amount * 0.1 : 0; // VIP减10%
    }

    // 短信通知
    private void sendSmsNotification(String userId, String orderId) {
        System.out.println("向用户 " + userId + " 发送短信：订单 " + orderId + " 创建成功");
    }

    // 日志记录
    private void log(String message) {
        System.out.println("[日志] " + message);
    }
}
```

#### 正例（遵循 SRP）：拆分电商核心职责

```
import java.util.Date;

// 1. 订单核心类：只负责订单创建
public class OrderService {
    public void createOrder(String orderId, double amount) {
        System.out.println("创建订单：" + orderId + "，原始金额：" + amount);
    }
}

// 2. 优惠计算类：只负责优惠规则
public class DiscountService {
    public double calculateDiscount(double amount, String userId) {
        return userId.startsWith("VIP") ? amount * 0.1 : 0;
    }
}

// 3. 通知服务类：只负责消息推送
public class NotificationService {
    public void sendSms(String userId, String orderId) {
        System.out.println("向用户 " + userId + " 发送短信：订单 " + orderId + " 创建成功");
    }
}

// 4. 日志类：只负责日志记录
public class OrderLogger {
    public void logOrder(String orderId) {
        System.out.println("[日志] 订单 " + orderId + " 于 " + new Date() + " 创建成功");
    }
}

// 调用示例（高层组装）
public class TestSRP {
    public static void main(String[] args) {
        String orderId = "ORD20260217001";
        double amount = 1000.0;
        String userId = "VIP123456";

        OrderService orderService = new OrderService();
        DiscountService discountService = new DiscountService();
        NotificationService notificationService = new NotificationService();
        OrderLogger logger = new OrderLogger();

        // 分步执行，职责清晰
        orderService.createOrder(orderId, amount);
        double discount = discountService.calculateDiscount(amount, userId);
        System.out.println("优惠后金额：" + (amount - discount));
        notificationService.sendSms(userId, orderId);
        logger.logOrder(orderId);
    }
}
```

#### 核心说明

电商场景中，订单、优惠、通知、日志是完全独立的职责：

- 优惠规则变更（比如新增满减），只需改 `DiscountService`，不影响订单创建；
- 通知方式变更（比如从短信改推送），只需改 `NotificationService`，无需动订单逻辑。

### 2. 开放封闭原则 (OCP)

#### 业务背景

电商支付场景：初期只支持支付宝支付，后续要新增微信、银联支付，违反 OCP 会直接修改原有支付类，遵循 OCP 则通过扩展实现新支付方式。

#### 反例（违反 OCP）：修改原有代码新增支付方式

```
// 支付类：新增支付方式需修改原有方法
public class PaymentService {
    // 初始只支持支付宝
    public void pay(String orderId, double amount) {
        payByAlipay(orderId, amount);
    }

    // 新增微信支付 → 被迫修改类，新增方法/改原有方法
    public void pay(String orderId, double amount, String payType) {
        if ("alipay".equals(payType)) {
            payByAlipay(orderId, amount);
        } else if ("wechat".equals(payType)) { // 新增逻辑，修改原类
            payByWechat(orderId, amount);
        }
    }

    private void payByAlipay(String orderId, double amount) {
        System.out.println("支付宝支付：订单 " + orderId + "，金额 " + amount);
    }

    private void payByWechat(String orderId, double amount) {
        System.out.println("微信支付：订单 " + orderId + "，金额 " + amount);
    }
}
```

#### 正例（遵循 OCP）：扩展支付方式，不修改原有代码

```
// 抽象支付接口（对扩展开放）
public interface PaymentStrategy {
    void pay(String orderId, double amount);
}

// 原有实现：支付宝支付（无需修改）
public class AlipayStrategy implements PaymentStrategy {
    @Override
    public void pay(String orderId, double amount) {
        System.out.println("支付宝支付：订单 " + orderId + "，金额 " + amount);
    }
}

// 扩展实现：微信支付（新增类，不修改原有代码）
public class WechatPayStrategy implements PaymentStrategy {
    @Override
    public void pay(String orderId, double amount) {
        System.out.println("微信支付：订单 " + orderId + "，金额 " + amount);
    }
}

// 扩展实现：银联支付（继续扩展，仍无需改原有代码）
public class UnionPayStrategy implements PaymentStrategy {
    @Override
    public void pay(String orderId, double amount) {
        System.out.println("银联支付：订单 " + orderId + "，金额 " + amount);
    }
}

// 支付管理器（依赖抽象，对修改封闭）
public class PaymentManager {
    private PaymentStrategy strategy;

    public PaymentManager(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void processPayment(String orderId, double amount) {
        strategy.pay(orderId, amount);
    }
}

// 调用示例
public class TestOCP {
    public static void main(String[] args) {
        String orderId = "ORD20260217002";
        double amount = 500.0;

        // 支付宝支付（原有逻辑）
        PaymentManager alipayManager = new PaymentManager(new AlipayStrategy());
        alipayManager.processPayment(orderId, amount);

        // 微信支付（扩展逻辑，无修改）
        PaymentManager wechatManager = new PaymentManager(new WechatPayStrategy());
        wechatManager.processPayment(orderId, amount);
    }
}
```

#### 核心说明

电商支付方式是高频扩展场景，遵循 OCP 后：

- 新增「花呗支付」只需新增 `HuabeiPayStrategy` 实现接口；
- 原有支付逻辑（支付宝 / 微信）完全无需修改，避免引入新 Bug。

### 3. 里氏替换原则 (LSP)

#### 业务背景

电商物流场景：父类「快递物流」约定了「配送」和「计算运费」的契约，子类「顺丰快递」「邮政平邮」必须能完全替换父类，不能破坏契约（比如平邮不能实现「次日达」却继承「极速物流」）。

#### 反例（违反 LSP）：子类破坏父类契约

```
// 父类：极速物流（契约：次日达，运费按重量×2）
public class ExpressLogistics {
    // 配送：次日达
    public void deliver(String orderId) {
        System.out.println("极速物流：订单 " + orderId + " 次日达");
    }

    // 计算运费：重量×2
    public double calculateFee(double weight) {
        return weight * 2;
    }
}

// 子类：邮政平邮（无法满足"次日达"，破坏父类契约）
public class PostLogistics extends ExpressLogistics {
    @Override
    public void deliver(String orderId) {
        // 违反父类契约：平邮无法次日达
        throw new UnsupportedOperationException("邮政平邮不支持次日达！");
    }

    @Override
    public double calculateFee(double weight) {
        // 运费规则也改了，破坏契约
        return weight * 1;
    }
}

// 调用时出错
public class TestLSP {
    public static void main(String[] args) {
        // 期望用极速物流，实际传入平邮，程序崩溃
        ExpressLogistics logistics = new PostLogistics();
        logistics.deliver("ORD20260217003"); // 抛出异常
    }
}
```

#### 正例（遵循 LSP）：重构抽象，子类符合父类契约

```
// 基础物流抽象：定义所有物流的通用契约（运费计算、配送）
public abstract class Logistics {
    // 通用契约：计算运费（子类可自定义规则，但必须实现）
    public abstract double calculateFee(double weight);

    // 通用契约：配送（子类可自定义时效，但必须实现）
    public abstract void deliver(String orderId);
}

// 极速物流子类：符合自身契约（次日达，运费×2）
public class ExpressLogistics extends Logistics {
    @Override
    public double calculateFee(double weight) {
        return weight * 2;
    }

    @Override
    public void deliver(String orderId) {
        System.out.println("顺丰极速物流：订单 " + orderId + " 次日达");
    }
}

// 平邮子类：符合自身契约（7日达，运费×1）
public class PostLogistics extends Logistics {
    @Override
    public double calculateFee(double weight) {
        return weight * 1;
    }

    @Override
    public void deliver(String orderId) {
        System.out.println("邮政平邮：订单 " + orderId + " 7日内送达");
    }
}

// 调用示例：子类可完全替换父类，行为稳定
public class TestLSP {
    public static void main(String[] args) {
        // 替换为极速物流，行为正常
        Logistics logistics1 = new ExpressLogistics();
        logistics1.deliver("ORD20260217003");
        System.out.println("极速物流运费：" + logistics1.calculateFee(5));

        // 替换为平邮，行为也正常（符合自身契约）
        Logistics logistics2 = new PostLogistics();
        logistics2.deliver("ORD20260217003");
        System.out.println("平邮运费：" + logistics2.calculateFee(5));
    }
}
```

#### 核心说明

电商物流的核心是「配送时效 + 运费规则」，LSP 要求：

- 子类不能否定父类的核心契约（比如「极速物流」的子类必须支持极速配送）；
- 不同物流类型应抽象为同级子类，而非继承不符合自身特性的父类。

### 4. 接口隔离原则 (ISP)

#### 业务背景

电商商品场景：不同商品类型（实物商品、虚拟商品、服务类商品）有不同行为，违反 ISP 会定义一个包含所有行为的大接口，导致子类被迫实现无关方法（比如虚拟商品无需「物流配送」）。

#### 反例（违反 ISP）：大而全的商品接口

```
// 大接口：包含实物+虚拟商品的所有行为
public interface IProduct {
    void sell(); // 售卖
    void deliver(); // 物流配送（虚拟商品不需要）
    void recharge(); // 充值（实物商品不需要）
    void refund(); // 退款
}

// 虚拟商品（话费）：被迫实现无关的deliver方法
public class VirtualProduct implements IProduct {
    @Override
    public void sell() {
        System.out.println("售卖话费商品");
    }

    @Override
    public void deliver() {
        // 虚拟商品无需配送，被迫空实现
    }

    @Override
    public void recharge() {
        System.out.println("话费充值成功");
    }

    @Override
    public void refund() {
        System.out.println("话费退款");
    }
}
```

#### 正例（遵循 ISP）：拆分小接口，按需实现

```
// 核心接口：所有商品通用（售卖、退款）
public interface IBaseProduct {
    void sell();
    void refund();
}

// 实物商品专属接口：物流配送
public interface IDeliverable {
    void deliver();
}

// 虚拟商品专属接口：充值
public interface IRechargeable {
    void recharge();
}

// 实物商品（手机）：实现核心+配送接口
public class PhysicalProduct implements IBaseProduct, IDeliverable {
    @Override
    public void sell() {
        System.out.println("售卖手机商品");
    }

    @Override
    public void refund() {
        System.out.println("手机退款");
    }

    @Override
    public void deliver() {
        System.out.println("手机物流配送");
    }
}

// 虚拟商品（话费）：实现核心+充值接口
public class VirtualProduct implements IBaseProduct, IRechargeable {
    @Override
    public void sell() {
        System.out.println("售卖话费商品");
    }

    @Override
    public void refund() {
        System.out.println("话费退款");
    }

    @Override
    public void recharge() {
        System.out.println("话费充值成功");
    }
}
```

#### 核心说明

电商商品类型差异大，ISP 要求：

- 接口只包含客户端需要的方法，避免「冗余方法」；
- 新增「服务类商品（比如安装服务）」，只需新增 `IServiceable` 接口，不影响原有商品实现。

### 5. 依赖倒置原则 (DIP)

#### 业务背景

电商订单优惠场景：高层的「订单结算」模块，不应直接依赖低层的「优惠券、满减、折扣」等具体优惠实现，而应依赖抽象的优惠接口。

#### 反例（违反 DIP）：高层依赖低层具体类

```
// 低层具体类：优惠券优惠
public class CouponDiscount {
    public double discount(double amount, String couponCode) {
        return "FULL100减20".equals(couponCode) ? 20 : 0;
    }
}

// 高层模块：订单结算（直接依赖具体优惠类）
public class OrderSettlementService {
    // 硬编码依赖优惠券，无法替换为满减
    private CouponDiscount couponDiscount = new CouponDiscount();

    public double settle(String orderId, double amount, String couponCode) {
        double discount = couponDiscount.discount(amount, couponCode);
        System.out.println("订单 " + orderId + " 优惠券优惠：" + discount);
        return amount - discount;
    }
}
```

#### 正例（遵循 DIP）：依赖抽象，通过注入解耦


```
// 抽象优惠接口：定义通用优惠行为
public interface Discount {
    double calculateDiscount(double amount, String param); // param：优惠券码/满减规则等
}

// 低层实现1：优惠券优惠
public class CouponDiscount implements Discount {
    @Override
    public double calculateDiscount(double amount, String couponCode) {
        return "FULL100减20".equals(couponCode) ? 20 : 0;
    }
}

// 低层实现2：满减优惠（扩展）
public class FullReduceDiscount implements Discount {
    @Override
    public double calculateDiscount(double amount, String rule) {
        return amount >= 200 ? 30 : 0; // 满200减30
    }
}

// 高层模块：依赖抽象，不依赖具体实现
public class OrderSettlementService {
    private Discount discount;

    // 构造器注入：灵活替换优惠类型
    public OrderSettlementService(Discount discount) {
        this.discount = discount;
    }

    public double settle(String orderId, double amount, String param) {
        double discountAmount = discount.calculateDiscount(amount, param);
        System.out.println("订单 " + orderId + " 优惠金额：" + discountAmount);
        return amount - discountAmount;
    }
}

// 调用示例
public class TestDIP {
    public static void main(String[] args) {
        String orderId = "ORD20260217004";
        double amount = 200.0;

        // 优惠券优惠（原有逻辑）
        Discount couponDiscount = new CouponDiscount();
        OrderSettlementService service1 = new OrderSettlementService(couponDiscount);
        service1.settle(orderId, amount, "FULL100减20");

        // 满减优惠（扩展逻辑，高层无需修改）
        Discount fullReduceDiscount = new FullReduceDiscount();
        OrderSettlementService service2 = new OrderSettlementService(fullReduceDiscount);
        service2.settle(orderId, amount, "FULL200减30");
    }
}
```

#### 核心说明

电商优惠规则是高频变更场景，DIP 带来的好处：

- 新增「折扣优惠」只需新增 `PercentDiscount` 实现 `Discount` 接口；
- 订单结算逻辑完全无需修改，符合 OCP 原则；
- 单元测试时可注入「模拟优惠实现」，无需依赖真实优惠逻辑。

### 总结

结合电商场景，SOLID 原则的核心落地要点：

1. **SRP**：电商核心模块（订单、支付、物流、优惠）拆分为独立类，一个类只管一件事；
2. **OCP**：电商高频扩展场景（支付方式、优惠规则）通过接口扩展，不修改原有代码；
3. **LSP**：电商子类（不同物流 / 商品）必须遵守父类核心契约，替换后程序行为稳定；
4. **ISP**：电商不同类型实体（实物 / 虚拟商品）只实现自身需要的接口，避免冗余；
5. **DIP**：电商高层模块（订单结算）依赖抽象（优惠接口），而非具体实现（优惠券 / 满减）。