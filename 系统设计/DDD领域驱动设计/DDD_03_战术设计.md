# 第3章：战术设计（Tactical Design）

在领域驱动设计（DDD）中，战术设计关注的是领域模型的具体实现细节和代码层面的构建。它提供了一套模式和工具，帮助开发者将领域知识转化为可工作的代码。战术设计是在战略设计（限界上下文、通用语言等）的基础上进行的更细粒度设计。

## 3.1 实体（Entity）

### 3.1.1 实体的核心特征：身份标识与生命周期
实体是领域模型中具有唯一标识的对象，其身份在整个生命周期中保持不变。即使实体的所有属性都发生变化，只要其标识符（ID）不变，我们仍将其视为同一个实体。

```java
public class User {
    private final UserId id; // 唯一标识
    private String name;
    private String email;
    
    // 构造函数、getter和setter等
}
```

### 3.1.2 实体与值对象的区别
- 实体：有唯一标识，关注生命周期和身份连续性，可变
- 值对象：无唯一标识，关注属性值，不可变

典型的区分案例：
- 用户（User）是实体：即使名称、邮箱等属性变化，仍然是同一个用户
- 地址（Address）通常是值对象：完全由其组成部分定义，不需追踪变化

### 3.1.3 实体的设计原则：行为驱动而非数据驱动
实体应该封装业务规则和行为，而不仅仅是数据的容器。

❌ 贫血模型（反面示例）：
```java
public class Order {
    private OrderId id;
    private List<OrderItem> items;
    private OrderStatus status;
    
    // 只有getter和setter，没有业务行为
}

// 业务逻辑放在服务层
public class OrderService {
    public void placeOrder(Order order) {
        // 处理订单逻辑
    }
}
```

✅ 充血模型（正确示例）：
```java
public class Order {
    private OrderId id;
    private List<OrderItem> items;
    private OrderStatus status;
    
    public void addItem(Product product, int quantity) {
        validateProductAvailability(product);
        OrderItem item = new OrderItem(product, quantity);
        items.add(item);
    }
    
    public void place() {
        validateOrderCanBePlaced();
        this.status = OrderStatus.PLACED;
        // 可能发布领域事件
    }
    
    private void validateOrderCanBePlaced() {
        if (items.isEmpty()) {
            throw new IllegalStateException("订单不能为空");
        }
        // 其他验证逻辑
    }
}
```

### 3.1.4 实体设计的常见错误与优化方法
1. **过大的实体**：实体包含过多责任和属性
   - 解决方案：提取值对象，分解成多个相关实体

2. **实体间过度耦合**：实体直接引用其他上下文的实体
   - 解决方案：使用ID引用，或通过防腐层隔离

3. **业务规则分散**：实体只包含数据，规则散布在各处
   - 解决方案：将业务规则封装进实体方法中

4. **可变性滥用**：过度暴露setter方法
   - 解决方案：使用行为方法表达状态变化，限制setter的使用

## 3.2 值对象（Value Object）

### 3.2.1 值对象的特性：不变性、无标识、可替换性
值对象是通过其属性定义的，没有概念上的标识。当两个值对象的所有属性相等时，它们就是相等的。

特性：
- **不变性**：创建后不可修改，任何修改都返回新实例
- **无标识**：没有唯一标识符，完全由属性定义
- **可替换性**：可以随时替换为具有相同属性的另一个实例

### 3.2.2 值对象的设计技巧与实现模式
```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    
    // 不提供setter，所有操作返回新实例
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("无法添加不同货币");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    // 必须重写equals和hashCode
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.equals(money.amount) && currency.equals(money.currency);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}
```

### 3.2.3 常见的值对象示例
- **货币金额**（Money）：金额和货币单位
- **日期区间**（DateRange）：开始日期和结束日期
- **地址**（Address）：街道、城市、邮编等
- **坐标**（Coordinate）：经度和纬度
- **个人姓名**（PersonName）：姓、名、中间名等

### 3.2.4 Java中实现不可变值对象的最佳实践
1. 类声明为`final`，防止被继承
2. 所有字段声明为`private final`
3. 不提供修改状态的方法（setter）
4. 如果包含可变对象，确保深度防御：
   - 构造函数中创建防御性副本
   - getter返回防御性副本
5. 重写`equals()`和`hashCode()`方法
6. 实现合适的`toString()`方法

Java 16+可使用Record类型简化值对象实现：
```java
public record Address(String street, String city, String zipCode) {
    // 业务方法
    public boolean isInternational() {
        // 实现
    }
}
```

## 3.3 聚合（Aggregate）

### 3.3.1 聚合的定义与边界
聚合是一组相关对象的集合，作为数据修改的单元。每个聚合有一个根实体（聚合根），外部对象只能引用聚合根。聚合内部可以包含其他实体和值对象。

聚合边界确定了一致性的边界：
- 聚合内的所有对象必须一起保持一致
- 跨聚合的一致性可以最终一致

### 3.3.2 聚合根（Aggregate Root）的职责与选择方法
聚合根是聚合中唯一可以被外部引用的实体，它负责：
- 保证聚合内部的不变性规则
- 控制对聚合内其他实体的访问
- 协调聚合内部对象的生命周期

选择聚合根的方法：
1. 识别真正的不变性规则
2. 选择能够强制这些规则的实体
3. 通常选择业务上有意义的主实体

例如：订单（Order）是聚合根，订单项（OrderItem）是内部实体：
```java
public class Order {
    private OrderId id;
    private CustomerId customerId; // 只引用聚合根ID
    private List<OrderItem> items; // 内部实体
    private OrderStatus status;
    
    public void addItem(Product product, int quantity) {
        // 验证和添加逻辑
    }
    
    // 其他业务方法
}
```

### 3.3.3 聚合间的引用规则：引用聚合根而非内部元素
聚合之间的引用应该只通过ID进行，而不是直接对象引用：

```java
// 错误示例
public class Order {
    private Customer customer; // 直接引用另一个聚合
}

// 正确示例
public class Order {
    private CustomerId customerId; // 只保存ID引用
}
```

### 3.3.4 聚合设计的原则
1. **一致性边界**：聚合内部必须保持强一致性
2. **事务边界**：一个事务应该只修改一个聚合
3. **并发控制**：乐观锁通常用于聚合级别
4. **最小化引用**：聚合之间通过ID引用，而非直接引用
5. **删除规则**：删除聚合根会删除聚合内的所有对象

### 3.3.5 聚合大小的权衡：大聚合vs小聚合
**大聚合优势**：
- 强一致性边界更大
- 封装更多的业务规则
- 减少跨聚合协作

**大聚合劣势**：
- 性能开销（加载、保存）
- 并发冲突增加
- 复杂性提高

**小聚合优势**：
- 加载和保存更高效
- 减少并发冲突
- 更容易理解和测试

**小聚合劣势**：
- 需要更复杂的跨聚合协作
- 可能需要最终一致性的机制

经验法则：**尽可能小，但足够大以保证业务规则**。

## 3.4 领域服务（Domain Service）

### 3.4.1 领域服务的适用场景：跨实体的操作
当某个操作概念上不属于任何单个实体或值对象时，应该使用领域服务。典型场景：
- 涉及多个聚合的协作
- 复杂的业务规则不适合放在单个实体中
- 需要访问外部资源（如第三方服务）的领域逻辑

```java
public class TransferService {
    public void transfer(AccountId fromId, AccountId toId, Money amount) {
        Account from = accountRepository.findById(fromId);
        Account to = accountRepository.findById(toId);
        
        // 验证业务规则
        from.withdraw(amount);
        to.deposit(amount);
        
        // 保存状态
        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

### 3.4.2 领域服务的设计原则与防腐原则
1. **无状态**：领域服务应该是无状态的
2. **命名明确**：名称应反映领域概念和操作
3. **专注于领域逻辑**：避免包含基础设施和应用层面的逻辑
4. **依赖倒置**：依赖接口而非具体实现

防腐原则：通过接口隔离外部系统影响，保护领域模型纯净。

### 3.4.3 领域服务与应用服务的区别

| 领域服务 | 应用服务 |
|---------|---------|
| 关注领域概念和规则 | 关注用例和应用流程 |
| 不处理事务、安全等技术问题 | 负责事务管理、安全、日志等 |
| 是领域模型的一部分 | 是应用层的一部分 |
| 由领域专家参与定义 | 主要由技术需求驱动 |

### 3.4.4 保持领域服务的纯粹性
1. 避免将基础设施关注点混入领域服务
2. 不在领域服务中进行事务管理、认证等
3. 使用依赖注入提供所需的资源库等依赖
4. 保持领域服务的方法参数和返回值为领域概念（实体、值对象等）

## 3.5 领域事件（Domain Event）

### 3.5.1 领域事件的作用与优势
领域事件表示领域中发生的有意义的事情，通常是过去式命名（OrderPlaced, PaymentReceived）。

优势：
- 降低系统耦合度
- 支持分布式系统集成
- 提高系统可扩展性
- 更好地体现领域专家的思维方式
- 简化复杂流程的实现

### 3.5.2 事件驱动架构与DDD的结合
```java
public class Order {
    private List<DomainEvent> events = new ArrayList<>();
    
    public void place() {
        validateOrderCanBePlaced();
        this.status = OrderStatus.PLACED;
        
        // 记录领域事件
        events.add(new OrderPlacedEvent(this.id, LocalDateTime.now()));
    }
    
    public List<DomainEvent> getAndClearEvents() {
        List<DomainEvent> result = new ArrayList<>(events);
        events.clear();
        return result;
    }
}

// 在应用服务中处理
public class OrderService {
    public void placeOrder(OrderId orderId) {
        Order order = repository.findById(orderId);
        order.place();
        
        // 获取并发布事件
        List<DomainEvent> events = order.getAndClearEvents();
        eventPublisher.publish(events);
        
        repository.save(order);
    }
}
```

### 3.5.3 事件溯源（Event Sourcing）与CQRS
**事件溯源**：存储实体状态变化的事件序列，而不是当前状态
- 优势：完整的审计记录、时间点回溯能力、更好的并发控制
- 挑战：查询复杂性、学习曲线高、最终一致性

**CQRS**（命令查询责任分离）：将读操作和写操作分离为不同的模型
- 读模型：优化查询性能，可能直接映射到DTO
- 写模型：关注领域规则和一致性，通常是充血模型

CQRS与事件溯源结合使用时，事件可用于更新读模型（投影）。

### 3.5.4 领域事件的发布与订阅机制实现
常见实现方式：
1. **同步发布**：适用于单体应用
   ```java
   public class SimpleEventBus {
       private Map<Class<? extends DomainEvent>, List<EventHandler>> handlers = new HashMap<>();
       
       public void publish(DomainEvent event) {
           handlers.getOrDefault(event.getClass(), Collections.emptyList())
                   .forEach(handler -> handler.handle(event));
       }
   }
   ```

2. **异步发布**：使用消息队列（Kafka、RabbitMQ等）
   ```java
   public class AsyncEventPublisher {
       private final KafkaTemplate<String, DomainEvent> kafkaTemplate;
       
       public void publish(DomainEvent event) {
           kafkaTemplate.send("domain-events", event.getEventId().toString(), event);
       }
   }
   ```

3. **事务性发布**：确保事件与状态变更在同一事务中
   - 使用事务性消息表
   - 使用CDC（变更数据捕获）工具如Debezium
   - 使用Outbox模式

## 3.6 资源库（Repository）

### 3.6.1 资源库的设计与职责
资源库提供访问聚合的抽象，隐藏持久化细节。主要职责：
- 提供对聚合的检索（通过ID或条件）
- 保存聚合状态
- 封装持久化逻辑

```java
public interface OrderRepository {
    Order findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    void save(Order order);
    void remove(Order order);
}
```

### 3.6.2 资源库与工厂的区别
- 工厂：关注对象的创建
- 资源库：关注对象的获取和存储

资源库通常使用工厂来创建对象，特别是从持久化存储中重构对象时。

### 3.6.3 面向集合的资源库与面向持久化的资源库
**面向集合的资源库**：
- 模拟内存中集合操作
- 领域层友好的抽象
- 不暴露查询语言或持久化细节

**面向持久化的资源库**：
- 提供更多查询能力（如分页、排序）
- 可能暴露一些持久化概念
- 通常在基础设施层实现

```java
// 面向集合的资源库接口（领域层）
public interface UserRepository {
    User findById(UserId id);
    List<User> findAll();
    void save(User user);
    void remove(User user);
}

// 面向持久化的资源库实现（基础设施层）
public class JpaUserRepository implements UserRepository {
    private final UserJpaRepository jpaRepository; // Spring Data接口
    
    @Override
    public User findById(UserId id) {
        return jpaRepository.findById(id.getValue())
                .map(this::toDomain)
                .orElseThrow(() -> new EntityNotFoundException("用户不存在"));
    }
    
    // 其他方法实现...
}
```

### 3.6.4 资源库的测试与模拟
1. **单元测试**：使用内存实现或模拟
   ```java
   public class InMemoryUserRepository implements UserRepository {
       private Map<UserId, User> users = new HashMap<>();
       
       @Override
       public User findById(UserId id) {
           User user = users.get(id);
           if (user == null) {
               throw new EntityNotFoundException("用户不存在");
           }
           return user;
       }
       
       // 其他方法实现...
   }
   ```

2. **集成测试**：使用测试数据库
   - 可使用H2等内存数据库
   - 或使用TestContainers提供真实数据库环境

3. **测试替身（Test Double）**：
   - 使用Mockito等工具模拟资源库行为
   - 验证与资源库的交互

## 3.7 工厂（Factory）

### 3.7.1 工厂的应用场景
当对象创建过程复杂时使用工厂：
- 需要协调多个对象创建
- 创建逻辑包含业务规则
- 需要隐藏创建细节
- 从持久化存储重构对象

### 3.7.2 工厂方法、抽象工厂在DDD中的应用
**工厂方法**：实体或值对象中的静态创建方法

```java
public class Order {
    // 工厂方法
    public static Order create(CustomerId customerId) {
        Order order = new Order(new OrderId(), customerId, OrderStatus.DRAFT);
        // 其他初始化逻辑
        return order;
    }
    
    private Order(OrderId id, CustomerId customerId, OrderStatus status) {
        // 构造逻辑
    }
}
```

**领域工厂类**：独立的工厂类

```java
public class OrderFactory {
    public Order createOrder(CustomerId customerId) {
        OrderId orderId = orderIdGenerator.nextId();
        return new Order(orderId, customerId, OrderStatus.DRAFT);
    }
    
    public Order rebuildFromStorage(OrderData data) {
        // 从存储数据重构领域对象
    }
}
```

**抽象工厂**：在需要创建相关对象族时使用

```java
public interface PaymentFactory {
    Payment createPayment(Money amount);
    Receipt createReceipt(Payment payment);
}

public class CreditCardPaymentFactory implements PaymentFactory {
    // 实现方法
}

public class WeChatPaymentFactory implements PaymentFactory {
    // 实现方法
}
```

### 3.7.3 复杂对象创建的封装方法
1. **分步骤构建**：使用Builder模式
   ```java
   public class UserBuilder {
       private UserId id;
       private String name;
       private String email;
       private List<Role> roles = new ArrayList<>();
       
       public UserBuilder withId(UserId id) {
           this.id = id;
           return this;
       }
       
       // 其他withXxx方法
       
       public User build() {
           // 验证必要条件
           if (id == null) {
               throw new IllegalStateException("用户ID不能为空");
           }
           
           User user = new User(id, name, email);
           roles.forEach(user::addRole);
           return user;
       }
   }
   ```

2. **使用规范模式**：封装创建规则
   ```java
   public class ProductCreationSpecification {
       public boolean isSatisfiedBy(ProductData data) {
           return data.getName() != null && !data.getName().isEmpty() &&
                  data.getPrice() != null && data.getPrice().compareTo(BigDecimal.ZERO) > 0;
       }
       
       public Product createProduct(ProductData data) {
           if (!isSatisfiedBy(data)) {
               throw new IllegalArgumentException("产品数据不满足创建条件");
           }
           
           return new Product(
               new ProductId(),
               data.getName(),
               new Money(data.getPrice(), Currency.getInstance("CNY"))
           );
       }
   }
   ```

## 3.8 面试要点

### 3.8.1 实体与值对象的选择依据与案例分析
面试问题示例：
- 如何判断一个领域概念应该建模为实体还是值对象？
- 什么情况下值对象会转变为实体？反之呢？

解答要点：
- 关注概念的本质特性：需要追踪身份变化的是实体，只关注属性的是值对象
- 使用业务语言描述："我们需要知道这是同一个X"暗示实体；"我们只关心X的价值/特性"暗示值对象
- 提供具体案例：如银行账户（实体）vs货币金额（值对象）

### 3.8.2 聚合设计的常见陷阱与解决方案
面试问题示例：
- 聚合设计中最常见的错误是什么？
- 如何处理需要跨多个聚合的一致性需求？

解答要点：
- 过大聚合陷阱：导致性能问题和并发冲突
  - 解决：拆分为多个小聚合，使用最终一致性
- 直接引用陷阱：聚合间直接引用对象而非ID
  - 解决：始终通过ID引用其他聚合
- 事务边界混乱：一个事务修改多个聚合
  - 解决：重新划分聚合边界或使用领域事件

### 3.8.3 领域事件驱动设计的实现方法
面试问题示例：
- 如何在不引入重量级框架的情况下实现领域事件？
- 领域事件如何解决分布式系统的数据一致性问题？

解答要点：
- 实现方法：
  - 简单方案：聚合记录事件，应用服务发布
  - 高级方案：结合事务性发布和消息队列
- 一致性保证：
  - Outbox模式确保事件与状态变更原子性
  - 事件溯源提供完整的状态变更历史

### 3.8.4 资源库的设计模式与实践
面试问题示例：
- 资源库模式如何提高系统的可测试性？
- 如何处理复杂的查询需求而不破坏DDD的原则？

解答要点：
- 可测试性：
  - 通过接口隔离，便于创建测试替身
  - 内存实现简化单元测试
- 复杂查询：
  - 使用规范模式（Specification）封装查询条件
  - 对于复杂报表查询，考虑CQRS分离查询模型
  - 查询DTO直接映射到视图需求 