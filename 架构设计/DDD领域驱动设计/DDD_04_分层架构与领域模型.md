# 第4章：分层架构与领域模型

## 4.1 DDD经典四层架构

DDD推崇的经典四层架构是一种将软件系统按功能职责划分的架构模式，目的是实现关注点分离，提高系统的可维护性和可扩展性。

### 表示层（Presentation Layer）
- **核心职责**：负责向用户展示信息和解释用户指令
- **主要组件**：控制器(Controller)、视图(View)、前端页面、API接口
- **技术实现**：Web MVC框架、RESTful API、GraphQL等
- **设计要点**：
  - 不包含业务逻辑，仅负责用户交互
  - 将用户请求转换为应用层可理解的命令或查询
  - 将应用层返回的数据转换为适合展示的格式

### 应用层（Application Layer）
- **核心职责**：定义软件要完成的任务，协调领域对象执行
- **主要组件**：应用服务(Application Service)、DTO、装配器(Assembler)
- **技术实现**：服务类、事务脚本、用例实现
- **设计要点**：
  - 是领域层的直接客户端，组织工作流程
  - 不包含业务规则，只协调领域对象完成任务
  - 通常是事务边界的定义处
  - 提供粗粒度的操作接口给表示层

### 领域层（Domain Layer）
- **核心职责**：表达业务概念、业务状态和业务规则
- **主要组件**：实体(Entity)、值对象(Value Object)、聚合(Aggregate)、领域服务(Domain Service)
- **技术实现**：领域模型类、业务规则实现、领域事件
- **设计要点**：
  - DDD的核心，封装核心业务逻辑和规则
  - 不依赖其他层，保持业务纯粹性
  - 使用领域语言(Ubiquitous Language)构建模型
  - 业务规则应内聚于领域对象中

### 基础设施层（Infrastructure Layer）
- **核心职责**：为上层提供通用的技术能力
- **主要组件**：仓储实现(Repository Impl)、ORM映射、消息队列、缓存、日志等
- **技术实现**：数据库访问组件、第三方服务集成、技术框架等
- **设计要点**：
  - 支持其他层的技术需求
  - 实现领域层定义的接口(如仓储接口)
  - 处理技术细节，屏蔽外部系统复杂性
  - 适配外部系统和框架

### 层间依赖原则
- 依赖方向：从上至下（表示层→应用层→领域层→基础设施层）
- 依赖倒置：领域层定义接口，基础设施层实现接口
- 跨层调用：通过依赖注入或服务定位器模式实现

### 实例图示
```
┌─────────────────┐
│   表示层        │ ←── 用户请求
├─────────────────┤
│   应用层        │ ←── 用例协调
├─────────────────┤
│   领域层        │ ←── 业务核心
├─────────────────┤
│   基础设施层    │ ←── 技术支持
└─────────────────┘
```

## 4.2 领域层设计深入

领域层是DDD的核心，良好的领域层设计是实现复杂业务逻辑的关键。

### 领域层内部结构设计
- **实体(Entity)**
  - 具有唯一标识的对象，如用户、订单、商品
  - 生命周期中状态可变，但标识保持不变
  - 关注点：身份标识、状态变化、业务行为

  ```java
  public class Order {
      private OrderId id;
      private Customer customer;
      private List<OrderItem> items;
      private OrderStatus status;
      
      public void addItem(Product product, int quantity) {
          // 业务逻辑验证与处理
          items.add(new OrderItem(product, quantity));
      }
      
      public void confirm() {
          // 确认订单的业务规则
          this.status = OrderStatus.CONFIRMED;
      }
  }
  ```

- **值对象(Value Object)**
  - 无唯一标识，通过属性值判断相等性
  - 不可变对象，修改即替换
  - 关注点：属性组合、度量描述、计算逻辑
  
  ```java
  public class Money {
      private final BigDecimal amount;
      private final Currency currency;
      
      // 构造函数设值后不可变
      
      public Money add(Money other) {
          // 返回新对象，不修改原对象
          return new Money(this.amount.add(other.amount), this.currency);
      }
  }
  ```

- **聚合(Aggregate)**
  - 由根实体和内部关联对象组成的业务整体
  - 确保业务一致性的边界
  - 关注点：事务边界、数据一致性、业务完整性
  
  ```java
  // Order可作为聚合根，OrderItem只能通过Order访问
  public class Order {
      // 聚合根实体
      private List<OrderItem> items; // 聚合内部实体
      
      // 外部只能通过聚合根添加项目
      public void addItem(Product product, int quantity) {
          // 业务规则检查
          items.add(new OrderItem(this, product, quantity));
      }
  }
  ```

- **领域服务(Domain Service)**
  - 封装不属于单个实体或值对象的领域逻辑
  - 无状态，操作多个领域对象
  - 关注点：跨实体业务规则、复杂计算逻辑
  
  ```java
  public class PricingService {
      public Money calculateOrderTotal(Order order, DiscountPolicy discountPolicy) {
          // 复杂定价计算逻辑
          // 涉及订单、折扣政策等多个对象
      }
  }
  ```

### 业务规则与约束的表达方式
- **不变量(Invariant)**：业务规则确保对象始终处于有效状态
  - 通过构造函数、方法前置条件验证实现
  - 确保实体或聚合的内部一致性
  
- **领域事件(Domain Event)**：捕获领域中发生的重要事件
  - 记录状态变更
  - 触发后续业务流程
  - 支持事件溯源
  
  ```java
  public class OrderConfirmedEvent {
      private final OrderId orderId;
      private final LocalDateTime confirmedAt;
      
      // 订单确认事件，可用于触发后续流程
  }
  ```

- **规约(Specification)**：将业务规则封装为可复用对象
  - 将复杂条件判断封装为独立对象
  - 提高规则复用性和可测试性
  
  ```java
  public interface OrderSpecification {
      boolean isSatisfiedBy(Order order);
  }
  
  public class CanBeShippedSpecification implements OrderSpecification {
      public boolean isSatisfiedBy(Order order) {
          return order.isConfirmed() 
              && order.isPaymentReceived() 
              && order.hasStock();
      }
  }
  ```

### 领域逻辑的封装策略
- **行为与数据内聚**：业务逻辑应与所操作的数据放在一起
- **丰富的领域模型**：实体和值对象包含行为，而非仅存储数据
- **意图表达**：方法命名应反映业务概念和操作意图
- **防御性编程**：在领域模型中保护业务规则，确保状态一致性

### 领域层与其他层的交互方式
- **仓储接口(Repository)**：在领域层定义接口，基础设施层实现
  ```java
  // 在领域层定义
  public interface OrderRepository {
      Order findById(OrderId id);
      void save(Order order);
  }
  ```

- **领域事件发布**：领域层发布事件，应用层或基础设施层处理
- **工厂(Factory)**：复杂对象创建逻辑的封装
- **防腐层(Anti-corruption Layer)**：保护领域模型不受外部系统影响

## 4.3 应用层设计模式

应用层是领域层与外部世界的桥梁，负责用例实现和流程协调。

### 应用服务的职责定位
- **定义业务用例**：将用户意图转化为系统行为
- **协调领域对象**：组织和协调领域对象完成业务流程
- **管理事务**：确保业务操作的原子性
- **权限检查**：验证操作权限
- **结果转换**：将领域对象转化为DTO返回

```java
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final EventPublisher eventPublisher;
    
    // 创建订单用例
    @Transactional
    public OrderDTO createOrder(CreateOrderCommand command) {
        // 获取领域对象
        Customer customer = customerRepository.findById(command.getCustomerId());
        
        // 创建领域对象
        Order order = new Order(customer);
        
        // 调用领域行为
        for (OrderItemCommand itemCommand : command.getItems()) {
            Product product = productRepository.findById(itemCommand.getProductId());
            order.addItem(product, itemCommand.getQuantity());
        }
        
        // 持久化
        orderRepository.save(order);
        
        // 发布事件
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));
        
        // 转换结果
        return orderAssembler.toDTO(order);
    }
}
```

### 命令（Command）与查询（Query）分离原则
- **CQS原则**：将方法分为命令(改变状态)和查询(返回结果)
- **命令处理**：执行改变系统状态的操作，无需返回数据
- **查询处理**：获取数据，不改变系统状态
- **优势**：提高系统可读性、可维护性和性能优化空间

```java
// 命令处理器
public class PlaceOrderCommandHandler {
    public void handle(PlaceOrderCommand command) {
        // 执行下单操作，改变系统状态
    }
}

// 查询处理器
public class OrderQueryService {
    public OrderDTO getOrderById(OrderId id) {
        // 仅查询订单，不改变状态
    }
}
```

### DTO的设计与使用
- **数据传输对象**：在应用层边界传输数据的对象
- **设计原则**：
  - 只包含需要传输的数据，无业务行为
  - 根据视图需求设计，可能合并多个领域对象数据
  - 防止领域对象泄露给表示层
- **装配器模式**：在领域对象与DTO之间进行转换

```java
// 输出DTO
public class OrderDTO {
    private String orderId;
    private String customerName;
    private List<OrderItemDTO> items;
    private BigDecimal totalAmount;
    // getter方法
}

// 装配器
public class OrderAssembler {
    public OrderDTO toDTO(Order order) {
        OrderDTO dto = new OrderDTO();
        dto.setOrderId(order.getId().toString());
        dto.setCustomerName(order.getCustomer().getName());
        // 转换其他字段
        return dto;
    }
}
```

### 应用层的事务控制
- **声明式事务**：使用AOP或注解方式定义事务边界
  ```java
  @Transactional
  public OrderDTO createOrder(CreateOrderCommand command) {
      // 整个方法在单一事务中执行
  }
  ```
- **事务设计策略**：
  - 为每个业务用例定义事务边界
  - 事务应包含完整的业务操作
  - 注意处理事务回滚和异常情况
  - 考虑跨聚合操作的一致性要求

## 4.4 六边形架构（端口与适配器）

六边形架构(Hexagonal Architecture)，又称为端口与适配器(Ports and Adapters)架构，是DDD实现的一种架构模式，由Alistair Cockburn提出。

### 六边形架构的核心思想
- **内外分离**：区分应用核心与外部世界
- **可替换性**：外部组件可以被轻松替换，不影响核心业务
- **可测试性**：业务核心可以独立测试，不依赖外部系统
- **关注点分离**：清晰区分业务逻辑和技术细节

### 端口与适配器的设计
- **端口(Port)**：应用核心定义的接口，代表与外部世界的交互点
  - **输入端口**：应用对外提供的服务(如应用服务接口)
  - **输出端口**：应用需要的外部服务(如仓储接口)

```java
// 输入端口示例
public interface OrderService {
    OrderDTO createOrder(CreateOrderCommand command);
    void cancelOrder(OrderId orderId);
}

// 输出端口示例
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
}
```

- **适配器(Adapter)**：连接外部世界和应用核心的组件
  - **输入适配器**：将外部请求转化为应用核心可处理的形式(如Controller)
  - **输出适配器**：实现输出端口，连接外部基础设施(如Repository实现)

```java
// 输入适配器示例
@RestController
public class OrderController {
    private final OrderService orderService;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderDTO> createOrder(@RequestBody OrderRequest request) {
        // 将HTTP请求适配为应用层命令
        CreateOrderCommand command = convertToCommand(request);
        // 调用应用层服务
        OrderDTO result = orderService.createOrder(command);
        return ResponseEntity.ok(result);
    }
}

// 输出适配器示例
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaEntityRepository jpaRepository;
    
    @Override
    public Order findById(OrderId id) {
        // 将JPA实体转换为领域对象
        OrderJpaEntity entity = jpaRepository.findById(id.toString())
            .orElseThrow(() -> new EntityNotFoundException());
        return convertToDomain(entity);
    }
}
```

### 内部架构与外部世界的隔离
- **模型隔离**：领域模型与外部概念隔离
  - 防腐层(ACL)阻止外部概念污染内部模型
  - 转换映射层处理内外数据结构转换
- **技术隔离**：业务逻辑不依赖具体技术实现
  - 依赖倒置原则(DIP)支持隔离
  - 面向接口编程，而非实现

### DDD与六边形架构的结合
- **领域层**：放置在六边形架构的中心
- **应用层**：包含输入端口(应用服务)和应用逻辑
- **适配器层**：
  - 输入适配器：对应表示层
  - 输出适配器：对应基础设施层实现
- **结合优势**：
  - 强化领域模型的核心地位
  - 清晰的依赖方向
  - 高度可测试性
  - 技术实现可替换性

## 4.5 CQRS架构模式

命令查询职责分离(Command Query Responsibility Segregation, CQRS)是一种将系统操作分为命令(写操作)和查询(读操作)两部分的架构模式。

### 命令查询职责分离（CQRS）原理
- **基本原则**：将修改状态的操作(命令)与返回数据的操作(查询)分离
- **命令端**：处理系统状态变更，基于领域模型，强一致性
- **查询端**：处理数据查询，基于优化的查询模型，高性能
- **数据流**：
  - 命令流：用户→命令→命令处理器→领域模型→事件→事件处理器
  - 查询流：用户→查询→查询处理器→查询模型→数据

```
┌───────────┐       ┌───────────┐
│  命令模型  │──────→│  领域事件  │
└───────────┘       └─────┬─────┘
      ↑                   │
      │                   ↓
┌───────────┐       ┌───────────┐
│  命令请求  │       │  查询模型  │
└───────────┘       └───────────┘
                          ↑
                          │
                    ┌───────────┐
                    │  查询请求  │
                    └───────────┘
```

### 读写模型分离的优势与成本
- **优势**：
  - 性能优化：查询模型可针对读取进行优化
  - 扩展性：读写可独立扩展
  - 复杂性管理：复杂领域模型与简单查询模型分离
  - 并发能力：读写分离降低锁竞争

- **成本**：
  - 增加系统复杂度
  - 数据一致性挑战(最终一致性)
  - 开发和维护两套模型
  - 学习曲线陡峭

### CQRS实现级别与复杂度
- **简单CQRS**：同一数据库，不同模型和服务
  - 命令服务使用领域模型
  - 查询服务直接访问数据库视图
  - 共享数据库保持强一致性

- **中级CQRS**：同一数据库，读库异步更新
  - 命令操作主库
  - 事件触发读模型更新
  - 读写数据库分离，但物理上可能是同一数据库

- **完全CQRS**：物理分离的读写数据库
  - 命令端使用事务型数据库(如关系型数据库)
  - 查询端使用优化的读取存储(如文档数据库、搜索引擎)
  - 通过事件处理异步更新查询数据库
  - 最终一致性

### 何时应用CQRS？何时不应该？
- **适合场景**：
  - 读写比例失衡(读多写少)
  - 复杂领域模型与简单查询需求并存
  - 团队规模较大，可分离开发
  - 对读取性能要求高
  - 需要高度可扩展性

- **不适合场景**：
  - 简单CRUD应用
  - 强一致性要求高的场景
  - 团队规模小，无法支持多模型维护
  - 业务变更频繁，难以同步维护两套模型
  - 项目周期短，投入回报不成正比

## 4.6 领域模型与数据模型

领域模型与数据模型是两种不同关注点的模型，二者的映射与协调是DDD实现的关键挑战。

### 领域模型与数据库模型的映射策略
- **直接映射**：领域对象结构与表结构一一对应
  - 优点：简单直观，ORM支持良好
  - 缺点：领域模型设计可能受数据库影响

- **聚合映射**：将聚合根与内部实体映射到表
  - 单表映射：整个聚合存储在一个表中(序列化或JSON字段)
  - 多表映射：聚合根与内部实体映射到不同表，通过外键关联

- **事件溯源**：存储领域事件而非当前状态
  - 不直接存储领域对象，而是存储改变对象状态的事件序列
  - 通过回放事件重建领域对象当前状态
  - 优点：完整历史记录，高度灵活性
  - 缺点：查询复杂度高，重建开销大

### ORM框架在DDD中的应用与限制
- **ORM优势**：
  - 简化对象-关系映射
  - 减少数据访问代码
  - 提供缓存、延迟加载等特性

- **ORM限制**：
  - 对象关系阻抗不匹配(Object-Relational Impedance Mismatch)
  - 复杂领域概念难以映射到关系模型
  - 性能开销，特别是N+1查询问题
  - 可能导致贫血模型的倾向

- **最佳实践**：
  - 将ORM作为技术实现细节，不要泄露到领域层
  - 使用仓储模式隔离ORM细节
  - 考虑使用领域事件进行聚合间的同步

```java
// JPA实体
@Entity
@Table(name = "orders")
public class OrderJpaEntity {
    @Id
    private String id;
    
    @ManyToOne
    private CustomerJpaEntity customer;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItemJpaEntity> items;
    
    // JPA需要的getter/setter
}

// 仓储实现，隔离ORM细节
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;
    
    @Override
    public Order findById(OrderId id) {
        OrderJpaEntity entity = jpaRepository.findById(id.getValue())
            .orElseThrow(() -> new EntityNotFoundException());
        return mapper.toDomain(entity);
    }
    
    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepository.save(entity);
    }
}
```

### 贫血模型vs充血模型
- **贫血模型(Anemic Domain Model)**：
  - 领域对象只包含数据，无业务行为
  - 业务逻辑位于服务层
  - 本质是过程式编程，违背OO原则
  - 常见于传统三层架构

```java
// 贫血模型
public class OrderEntity {
    private String id;
    private List<OrderItemEntity> items;
    // 只有getter/setter，无业务逻辑
}

// 服务包含所有业务逻辑
public class OrderService {
    public void addItem(OrderEntity order, ProductEntity product, int quantity) {
        // 业务逻辑在服务中实现
        OrderItemEntity item = new OrderItemEntity();
        item.setProductId(product.getId());
        item.setQuantity(quantity);
        order.getItems().add(item);
    }
}
```

- **充血模型(Rich Domain Model)**：
  - 领域对象包含数据和行为
  - 业务规则内聚于领域对象
  - 符合OO封装原则
  - DDD推崇的模型

```java
// 充血模型
public class Order {
    private OrderId id;
    private List<OrderItem> items;
    
    // 业务行为与数据内聚
    public void addItem(Product product, int quantity) {
        // 业务规则验证
        if (quantity <= 0) {
            throw new IllegalArgumentException("数量必须为正数");
        }
        
        // 检查是否已存在该商品
        for (OrderItem item : items) {
            if (item.getProduct().equals(product)) {
                item.increaseQuantity(quantity);
                return;
            }
        }
        
        // 添加新商品项
        items.add(new OrderItem(this, product, quantity));
    }
}
```

### 领域模型持久化的最佳实践
- **仓储模式(Repository Pattern)**：
  - 为领域对象提供集合式访问接口
  - 隐藏持久化细节
  - 支持领域对象的重建和持久化

- **持久化无关性**：
  - 领域模型不应依赖持久化技术
  - 使用依赖倒置原则，领域定义接口，基础设施实现

- **聚合边界考量**：
  - 持久化操作应尊重聚合边界
  - 以聚合为单位进行加载和保存
  - 避免跨聚合的事务一致性需求

- **性能优化策略**：
  - 延迟加载(Lazy Loading)
  - 预加载(Eager Loading)
  - 缓存策略
  - 读写分离

## 4.7 面试要点

### 分层架构中的依赖规则与依赖倒置
- **依赖规则**：
  - 上层依赖下层，不允许反向依赖
  - 每一层只能与直接相邻的下层通信
  - 领域层不应依赖其他层

- **依赖倒置原则(DIP)**：
  - 高层模块不应依赖低层模块，两者都应依赖抽象
  - 抽象不应依赖细节，细节应依赖抽象
  - 应用于领域层定义接口，基础设施层实现

```java
// 领域层定义接口
public interface UserRepository {
    User findById(UserId id);
    void save(User user);
}

// 基础设施层实现接口
public class MySqlUserRepository implements UserRepository {
    // 实现领域定义的接口...
}

// 应用层依赖接口，而非实现
public class UserService {
    private final UserRepository userRepository; // 依赖抽象
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

- **面试答题要点**：
  - 解释依赖倒置如何支持DDD中的领域模型独立性
  - 举例说明接口在领域层定义的具体实现
  - 分析不遵循依赖规则的后果

### 如何处理跨层调用与性能优化
- **跨层调用问题**：
  - 表示层直接访问领域层破坏分层架构
  - 过多层间转换可能导致性能问题
  - 复杂查询可能导致多次数据库访问

- **解决方案**：
  - 应用层提供丰富的DTO，减少跨层调用需求
  - 针对复杂查询场景，考虑CQRS模式
  - 使用数据访问优化技术：批量加载、连接查询
  - 适当使用缓存减少重复访问
  - 特殊场景可考虑只读的跨层访问，但需谨慎管理

- **面试答题要点**：
  - 分析性能问题的根源
  - 提出系统性解决方案，而非临时修补
  - 权衡架构纯洁性与性能需求

### CQRS与传统架构的对比分析
- **传统架构**：
  - 同一模型处理读写操作
  - 简单直观，适合CRUD应用
  - 扩展性和性能优化受限
  - 无法针对读写不平衡进行优化

- **CQRS架构**：
  - 读写模型分离
  - 复杂度高，但扩展性强
  - 可针对读写场景分别优化
  - 支持事件驱动架构

- **选择标准**：
  - 业务复杂度
  - 读写比例
  - 扩展性需求
  - 一致性要求
  - 团队能力

- **面试答题要点**：
  - 分析不同场景下适合的架构选择
  - 说明CQRS带来的优势与挑战
  - 描述渐进式采用CQRS的策略

### 领域模型与持久化框架的协调策略
- **挑战**：
  - ORM框架往往要求无参构造函数和公开setter
  - 领域对象需要封装不变量和业务规则
  - 聚合边界与表关系可能不匹配

- **协调策略**：
  - 使用仓储模式隔离持久化细节
  - 考虑使用AR(Active Record)和DM(Domain Model)混合模式
  - 使用映射层(Mapper)在领域对象和数据库实体间转换
  - 考虑NoSQL或文档数据库，更适合存储聚合

```java
// 领域对象 - 无持久化依赖
public class User {
    private UserId id;
    private String name;
    private Password password;
    
    // 构造函数确保不变量
    public User(UserId id, String name, Password password) {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("用户名不能为空");
        }
        this.id = id;
        this.name = name;
        this.password = password;
    }
    
    // 业务方法
    public void changePassword(Password oldPassword, Password newPassword) {
        if (!this.password.equals(oldPassword)) {
            throw new IllegalArgumentException("原密码不正确");
        }
        this.password = newPassword;
    }
}

// 数据库实体 - 为ORM优化
@Entity
@Table(name = "users")
public class UserEntity {
    @Id
    private String id;
    
    private String name;
    private String passwordHash;
    
    // 无参构造函数和getter/setter
}

// 映射器 - 处理转换
public class UserMapper {
    public User toDomain(UserEntity entity) {
        UserId id = new UserId(entity.getId());
        Password password = Password.fromHash(entity.getPasswordHash());
        return new User(id, entity.getName(), password);
    }
    
    public UserEntity toEntity(User user) {
        UserEntity entity = new UserEntity();
        entity.setId(user.getId().getValue());
        entity.setName(user.getName());
        entity.setPasswordHash(user.getPassword().getHash());
        return entity;
    }
}
```

- **面试答题要点**：
  - 分析ORM与领域模型的匹配挑战
  - 描述防腐层的设计与实现
  - 讨论不同数据库技术对DDD的支持程度 