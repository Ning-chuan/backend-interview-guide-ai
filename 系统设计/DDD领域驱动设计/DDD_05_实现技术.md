# 第5章：DDD实现技术

DDD（领域驱动设计）作为一种软件开发方法论，其理论与实践紧密结合。本章将详细探讨如何在实际项目中落地DDD概念，选择合适的技术栈和实现方案。

## 5.1 Java实现DDD的技术栈

### Spring框架在DDD中的应用

#### Spring IoC与DI支持DDD分层架构
- **分层架构映射**：Spring IoC容器可以优雅地管理DDD中的各个层（表示层、应用层、领域层、基础设施层）
- **实现示例**：
```java
// 领域层 - 领域服务
@Service
@DomainService
public class OrderDomainServiceImpl implements OrderDomainService {
    // 领域逻辑实现
}

// 应用层 - 应用服务
@Service
@ApplicationService
public class OrderApplicationServiceImpl implements OrderApplicationService {
    @Autowired
    private OrderDomainService orderDomainService;
    
    // 应用服务逻辑
}
```
- **最佳实践**：使用自定义注解区分不同层次的服务，增强代码的表达能力

#### Spring AOP实现横切关注点
- **典型应用场景**：
  - 领域事件发布
  - 聚合事务管理
  - 权限检查
  - 日志与性能监控
- **实现示例**：
```java
@Aspect
@Component
public class DomainEventPublishAspect {
    @Autowired
    private DomainEventPublisher eventPublisher;
    
    @AfterReturning("@annotation(PublishDomainEvent)")
    public void publishEvent(JoinPoint joinPoint) {
        // 捕获并发布领域事件
    }
}
```

### 持久化技术选择

#### JPA/Hibernate与DDD的整合
- **优势**：对象-关系映射自然契合DDD实体与值对象概念
- **实现方式**：
  - 使用@Entity注解映射实体
  - 利用@Embeddable实现值对象
  - 通过@OneToMany等关系注解表达聚合内关系
- **代码示例**：
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private OrderId id;
    
    @Embedded
    private CustomerInfo customerInfo;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private Set<OrderItem> orderItems = new HashSet<>();
    
    // 领域行为...
}

@Embeddable
public class CustomerInfo {
    private String name;
    private String email;
    private String phone;
    
    // 构造函数和封装的值对象行为...
}
```
- **挑战与解决方案**：
  - 聚合映射复杂性：合理使用懒加载和关联策略
  - 性能问题：合理设计查询和索引

#### MyBatis在DDD中的应用模式
- **使用场景**：复杂查询、性能敏感场景、遗留系统整合
- **DDD友好实践**：
  - 资源库接口定义在领域层
  - 实现类位于基础设施层
  - 使用XML配置或注解灵活映射
- **代码示例**：
```java
// 领域层定义资源库接口
public interface OrderRepository {
    Order findById(OrderId orderId);
    void save(Order order);
}

// 基础设施层实现
@Repository
public class MyBatisOrderRepository implements OrderRepository {
    @Autowired
    private OrderMapper orderMapper;
    
    @Override
    public Order findById(OrderId orderId) {
        OrderDO orderDO = orderMapper.findById(orderId.getValue());
        return orderAssembler.toDomain(orderDO);
    }
    
    @Override
    public void save(Order order) {
        OrderDO orderDO = orderAssembler.toData(order);
        if (orderDO.getId() == null) {
            orderMapper.insert(orderDO);
        } else {
            orderMapper.update(orderDO);
        }
    }
}
```

#### Spring Data的使用
- **简化资源库实现**：提供Repository接口自动生成实现
- **DDD友好功能**：
  - 自定义方法名称生成查询
  - 基于规范（Specification）的动态查询
  - 支持分页和排序
- **实现示例**：
```java
public interface OrderRepository extends JpaRepository<Order, OrderId> {
    // 根据方法名自动生成查询
    List<Order> findByCustomerInfoEmail(String email);
    
    // 使用@Query提供复杂查询
    @Query("SELECT o FROM Order o JOIN o.orderItems i WHERE i.productId = :productId")
    List<Order> findOrdersContainingProduct(@Param("productId") ProductId productId);
}
```

### 事件驱动技术

#### Spring事件机制
- **适用场景**：同进程内的领域事件处理
- **实现方式**：
  - ApplicationEventPublisher发布事件
  - @EventListener注解处理事件
- **代码示例**：
```java
// 领域事件定义
public class OrderCreatedEvent extends DomainEvent {
    private final OrderId orderId;
    // 事件数据和getter
}

// 在领域模型中发布事件
@DomainService
public class OrderService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public Order createOrder(/*参数*/) {
        // 创建订单领域逻辑
        Order order = new Order(/*参数*/);
        
        // 发布领域事件
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));
        return order;
    }
}

// 事件监听器
@Component
public class OrderCreatedEventHandler {
    @EventListener
    public void handle(OrderCreatedEvent event) {
        // 处理订单创建事件
    }
}
```

#### 消息中间件在领域事件中的应用
- **技术选择**：
  - Kafka：高吞吐量、持久化事件流
  - RabbitMQ：灵活路由、多种交换类型
  - RocketMQ/Pulsar：国内广泛应用
- **实现策略**：
  - 事务性发件箱模式（Transactional Outbox）
  - 消息幂等性处理
  - 事件版本管理
- **代码示例**：
```java
@Service
public class DomainEventPublisher {
    @Autowired
    private KafkaTemplate<String, DomainEvent> kafkaTemplate;
    
    @Transactional
    public void publish(DomainEvent event) {
        // 1. 保存事件到事件表（事务性发件箱）
        eventRepository.save(event);
        
        // 2. 发送到Kafka
        kafkaTemplate.send("domain-events", event.getAggregateId(), event);
    }
}

// 消费端
@KafkaListener(topics = "domain-events")
public void handleOrderEvents(DomainEvent event) {
    if (event instanceof OrderCreatedEvent) {
        // 处理订单创建事件
    }
}
```

## 5.2 实体与值对象的Java实现

### 实体的代码实现模式

#### 标识符生成策略
- **UUID策略**：
  - 优势：分布式生成，无需中央协调
  - 缺点：索引效率低，无序
  - 实现：`UUID.randomUUID()`
- **数据库序列**：
  - 优势：有序，便于索引
  - 缺点：中央依赖，多数据中心挑战
  - 实现：JPA的`@GeneratedValue(strategy = GenerationType.SEQUENCE)`
- **Snowflake算法**：
  - 优势：分布式高性能，有序
  - 组成：时间戳 + 工作机器ID + 序列号
  - 实现示例：
```java
public class SnowflakeIdGenerator {
    private final long workerId;
    private final long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;
    
    // 各部分占位定义
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long sequenceBits = 12L;
    
    // 实现方法...
    
    public synchronized long nextId() {
        long timestamp = timeGen();
        // 时钟回拨处理
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("Clock moved backwards");
        }
        
        // 同一毫秒内序列递增
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }
        
        lastTimestamp = timestamp;
        
        // 组合ID
        return ((timestamp - twepoch) << timestampLeftShift) |
               (datacenterId << datacenterIdShift) |
               (workerId << workerIdShift) |
               sequence;
    }
}
```
- **领域特定标识符**：
  - 优势：携带业务含义
  - 实现：前缀 + 时间戳 + 序列号
  - 示例：订单号 ORDER20230615001

#### 实体相等性判断
- **基于ID的相等性**：
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    
    Order order = (Order) o;
    return id != null && id.equals(order.id);
}

@Override
public int hashCode() {
    return id != null ? id.hashCode() : 0;
}
```
- **注意事项**：
  - 处理新建实体（ID为null）的相等性
  - 关注类层次结构中的equals行为
  - 确保hashCode与equals一致性

### 值对象的实现技巧

#### 不可变对象设计
- **实现方式**：
  - 所有字段final
  - 不提供setter方法
  - 返回新对象而非修改现有对象
- **代码示例**：
```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    
    // 不可变操作 - 返回新对象
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add money with different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    // Getter但没有Setter
    public BigDecimal getAmount() {
        return amount;
    }
    
    public Currency getCurrency() {
        return currency;
    }
    
    // 值对象相等性 - 基于所有属性
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 && 
               currency.equals(money.currency);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}
```

#### Java 16+ Record的应用
- **简化值对象实现**：自动生成equals、hashCode和toString
- **示例**：
```java
public record Address(
    String street,
    String city,
    String zipCode,
    String country
) {
    // 构造函数可以添加验证
    public Address {
        Objects.requireNonNull(street, "Street cannot be null");
        Objects.requireNonNull(city, "City cannot be null");
        Objects.requireNonNull(zipCode, "Zip code cannot be null");
        Objects.requireNonNull(country, "Country cannot be null");
    }
    
    // 可以添加业务方法
    public boolean isSameCountry(Address other) {
        return country.equals(other.country);
    }
}
```
- **限制**：
  - 所有字段必须在构造函数中初始化
  - 不能继承其他类（但可以实现接口）

#### 值对象集合的处理
- **不可变集合**：使用Guava或Java Collections的不可变集合
```java
public class Order {
    private final Set<OrderItem> orderItems;
    
    public Order() {
        this.orderItems = new HashSet<>();
    }
    
    public Set<OrderItem> getOrderItems() {
        return Collections.unmodifiableSet(orderItems);
    }
    
    public void addOrderItem(OrderItem item) {
        orderItems.add(item);
    }
}
```
- **集合操作封装**：将集合操作封装在实体方法内
- **防御性复制**：避免集合引用泄露
- **自定义集合类**：实现特定业务规则的集合类

## 5.3 聚合与资源库实现

### 聚合根的设计与实现

#### 聚合内一致性规则实现
- **规则验证**：
  - 聚合根负责验证内部一致性
  - 业务规则集中在领域实体中
- **示例实现**：
```java
public class Order {
    private OrderId id;
    private OrderStatus status;
    private CustomerId customerId;
    private Set<OrderItem> orderItems = new HashSet<>();
    private Money totalAmount;
    
    // 聚合根方法确保一致性
    public void addOrderItem(Product product, int quantity) {
        // 业务规则验证
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Can only add items to draft orders");
        }
        
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        
        // 创建订单项
        OrderItem newItem = new OrderItem(new OrderItemId(), id, product.getId(), 
                                         product.getPrice(), quantity);
                                         
        // 更新聚合内状态
        orderItems.add(newItem);
        recalculateTotalAmount();
    }
    
    // 内部一致性方法
    private void recalculateTotalAmount() {
        this.totalAmount = orderItems.stream()
            .map(item -> item.getUnitPrice().multiply(item.getQuantity()))
            .reduce(Money.ZERO, Money::add);
    }
}
```

#### 聚合间引用与边界控制
- **引用策略**：
  - 按ID引用其他聚合
  - 避免直接对象引用导致的聚合边界模糊
- **代码示例**：
```java
// 不正确 - 直接引用其他聚合
public class Order {
    private Customer customer; // 错误：直接引用聚合
}

// 正确 - 通过ID引用
public class Order {
    private CustomerId customerId; // 正确：通过ID引用
}
```
- **领域事件实现聚合间通信**：
```java
public class Order {
    private List<DomainEvent> domainEvents = new ArrayList<>();
    
    public void place() {
        // 业务逻辑
        this.status = OrderStatus.PLACED;
        
        // 记录领域事件
        domainEvents.add(new OrderPlacedEvent(this.id, this.customerId));
    }
    
    public List<DomainEvent> getDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
}
```

### 资源库模式的Java实现

#### 领域资源库接口设计
- **资源库接口原则**：
  - 位于领域层
  - 体现领域概念
  - 隐藏持久化细节
- **常见方法**：
```java
// 领域层接口
public interface OrderRepository {
    Order findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    void save(Order order);
    void remove(Order order);
    
    // 集合操作
    List<Order> findByStatus(OrderStatus status, int page, int size);
    
    // 规范模式查询
    List<Order> findBySpecification(Specification<Order> spec);
}
```

#### ORM映射策略
- **单表继承**：
  - 特点：将整个继承层次结构映射到单个表
  - 实现：`@Inheritance(strategy = InheritanceType.SINGLE_TABLE)`
  - 适用：简单继承，较少字段差异
- **连接表继承**：
  - 特点：每个子类一个表，共享字段在父类表
  - 实现：`@Inheritance(strategy = InheritanceType.JOINED)`
  - 适用：深度继承，较多共享字段
- **表格每类继承**：
  - 特点：每个具体类一个完整表
  - 实现：`@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)`
  - 适用：较少使用多态查询

#### 乐观锁与并发控制
- **JPA版本控制**：
```java
@Entity
public class Order {
    @Id
    private OrderId id;
    
    @Version
    private Long version;
    
    // 其他字段和方法
}
```
- **手动版本控制**：
```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    @Override
    public void save(Order order) {
        OrderDO orderDO = orderAssembler.toData(order);
        int updated = orderMapper.updateWithVersion(orderDO);
        
        if (updated == 0) {
            throw new ConcurrencyException("Order was modified concurrently");
        }
        
        // 更新版本号
        order.incrementVersion();
    }
}
```
- **冲突解决策略**：
  - 最后提交者胜出
  - 合并变更（复杂但用户友好）
  - 提示用户手动解决

## 5.4 领域事件实现方案

### 同步事件vs异步事件
- **同步事件特点**：
  - 即时处理，保证事务一致性
  - 实现简单，调试方便
  - 存在耦合，影响性能
- **异步事件特点**：
  - 提高响应性能，支持高并发
  - 松耦合，故障隔离
  - 最终一致性挑战
- **实现对比**：
```java
// 同步事件
@Transactional
public Order placeOrder(PlaceOrderCommand command) {
    Order order = orderRepository.findById(command.getOrderId());
    order.place();
    orderRepository.save(order);
    
    // 同步发布和处理
    List<DomainEvent> events = order.getDomainEvents();
    for (DomainEvent event : events) {
        applicationEventPublisher.publishEvent(event);
    }
    
    return order;
}

// 异步事件
@Transactional
public Order placeOrder(PlaceOrderCommand command) {
    Order order = orderRepository.findById(command.getOrderId());
    order.place();
    orderRepository.save(order);
    
    // 异步发布
    List<DomainEvent> events = order.getDomainEvents();
    for (DomainEvent event : events) {
        eventOutbox.save(event); // 保存到发件箱表
    }
    
    return order;
}
```

### 事件存储与发布
- **事务性发件箱模式**：
  - 原理：事件与业务数据在同一事务中保存
  - 实现：额外的事件表 + 后台发布进程
```java
// 发件箱表结构
@Entity
@Table(name = "event_outbox")
public class EventOutbox {
    @Id
    private UUID id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    private String eventData; // JSON格式
    private LocalDateTime createdAt;
    private boolean published;
}

// 后台发布服务
@Service
public class EventPublisherService {
    @Scheduled(fixedRate = 1000)
    public void publishPendingEvents() {
        List<EventOutbox> pendingEvents = eventOutboxRepository
            .findByPublishedOrderByCreatedAsc(false, PageRequest.of(0, 100));
            
        for (EventOutbox event : pendingEvents) {
            try {
                kafkaTemplate.send("domain-events", event.getEventData());
                event.markAsPublished();
                eventOutboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish event: " + event.getId(), e);
                // 可实现重试策略
            }
        }
    }
}
```

### 事件溯源的实现技术
- **基本原理**：
  - 存储实体状态变化的事件序列而非当前状态
  - 通过重放事件重建实体状态
- **实现框架对比**：
  - **Event Store**：专用事件存储数据库
  - **Axon Framework**：完整DDD+CQRS+ES框架
  - **Eventuate**：微服务架构事件驱动框架
- **Axon Framework示例**：
```java
@Aggregate
public class Order {
    @AggregateIdentifier
    private OrderId id;
    private OrderStatus status;
    private CustomerId customerId;
    private Set<OrderItem> orderItems = new HashSet<>();
    
    @CommandHandler
    public Order(CreateOrderCommand command) {
        apply(new OrderCreatedEvent(command.getOrderId(), command.getCustomerId()));
    }
    
    @CommandHandler
    public void addItem(AddOrderItemCommand command) {
        // 验证
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Can only add items to draft orders");
        }
        
        apply(new OrderItemAddedEvent(id, command.getProductId(), 
                                      command.getPrice(), command.getQuantity()));
    }
    
    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.customerId = event.getCustomerId();
        this.status = OrderStatus.DRAFT;
    }
    
    @EventSourcingHandler
    public void on(OrderItemAddedEvent event) {
        OrderItem item = new OrderItem(
            new OrderItemId(), 
            event.getProductId(),
            event.getPrice(),
            event.getQuantity()
        );
        this.orderItems.add(item);
    }
}
```

## 5.5 CQRS与读写分离实现

### 命令模型与查询模型的设计
- **命令模型**：
  - 专注于业务规则和一致性
  - 富领域模型，完整聚合
  - 强一致性保障
- **查询模型**：
  - 针对展示和报表优化
  - 扁平化结构，便于查询
  - 可接受最终一致性
- **实现对比**：
```java
// 命令模型（写模型）
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private OrderId id;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Embedded
    private CustomerId customerId;
    
    @OneToMany(cascade = CascadeType.ALL)
    private Set<OrderItem> items;
    
    // 领域行为...
}

// 查询模型（读模型）
@Entity
@Table(name = "order_summaries")
public class OrderSummary {
    @Id
    private String orderId;
    private String customerName;
    private String status;
    private BigDecimal totalAmount;
    private int itemCount;
    private LocalDateTime createdAt;
    
    // 只有getter，没有业务逻辑
}
```

### 数据同步策略：即时同步vs最终一致性
- **即时同步**：
  - 实现：同一事务内更新
  - 优点：强一致性，简单直接
  - 缺点：性能开销，增加耦合
- **最终一致性**：
  - 实现：基于事件的异步更新
  - 优点：高性能，松耦合
  - 缺点：一致性延迟，复杂度高
- **代码示例**：
```java
// 基于事件的异步同步
@Service
public class OrderProjectionService {
    @Autowired
    private OrderSummaryRepository orderSummaryRepository;
    
    @EventListener
    public void on(OrderCreatedEvent event) {
        OrderSummary summary = new OrderSummary();
        summary.setOrderId(event.getOrderId().toString());
        summary.setCustomerName(event.getCustomerName());
        summary.setStatus("CREATED");
        summary.setCreatedAt(LocalDateTime.now());
        orderSummaryRepository.save(summary);
    }
    
    @EventListener
    public void on(OrderStatusChangedEvent event) {
        OrderSummary summary = orderSummaryRepository.findById(event.getOrderId().toString())
            .orElseThrow(() -> new IllegalStateException("Order summary not found"));
        summary.setStatus(event.getNewStatus().toString());
        orderSummaryRepository.save(summary);
    }
}
```

### 读写分离的实现方案
- **单数据库读写分离**：
  - 实现：同一数据库不同表或视图
  - 优点：简单实用，适合中小规模
  - 缺点：数据库成为瓶颈
- **多数据库CQRS模式**：
  - 实现：命令库（关系型）+ 查询库（文档型/列式/搜索引擎）
  - 优点：极高扩展性，针对查询优化
  - 缺点：系统复杂度较高
  - 常见组合：
    - PostgreSQL（命令）+ Elasticsearch（查询）
    - MySQL（命令）+ MongoDB（查询）
    - SQL Server（命令）+ Redis（查询缓存）
- **架构示例**：
```
  用户命令                        用户查询
     │                               │
     ▼                               ▼
┌──────────┐                   ┌──────────┐
│ 命令API  │                   │ 查询API  │
└────┬─────┘                   └────┬─────┘
     │                               │
     ▼                               ▼
┌──────────┐                   ┌──────────┐
│ 应用服务 │                   │查询服务  │
└────┬─────┘                   └────┬─────┘
     │         ┌─────────┐          │
     │         │ 事件总线│          │
     │         └────┬────┘          │
     │              │               │
     ▼              ▼               ▼
┌──────────┐    ┌──────────┐   ┌──────────┐
│ 命令数据库│───►│ 事件存储 │───►│ 查询数据库│
└──────────┘    └──────────┘   └──────────┘
```

## 5.6 测试策略

### 领域模型单元测试
- **测试重点**：
  - 实体业务规则
  - 值对象不变量
  - 领域服务逻辑
- **测试工具**：JUnit 5, Mockito
- **示例**：
```java
class OrderTest {
    @Test
    void shouldAddItemToOrder() {
        // Given
        Order order = new Order(new OrderId(), new CustomerId());
        Product product = new Product(new ProductId(), "Test Product", 
                                      new Money(BigDecimal.TEN, Currency.USD));
        
        // When
        order.addOrderItem(product, 2);
        
        // Then
        assertEquals(1, order.getOrderItems().size());
        OrderItem item = order.getOrderItems().iterator().next();
        assertEquals(product.getId(), item.getProductId());
        assertEquals(2, item.getQuantity());
        assertEquals(new Money(new BigDecimal("20"), Currency.USD), order.getTotalAmount());
    }
    
    @Test
    void shouldNotAddItemToPlacedOrder() {
        // Given
        Order order = new Order(new OrderId(), new CustomerId());
        order.place();
        Product product = new Product(new ProductId(), "Test Product", 
                                      new Money(BigDecimal.TEN, Currency.USD));
        
        // Then
        assertThrows(IllegalStateException.class, () -> {
            order.addOrderItem(product, 2);
        });
    }
}
```

### 聚合与资源库测试
- **测试方式**：
  - 使用内存实现测试资源库
  - 基于H2/HSQLDB的集成测试
  - 测试数据容器（Testcontainers）
- **代码示例**：
```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
class OrderRepositoryTest {
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private OrderJpaRepository orderRepository;
    
    @Test
    void shouldSaveAndFindOrder() {
        // Given
        Order order = new Order(new OrderId(), new CustomerId());
        Product product = new Product(new ProductId(), "Test Product", 
                                      new Money(BigDecimal.TEN, Currency.USD));
        order.addOrderItem(product, 2);
        
        // When
        orderRepository.save(order);
        entityManager.flush();
        entityManager.clear();
        
        // Then
        Optional<Order> found = orderRepository.findById(order.getId());
        assertTrue(found.isPresent());
        assertEquals(1, found.get().getOrderItems().size());
        assertEquals(new Money(new BigDecimal("20"), Currency.USD), 
                     found.get().getTotalAmount());
    }
}
```

### 应用服务集成测试
- **测试重点**：
  - 事务行为
  - 应用服务协作
  - 事件发布
- **Spring测试工具**：
  - @SpringBootTest
  - @Transactional
  - ApplicationEvents
- **示例**：
```java
@SpringBootTest
class OrderApplicationServiceTest {
    @Autowired
    private OrderApplicationService orderService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @SpyBean
    private ApplicationEventPublisher eventPublisher;
    
    @Test
    void shouldPlaceOrderAndPublishEvent() {
        // Given
        CreateOrderCommand createCmd = new CreateOrderCommand(new CustomerId());
        OrderId orderId = orderService.createOrder(createCmd);
        
        AddOrderItemCommand addItemCmd = new AddOrderItemCommand(
            orderId, 
            new ProductId(),
            new Money(BigDecimal.TEN, Currency.USD),
            2
        );
        orderService.addOrderItem(addItemCmd);
        
        // When
        PlaceOrderCommand placeCmd = new PlaceOrderCommand(orderId);
        orderService.placeOrder(placeCmd);
        
        // Then
        Order order = orderRepository.findById(orderId);
        assertEquals(OrderStatus.PLACED, order.getStatus());
        
        verify(eventPublisher).publishEvent(any(OrderPlacedEvent.class));
    }
}
```

### BDD在DDD中的应用
- **工具**：Cucumber, JBehave
- **应用场景**：验收测试，特性开发
- **示例**：
```gherkin
Feature: Order Management

  Scenario: Customer places an order
    Given a customer with id "C12345"
    And a product "P1" with price $10.00
    When the customer creates a new order
    And adds 2 units of product "P1" to the order
    And places the order
    Then the order should be in "PLACED" status
    And the order total should be $20.00
    And a "OrderPlaced" event should be published
```

## 5.7 面试要点

### Spring框架与DDD结合的最佳实践
- **关键问题**：
  - 如何使用Spring管理DDD分层架构？
  - 依赖倒置如何通过Spring IoC实现？
  - 说明一个你设计过的DDD项目中的Spring配置结构
- **回答要点**：
  - 使用自定义注解区分不同层（@DomainService, @ApplicationService）
  - 将资源库接口放在领域层，实现在基础设施层
  - 领域事件与Spring事件机制的整合方案
  - 分包策略：按领域模块横向分包，每个包内按层纵向分包

### 领域事件异步处理的实现方案
- **关键问题**：
  - 如何保证事件发布的可靠性？
  - 如何处理事件重复和顺序问题？
  - 在分布式系统中如何追踪事件相关性？
- **回答要点**：
  - 事务性发件箱模式详解
  - 幂等性处理策略：乐观锁、幂等键、状态检查
  - 事件ID与关联ID策略（命令ID、聚合ID、流程ID）
  - 问题处理：重试策略、死信队列、监控告警

### 如何解决ORM阻抗不匹配问题
- **关键问题**：
  - 领域模型与关系模型的主要差异是什么？
  - 如何处理复杂的领域对象映射？
  - 如何避免性能问题？
- **回答要点**：
  - 使用防腐层隔离领域模型和ORM映射
  - 聚合边界控制策略（延迟加载、级联策略选择）
  - 值对象映射方案（@Embeddable vs JSON存储）
  - 性能优化策略：查询优化、批处理、缓存策略

### 测试驱动开发在DDD中的应用策略
- **关键问题**：
  - 如何为领域模型编写有效的单元测试？
  - 测试驱动设计如何帮助发现领域模型？
  - 如何结合BDD与DDD？
- **回答要点**：
  - 由内而外的测试策略：先领域模型，再应用服务
  - 测试粒度的选择：领域行为 vs 用例场景
  - 测试设计模式：Test Data Builder, Object Mother
  - BDD助力领域通用语言：从Gherkin场景到领域模型 