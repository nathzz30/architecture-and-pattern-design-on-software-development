# Hexagonal Architecture - A Complete Guide for Developers

## Table of Contents
1. [Introduction](#introduction)
2. [What is Hexagonal Architecture?](#what-is-hexagonal-architecture)
3. [Core Concepts](#core-concepts)
4. [The Architecture Layers](#the-architecture-layers)
5. [Ports and Adapters Explained](#ports-and-adapters-explained)
6. [Generic Examples](#generic-examples)
7. [Benefits and Trade-offs](#benefits-and-trade-offs)
8. [Testing Strategy](#testing-strategy)
9. [Common Patterns](#common-patterns)
10. [Best Practices](#best-practices)
11. [When to Use This Architecture](#when-to-use-this-architecture)

---

## Introduction

Hexagonal Architecture, also known as **Ports and Adapters Architecture**, was introduced by Alistair Cockburn in 2005. It's a software design pattern that aims to create loosely coupled application components that can be easily connected to their software environment through ports and adapters.

### Key Goals:
- Isolate business logic from external concerns
- Make the application technology-agnostic
- Improve testability
- Enable easy replacement of external dependencies

---

## What is Hexagonal Architecture?

### The Hexagon Metaphor

The "hexagon" represents your application's core business logic. Each side of the hexagon can have multiple ports, and each port can have multiple adapters.

```
                 External World
                      ↓
           ┌──────────────────────┐
           │      Adapters        │
           │  (Implementations)   │
           └──────────┬───────────┘
                      │
           ┌──────────┴───────────┐
           │       Ports          │
           │    (Interfaces)      │
           └──────────┬───────────┘
                      │
           ┌──────────┴───────────┐
           │   Business Logic     │
           │     (Core/Domain)    │
           └──────────────────────┘
```

### Why "Hexagonal"?

The hexagon shape is arbitrary - it could be any polygon. The point is that the application has **multiple sides**, each representing a way to interact with the outside world.

---

## Core Concepts

### 1. The Application Core (Hexagon Center)

This is your **business logic** - the heart of your application. It:
- Contains domain models and business rules
- Has NO dependencies on external frameworks or libraries
- Defines interfaces (ports) for what it needs
- Remains pure and framework-agnostic

### 2. Ports (Interfaces)

Ports are **contracts** that define how the outside world can interact with your application, or how your application can interact with the outside world.

**Two types:**
- **Primary/Driving Ports**: How external actors use your application (e.g., API endpoints)
- **Secondary/Driven Ports**: How your application uses external services (e.g., database, email)

### 3. Adapters (Implementations)

Adapters are **concrete implementations** that connect external systems to your ports. They translate between the external world's language and your application's language.

**Two types:**
- **Primary/Driving Adapters**: Receive input and drive the application (e.g., REST controller, CLI)
- **Secondary/Driven Adapters**: Execute actions requested by the application (e.g., database repository, email sender)

---

## The Architecture Layers

### Visual Representation

```
┌─────────────────────────────────────────────────────────┐
│                    EXTERNAL WORLD                        │
│  (HTTP, GraphQL, CLI, Message Queue, Scheduled Jobs)   │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │ REST    │            │  CLI    │
    │Adapter  │            │Adapter  │
    └────┬────┘            └────┬────┘
         │                       │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   PRIMARY PORTS       │
         │   (Input Interfaces)  │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   APPLICATION CORE    │
         │   (Business Logic)    │
         │   - Use Cases         │
         │   - Domain Models     │
         │   - Business Rules    │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │  SECONDARY PORTS      │
         │  (Output Interfaces)  │
         └───────────┬───────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │Database │            │ Email   │
    │Adapter  │            │Adapter  │
    └────┬────┘            └────┬────┘
         │                       │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   EXTERNAL SERVICES   │
         │  (Database, APIs,     │
         │   Email, Storage)     │
         └───────────────────────┘
```

---

## Ports and Adapters Explained

### Ports (Interfaces)

Think of ports like **USB ports on a computer**. They define:
- What shape the plug must have
- What signals are expected
- What protocol to use

But they don't care about:
- What specific device is plugged in
- How the device is manufactured
- What brand it is

```typescript
// Example: Port (Interface)
interface PaymentPort {
  processPayment(amount: number, currency: string): Promise<PaymentResult>
  refundPayment(transactionId: string): Promise<RefundResult>
}
```

### Adapters (Implementations)

Adapters are like **specific devices** that plug into those ports:
- A mouse adapter
- A keyboard adapter
- A USB drive adapter

Each adapter:
- Implements the port's interface
- Translates between external system and application
- Contains technology-specific code

```typescript
// Example: Adapter (Implementation)
class StripePaymentAdapter implements PaymentPort {
  async processPayment(amount: number, currency: string): Promise<PaymentResult> {
    // Stripe-specific implementation
    const charge = await stripe.charges.create({
      amount: amount * 100,
      currency: currency,
    })
    return { success: true, transactionId: charge.id }
  }
  
  async refundPayment(transactionId: string): Promise<RefundResult> {
    // Stripe-specific refund logic
    const refund = await stripe.refunds.create({ charge: transactionId })
    return { success: true, refundId: refund.id }
  }
}
```

---

## Generic Examples

### Example 1: E-commerce Order Processing

#### Domain Model (Core)

```typescript
// Domain entity
class Order {
  constructor(
    public readonly id: string,
    public readonly items: OrderItem[],
    public status: OrderStatus,
    public totalAmount: number
  ) {}
  
  // Business logic
  canBeCancelled(): boolean {
    return this.status === 'PENDING' || this.status === 'PROCESSING'
  }
  
  calculateTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
  }
}
```

#### Secondary Port (Output Interface)

```typescript
// Repository port - what the application needs
interface OrderRepositoryPort {
  save(order: Order): Promise<void>
  findById(orderId: string): Promise<Order | null>
  findByCustomerId(customerId: string): Promise<Order[]>
}
```

#### Secondary Adapter (Database Implementation)

```typescript
// SQL Database Adapter
class SqlOrderRepository implements OrderRepositoryPort {
  constructor(private database: SqlDatabase) {}
  
  async save(order: Order): Promise<void> {
    await this.database.query(
      'INSERT INTO orders (id, items, status, total) VALUES (?, ?, ?, ?)',
      [order.id, JSON.stringify(order.items), order.status, order.totalAmount]
    )
  }
  
  async findById(orderId: string): Promise<Order | null> {
    const row = await this.database.query('SELECT * FROM orders WHERE id = ?', [orderId])
    return row ? this.mapToOrder(row) : null
  }
  
  async findByCustomerId(customerId: string): Promise<Order[]> {
    const rows = await this.database.query('SELECT * FROM orders WHERE customer_id = ?', [customerId])
    return rows.map(row => this.mapToOrder(row))
  }
  
  private mapToOrder(row: any): Order {
    return new Order(row.id, JSON.parse(row.items), row.status, row.total)
  }
}

// MongoDB Adapter - same port, different implementation
class MongoOrderRepository implements OrderRepositoryPort {
  constructor(private collection: MongoCollection) {}
  
  async save(order: Order): Promise<void> {
    await this.collection.insertOne({
      _id: order.id,
      items: order.items,
      status: order.status,
      totalAmount: order.totalAmount,
    })
  }
  
  async findById(orderId: string): Promise<Order | null> {
    const doc = await this.collection.findOne({ _id: orderId })
    return doc ? new Order(doc._id, doc.items, doc.status, doc.totalAmount) : null
  }
  
  async findByCustomerId(customerId: string): Promise<Order[]> {
    const docs = await this.collection.find({ customerId }).toArray()
    return docs.map(doc => new Order(doc._id, doc.items, doc.status, doc.totalAmount))
  }
}
```

#### Primary Port (Input Interface)

```typescript
// Use case interface
interface PlaceOrderUseCase {
  execute(request: PlaceOrderRequest): Promise<PlaceOrderResponse>
}
```

#### Application Service (Use Case)

```typescript
// Business logic orchestration
class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    private orderRepository: OrderRepositoryPort,
    private paymentService: PaymentPort,
    private notificationService: NotificationPort
  ) {}
  
  async execute(request: PlaceOrderRequest): Promise<PlaceOrderResponse> {
    // 1. Create domain entity
    const order = new Order(
      generateId(),
      request.items,
      'PENDING',
      0
    )
    
    // 2. Apply business rules
    order.totalAmount = order.calculateTotal()
    
    // 3. Save order
    await this.orderRepository.save(order)
    
    // 4. Process payment
    const paymentResult = await this.paymentService.processPayment(
      order.totalAmount,
      request.currency
    )
    
    if (!paymentResult.success) {
      order.status = 'FAILED'
      await this.orderRepository.save(order)
      throw new PaymentFailedError()
    }
    
    // 5. Update order status
    order.status = 'CONFIRMED'
    await this.orderRepository.save(order)
    
    // 6. Send notification
    await this.notificationService.sendOrderConfirmation(order.id, request.customerEmail)
    
    return { orderId: order.id, status: order.status }
  }
}
```

#### Primary Adapter (HTTP Controller)

```typescript
// REST API Adapter
class OrderController {
  constructor(private placeOrderUseCase: PlaceOrderUseCase) {}
  
  async handlePlaceOrder(httpRequest: HttpRequest): Promise<HttpResponse> {
    try {
      // Convert HTTP request to use case request
      const request: PlaceOrderRequest = {
        items: httpRequest.body.items,
        currency: httpRequest.body.currency,
        customerEmail: httpRequest.body.email,
      }
      
      // Execute use case
      const response = await this.placeOrderUseCase.execute(request)
      
      // Convert use case response to HTTP response
      return {
        statusCode: 201,
        body: {
          orderId: response.orderId,
          status: response.status,
        },
      }
    } catch (error) {
      return {
        statusCode: 400,
        body: { error: error.message },
      }
    }
  }
}
```

### Example 2: Notification System

#### Secondary Port (Output)

```typescript
// Notification port - what the application needs
interface NotificationPort {
  send(recipient: string, message: string, channel: NotificationChannel): Promise<void>
}

type NotificationChannel = 'EMAIL' | 'SMS' | 'PUSH'
```

#### Multiple Adapters for Same Port

```typescript
// Email Adapter
class EmailNotificationAdapter implements NotificationPort {
  constructor(private emailClient: EmailClient) {}
  
  async send(recipient: string, message: string, channel: NotificationChannel): Promise<void> {
    if (channel !== 'EMAIL') {
      throw new Error('This adapter only supports EMAIL')
    }
    
    await this.emailClient.sendEmail({
      to: recipient,
      subject: 'Notification',
      body: message,
    })
  }
}

// SMS Adapter
class SmsNotificationAdapter implements NotificationPort {
  constructor(private smsClient: SmsClient) {}
  
  async send(recipient: string, message: string, channel: NotificationChannel): Promise<void> {
    if (channel !== 'SMS') {
      throw new Error('This adapter only supports SMS')
    }
    
    await this.smsClient.sendSms({
      phoneNumber: recipient,
      text: message,
    })
  }
}

// Composite Adapter (routes to appropriate adapter)
class CompositeNotificationAdapter implements NotificationPort {
  constructor(
    private emailAdapter: EmailNotificationAdapter,
    private smsAdapter: SmsNotificationAdapter,
    private pushAdapter: PushNotificationAdapter
  ) {}
  
  async send(recipient: string, message: string, channel: NotificationChannel): Promise<void> {
    switch (channel) {
      case 'EMAIL':
        return this.emailAdapter.send(recipient, message, channel)
      case 'SMS':
        return this.smsAdapter.send(recipient, message, channel)
      case 'PUSH':
        return this.pushAdapter.send(recipient, message, channel)
      default:
        throw new Error(`Unsupported channel: ${channel}`)
    }
  }
}
```

---

## Benefits and Trade-offs

### Benefits ✅

#### 1. Technology Independence
Your business logic doesn't depend on specific frameworks or libraries.

```typescript
// Business logic doesn't know about Express, Fastify, or any HTTP framework
class TransferMoneyService {
  constructor(private accountRepo: AccountRepositoryPort) {}
  
  async execute(fromId: string, toId: string, amount: number): Promise<void> {
    const fromAccount = await this.accountRepo.findById(fromId)
    const toAccount = await this.accountRepo.findById(toId)
    
    if (fromAccount.balance < amount) {
      throw new InsufficientFundsError()
    }
    
    fromAccount.withdraw(amount)
    toAccount.deposit(amount)
    
    await this.accountRepo.save(fromAccount)
    await this.accountRepo.save(toAccount)
  }
}
```

#### 2. Testability
Easy to test business logic in isolation.

```typescript
// Test without real database
describe('TransferMoneyService', () => {
  it('should transfer money between accounts', async () => {
    // Mock repository
    const mockRepo: AccountRepositoryPort = {
      findById: jest.fn()
        .mockResolvedValueOnce(new Account('1', 1000))
        .mockResolvedValueOnce(new Account('2', 500)),
      save: jest.fn().mockResolvedValue(undefined),
    }
    
    const service = new TransferMoneyService(mockRepo)
    await service.execute('1', '2', 100)
    
    expect(mockRepo.save).toHaveBeenCalledTimes(2)
  })
})
```

#### 3. Maintainability
Clear separation of concerns makes code easier to understand and modify.

#### 4. Flexibility
Easy to swap implementations without changing business logic.

```typescript
// Development: Use in-memory database
const devRepo = new InMemoryAccountRepository()

// Production: Use PostgreSQL
const prodRepo = new PostgresAccountRepository(dbConnection)

// Both implement the same port, business logic doesn't change
const service = new TransferMoneyService(prodRepo)
```

#### 5. Parallel Development
Teams can work on different adapters simultaneously.

### Trade-offs ❌

#### 1. Initial Complexity
More interfaces and abstractions to manage.

#### 2. Learning Curve
Team needs to understand the architecture.

#### 3. Boilerplate Code
More files and classes compared to simpler architectures.

#### 4. Over-engineering Risk
May be overkill for simple CRUD applications.

---

## Testing Strategy

### 1. Unit Tests (Core Business Logic)

```typescript
describe('Order', () => {
  it('should calculate total correctly', () => {
    const order = new Order('1', [
      { productId: 'A', price: 10, quantity: 2 },
      { productId: 'B', price: 20, quantity: 1 },
    ], 'PENDING', 0)
    
    expect(order.calculateTotal()).toBe(40)
  })
  
  it('should not allow cancellation when shipped', () => {
    const order = new Order('1', [], 'SHIPPED', 100)
    expect(order.canBeCancelled()).toBe(false)
  })
})
```

### 2. Integration Tests (Use Cases with Mocked Ports)

```typescript
describe('PlaceOrderService', () => {
  it('should process order successfully', async () => {
    const mockOrderRepo: OrderRepositoryPort = {
      save: jest.fn().mockResolvedValue(undefined),
      findById: jest.fn(),
      findByCustomerId: jest.fn(),
    }
    
    const mockPayment: PaymentPort = {
      processPayment: jest.fn().mockResolvedValue({ success: true, transactionId: 'txn1' }),
      refundPayment: jest.fn(),
    }
    
    const mockNotification: NotificationPort = {
      send: jest.fn().mockResolvedValue(undefined),
    }
    
    const service = new PlaceOrderService(mockOrderRepo, mockPayment, mockNotification)
    
    const result = await service.execute({
      items: [{ productId: 'A', price: 10, quantity: 1 }],
      currency: 'USD',
      customerEmail: 'test@example.com',
    })
    
    expect(result.status).toBe('CONFIRMED')
    expect(mockOrderRepo.save).toHaveBeenCalled()
    expect(mockPayment.processPayment).toHaveBeenCalledWith(10, 'USD')
    expect(mockNotification.send).toHaveBeenCalled()
  })
})
```

### 3. Adapter Tests

```typescript
describe('SqlOrderRepository', () => {
  it('should save order to database', async () => {
    const mockDatabase = createTestDatabase()
    const repository = new SqlOrderRepository(mockDatabase)
    
    const order = new Order('1', [], 'PENDING', 100)
    await repository.save(order)
    
    const saved = await repository.findById('1')
    expect(saved).toEqual(order)
  })
})
```

### 4. End-to-End Tests

```typescript
describe('Order API E2E', () => {
  it('should place order via REST API', async () => {
    const response = await fetch('http://localhost:3000/orders', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        items: [{ productId: 'A', price: 10, quantity: 1 }],
        currency: 'USD',
        email: 'test@example.com',
      }),
    })
    
    expect(response.status).toBe(201)
    const data = await response.json()
    expect(data.orderId).toBeDefined()
  })
})
```

---

## Common Patterns

### 1. Dependency Injection

```typescript
// Manual DI
class OrderService {
  constructor(
    private orderRepo: OrderRepositoryPort,
    private paymentService: PaymentPort
  ) {}
}

// Usage
const orderRepo = new SqlOrderRepository(database)
const paymentService = new StripePaymentAdapter(stripeClient)
const orderService = new OrderService(orderRepo, paymentService)
```

### 2. Factory Pattern for Adapters

```typescript
class RepositoryFactory {
  static createOrderRepository(config: Config): OrderRepositoryPort {
    switch (config.database) {
      case 'postgres':
        return new PostgresOrderRepository(config.connectionString)
      case 'mongodb':
        return new MongoOrderRepository(config.connectionString)
      case 'in-memory':
        return new InMemoryOrderRepository()
      default:
        throw new Error(`Unsupported database: ${config.database}`)
    }
  }
}
```

### 3. Strategy Pattern

```typescript
interface PricingStrategy {
  calculatePrice(basePrice: number): number
}

class RegularPricing implements PricingStrategy {
  calculatePrice(basePrice: number): number {
    return basePrice
  }
}

class DiscountPricing implements PricingStrategy {
  constructor(private discountPercent: number) {}
  
  calculatePrice(basePrice: number): number {
    return basePrice * (1 - this.discountPercent / 100)
  }
}

class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}
  
  calculate(basePrice: number): number {
    return this.strategy.calculatePrice(basePrice)
  }
}
```

### 4. Repository Pattern

```typescript
interface Repository<T> {
  save(entity: T): Promise<void>
  findById(id: string): Promise<T | null>
  findAll(): Promise<T[]>
  delete(id: string): Promise<void>
}

abstract class BaseRepository<T> implements Repository<T> {
  abstract save(entity: T): Promise<void>
  abstract findById(id: string): Promise<T | null>
  abstract findAll(): Promise<T[]>
  abstract delete(id: string): Promise<void>
}
```

---

## Best Practices

### 1. Keep Business Logic Pure

```typescript
// ✅ Good: Pure business logic
class ShoppingCart {
  private items: CartItem[] = []
  
  addItem(product: Product, quantity: number): void {
    if (quantity <= 0) {
      throw new InvalidQuantityError()
    }
    
    this.items.push({ product, quantity })
  }
  
  calculateTotal(): number {
    return this.items.reduce((sum, item) => 
      sum + item.product.price * item.quantity, 0
    )
  }
}

// ❌ Bad: Business logic coupled to database
class ShoppingCart {
  async addItem(productId: string, quantity: number): Promise<void> {
    const product = await database.query('SELECT * FROM products WHERE id = ?', [productId])
    await database.query('INSERT INTO cart_items VALUES (?, ?)', [productId, quantity])
  }
}
```

### 2. Define Clear Port Interfaces

```typescript
// ✅ Good: Interface segregation
interface ProductReader {
  findById(id: string): Promise<Product | null>
  findByCategory(category: string): Promise<Product[]>
}

interface ProductWriter {
  save(product: Product): Promise<void>
  delete(id: string): Promise<void>
}

// ❌ Bad: Too many responsibilities in one interface
interface ProductRepository {
  findById(id: string): Promise<Product | null>
  findByCategory(category: string): Promise<Product[]>
  save(product: Product): Promise<void>
  delete(id: string): Promise<void>
  sendEmailNotification(productId: string): Promise<void>  // ❌ Wrong responsibility
  generatePdf(productId: string): Promise<Buffer>  // ❌ Wrong responsibility
}
```

### 3. Use Value Objects

```typescript
// ✅ Good: Value object for validation
class Email {
  private readonly value: string
  
  constructor(email: string) {
    if (!this.isValid(email)) {
      throw new InvalidEmailError()
    }
    this.value = email
  }
  
  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
  }
  
  toString(): string {
    return this.value
  }
}

// Usage
const email = new Email('user@example.com')  // Validates on construction

// ❌ Bad: Primitive obsession
function sendEmail(email: string): void {
  // Email validation scattered everywhere
  if (!email.includes('@')) {
    throw new Error('Invalid email')
  }
  // ...
}
```

### 4. Keep Adapters Thin

```typescript
// ✅ Good: Adapter only translates
class HttpOrderController {
  constructor(private placeOrderUseCase: PlaceOrderUseCase) {}
  
  async handleRequest(req: HttpRequest): Promise<HttpResponse> {
    // Just translate HTTP → Domain and Domain → HTTP
    const useCaseRequest = this.toUseCaseRequest(req.body)
    const useCaseResponse = await this.placeOrderUseCase.execute(useCaseRequest)
    return this.toHttpResponse(useCaseResponse)
  }
}

// ❌ Bad: Business logic in adapter
class HttpOrderController {
  async handleRequest(req: HttpRequest): Promise<HttpResponse> {
    // ❌ Business logic in controller
    if (req.body.items.length === 0) {
      return { status: 400, body: 'Cart is empty' }
    }
    
    const total = req.body.items.reduce((sum, item) => sum + item.price, 0)
    
    if (total > 10000) {
      return { status: 400, body: 'Order too large' }
    }
    // ...
  }
}
```

### 5. Use DTOs for Data Transfer

```typescript
// ✅ Good: Separate DTOs from domain
interface CreateOrderRequest {
  customerId: string
  items: { productId: string; quantity: number }[]
}

interface CreateOrderResponse {
  orderId: string
  status: string
  total: number
}

// ❌ Bad: Exposing domain entities directly
async function createOrder(order: Order): Promise<Order> {
  // Exposing internal structure
}
```

---

## When to Use This Architecture

### ✅ Use Hexagonal Architecture When:

1. **Long-term maintenance is important**
   - Project will be maintained for years
   - Multiple developers will work on it

2. **Business logic is complex**
   - Rich domain models
   - Complex business rules
   - Need for domain experts

3. **Technology flexibility is needed**
   - May need to switch databases
   - May add multiple interfaces (REST, GraphQL, gRPC)
   - External services may change

4. **High testability is required**
   - Need comprehensive unit tests
   - TDD/BDD approach
   - Continuous testing

5. **Multiple teams working on different parts**
   - Frontend team, backend team, infrastructure team
   - Need clear boundaries

### ❌ Consider Simpler Alternatives When:

1. **Simple CRUD application**
   - Basic create, read, update, delete operations
   - No complex business logic

2. **Prototype or MVP**
   - Need to validate idea quickly
   - May be thrown away

3. **Small team with simple requirements**
   - Overhead not justified
   - Everyone understands the entire codebase

4. **Tight deadline with limited scope**
   - Speed more important than architecture
   - Project has clear end date

---

## Summary

**Hexagonal Architecture is about:**

### Core Principles:
- **Ports** = Interfaces that define contracts
- **Adapters** = Implementations that fulfill contracts
- **Core** = Business logic independent of external concerns

### Key Benefits:
- 🧪 **Highly Testable** - Mock ports for unit tests
- 🔄 **Flexible** - Swap implementations easily
- 📦 **Maintainable** - Clear separation of concerns
- 🚀 **Scalable** - Easy to extend with new adapters
- 🎯 **Focused** - Business logic stays pure

### Remember:
> "Make it work, make it right, make it fast" - Kent Beck

Start with working code, refactor toward hexagonal architecture when complexity justifies it, and optimize performance where needed.

---

## Further Reading

### Books:
- **"Implementing Domain-Driven Design"** by Vaughn Vernon
- **"Clean Architecture"** by Robert C. Martin
- **"Patterns of Enterprise Application Architecture"** by Martin Fowler

### Articles:
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) by Alistair Cockburn
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) by Robert C. Martin

### Videos:
- "Hexagonal Architecture" talks at various conferences
- Domain-Driven Design courses

---

**Document Version:** 1.0  
**Last Updated:** December 2024

---

## Questions?

Remember: Architecture is a tool to solve problems. Use it when it makes sense, not because it's trendy. The best architecture is the one that helps your team deliver value to users while keeping the codebase maintainable.
