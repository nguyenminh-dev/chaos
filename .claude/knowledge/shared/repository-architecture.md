# Repository Architecture

**Purpose**: Overall architecture and organization of the WION microservices repository.

**Scope**: This document applies to the **entire repository** structure and organization.

---

## Repository Overview

The WION platform is organized as a **polyglot microservices architecture** with 36+ independent services organized around business domains.

### Repository Structure

```
super_app/
├── src/
│   ├── modules/                    # ABP framework modules
│   │   ├── Volo.*                  # Official ABP modules
│   │   ├── TMT.*                   # Custom framework modules
│   │   └── WiOnPos.*              # WION-specific modules
│   ├── services/                    # Business microservices (36+ services)
│   │   ├── wion-billing/           # Billing service
│   │   ├── customer-management/    # Customer management
│   │   ├── ordering/               # Order processing
│   │   ├── notification/           # Notifications
│   │   └── [32+ other services]
│   └── templates/                  # Service templates
├── .claude/                        # Knowledge base
│   ├── CLAUDE.md                   # AI engineering principles
│   ├── knowledge/                   # Service documentation
│   ├── playbooks/                  # AI workflow patterns
│   ├── templates/                  # Documentation templates
│   └── prompts/                    # AI prompt templates
└── docs/                          # Additional documentation
```

---

## Service Architecture

### Service Categories

#### 1. Core Platform Services
- **account** - Account management
- **authen** - Authentication service
- **tenant-management** - Tenant lifecycle
- **notification-management** - Notification orchestration
- **file-management** - File storage and management

#### 2. Business Domain Services
- **wion-billing** - Financial operations, payments, wallets
- **customer-management** - Customer data management
- **ordering** - Order processing and management
- **product-management** - Product catalog
- **inventory-management** - Stock and inventory

#### 3. Support Services
- **notification** - Notification delivery
- **socket-notification** - Real-time notifications
- **socket-liveapp** - Live application sockets
- **socket-tdeskapp** - TDesk application sockets
- **socket-tpos** - POS system sockets

#### 4. Integration Services
- **integration-management** - Third-party integrations
- **billing-management** - Legacy billing (being replaced by wion-billing)
- **public-management** - Public API management
- **portal** - Web portal service
- **tshop** - E-commerce platform

#### 5. Specialized Services
- **tdesk** - TDesk application
- **tdesk-web** - TDesk web interface
- **tictic** - TicTic application
- **tpos-authentication** - POS authentication
- **liveapp** - Live application platform
- **tcheck** - Check processing service

#### 6. Infrastructure Services
- **admin-bff** - Backend for frontend
- **health-check** - Health monitoring
- **setting-management** - Configuration management
- **version-management** - Version control
- **search-engine** - Search capabilities
- **delivery-otp** - OTP delivery service

---

## Technology Stack

### Backend Technologies
- **.NET 6.0+** - Primary framework for most services
- **ABP Framework** - Application framework for .NET services
- **Node.js** - Select services (if applicable)
- **PostgreSQL** - Primary database
- **Redis** - Caching and message queuing
- **RabbitMQ** - Message broker
- **gRPC** - Inter-service communication

### Frontend Technologies
- **Angular** - Web applications
- **React** - Some web applications
- **Mobile** - Native mobile applications

---

## Common Framework Modules

### TMT Framework Modules
- **TMT.Abp.Domain.Repositories** - Enhanced repository pattern
- **TMT.Abp.Application.Services** - Application services
- **TMT.Abp.MultiTenancy** - Multi-tenancy support
- **TMT.Abp.Kafka** - Kafka event streaming
- **TMT.UniqueKey** - Unique key generation
- **TMT.Share.Contracts** - Shared service contracts

### WION Framework Modules
- **Wion.CodeGenerate** - Code generation utilities
- **TMT.WionFnb.BuildingBlocks.Domain** - Domain building blocks
- **TMT.Wionpos.FeatureManagement** - Feature flags
- **Tmt.Wionpos.RolePermission** - Role-based permissions

---

## Service Communication Patterns

### Synchronous Communication
- **REST APIs** - External service communication
- **gRPC** - High-performance inter-service communication
- **HTTP/HTTPS** - Standard web communication

### Asynchronous Communication
- **RabbitMQ** - Event-driven messaging
- **Kafka** - Event streaming for high-volume scenarios
- **Domain Events** - In-process event handling
- **Distributed Events** - Cross-service events

---

## Data Architecture

### Database Patterns
- **PostgreSQL** - Primary data store
- **Read Replicas** - Query performance optimization
- **Redis** - Caching layer
- **Database per Service** - Each service owns its database

### Data Consistency
- **Strong Consistency** - Within service boundaries
- **Eventual Consistency** - Cross-service operations
- **Saga Pattern** - Distributed transactions
- **CQRS** - Command-query separation

---

## Deployment Architecture

### Service Organization
- **Microservices** - Independent deployment units
- **Containerization** - Docker-based deployment
- **Orchestration** - Kubernetes (if applicable)
- **Load Balancing** - Per-service load balancers

### Scalability Patterns
- **Horizontal Scaling** - Stateless services
- **Database Sharding** - Large dataset partitioning
- **Caching Strategy** - Multi-layer caching
- **CDN** - Static content delivery

---

## Security Architecture

### Authentication & Authorization
- **API Key Authentication** - Service-to-service communication
- **JWT Tokens** - User authentication
- **Role-Based Access Control** - Permission management
- **Tenant Isolation** - Multi-tenancy security

### Data Protection
- **Encryption at Rest** - Database encryption
- **Encryption in Transit** - TLS/SSL
- **PII Protection** - Personal data handling
- **Audit Logging** - Security event tracking

---

## Integration Patterns

### External Service Integration
- **TPayGate** - Payment gateway
- **Invoice Hub** - Electronic invoicing
- **SMS Providers** - SMS notifications
- **Email Providers** - Email notifications

### Integration Strategies
- **API Clients** - HTTP/gRPC clients
- **Webhook Handlers** - Event callbacks
- **Message Consumers** - Event listeners
- **Circuit Breakers** - Fault tolerance

---

## Development Workflow

### Service Creation Workflow
1. **Use Service Template** - Start with standard template
2. **Define Bounded Context** - Establish service boundaries
3. **Implement Domain Layer** - Business logic first
4. **Add Application Layer** - Use cases
5. **Expose APIs** - Service contracts
6. **Implement Infrastructure** - Database, external services
7. **Add Tests** - Comprehensive test coverage
8. **Document** - Knowledge base updates

### Code Organization Principles
- **Domain-Driven Design** - Business-centric organization
- **Clean Architecture** - Layer separation
- **SOLID Principles** - Maintainable code
- **DRY Principle** - Reusable components

---

## Common Patterns

### Repository Pattern
- **Repository Interface** - Domain layer definition
- **EF Core Implementation** - Infrastructure implementation
- **Specification Pattern** - Query encapsulation

### Application Service Pattern
- **Service Interface** - Application.Contracts layer
- **Service Implementation** - Application layer
- **DTO Mapping** - AutoMapper integration

### Event Handling Pattern
- **Domain Events** - Local event handling
- **Distributed Events** - Cross-service events
- **Event Bus** - Event routing

---

## Monitoring & Observability

### Logging Strategy
- **Structured Logging** - JSON format
- **Log Aggregation** - Centralized logging
- **Correlation IDs** - Request tracing
- **Performance Logging** - Operation timing

### Health Monitoring
- **Health Endpoints** - Service health checks
- **Dependency Health** - External service status
- **Metrics Collection** - Performance metrics
- **Alerting** - Proactive monitoring

---

## Documentation Structure

### Knowledge Base Organization
```
.claude/knowledge/
├── shared/                      # Cross-service conventions
│   ├── abp-conventions.md      # ABP framework patterns
│   ├── ddd-conventions.md     # DDD patterns
│   ├── api-conventions.md     # API patterns
│   ├── event-conventions.md   # Event patterns
│   ├── repository-architecture.md # This file
│   └── documentation-standards.md
├── wion-billing/              # Service-specific documentation
│   ├── README.md
│   ├── architecture/
│   ├── domains/
│   ├── application/
│   ├── api/
│   └── events/
└── [other services]/           # Future service documentation
```

---

## Infrastructure Architecture

### Database Patterns

#### Multiple Database Support
- **PostgreSQL** - Primary database for most services
- **MongoDB** - Used for specific services (notification service)
- **In-Memory** - Testing and development

#### Database Configuration Patterns
- **DbContext per service** - Each service owns its database
- **Connection pooling** - Optimized database connections
- **Read replicas** - Query performance optimization
- **Migration system** - EF Core migrations

#### Event Outbox/Inbox Pattern
- **Outbox pattern** - Guaranteed event delivery
- **Inbox pattern** - Idempotent event processing
- **Event tracking** - Distributed event management

---

### Integration Architecture

#### Service Communication
- **REST APIs** - External service communication
- **gRPC** - High-performance inter-service communication
- **Message Bus** - Event-driven communication
- **Kafka** - High-volume event streaming

#### Integration Patterns
- **Service Discovery** - Dynamic service location
- **Circuit Breakers** - Fault tolerance
- **Retries with Exponential Backoff** - Resilience
- **Dead Letter Queues** - Error handling

---

### Security Architecture

#### Authentication & Authorization
- **API Key Authentication** - Service-to-service communication
- **JWT Tokens** - User authentication
- **Role-Based Access Control** - Permission management
- **Tenant Isolation** - Multi-tenancy security

#### Security Implementation
- **Multi-tenant filtering** - Automatic tenant isolation
- **Data filters** - Query-level security
- **Permission-based authorization** - Fine-grained access control
- **Audit logging** - Security event tracking

---

### Performance Patterns

#### Caching Strategies
- **Redis caching** - Distributed caching
- **In-memory caching** - Application-level caching
- **Query result caching** - Database optimization
- **API response caching** - Performance optimization

#### Database Optimization
- **Indexing strategies** - Query performance
- **Connection pooling** - Resource optimization
- **Query optimization** - Performance tuning
- **Database sharding** - Scalability

---

### Configuration Management

#### Configuration Patterns
- **appsettings.json** - Application configuration
- **Environment variables** - Deployment configuration
- **Feature flags** - Runtime feature toggling
- **Tenant-specific settings** - Multi-tenant configuration

---

## Migration & Modernization

### Legacy Services
- **billing-management** - Being replaced by wion-billing
- **Other legacy services** - Planned modernization

### Modern Service Standards
- **ABP Framework** - Standard application framework
- **Clean Architecture** - Layer separation
- **DDD Implementation** - Domain-centric design
- **Comprehensive Testing** - Quality assurance
- **Documentation** - Knowledge base maintenance

---

## Related Documentation
- [ABP Conventions](./abp-conventions.md) - ABP framework patterns
- [DDD Conventions](./ddd-conventions.md) - Domain-driven design
- [API Conventions](./api-conventions.md) - REST API patterns
- [Event Conventions](./event-conventions.md) - Event patterns

---

## Support

**Questions about repository architecture**?
- **Architecture Team**: architecture@wion.vn
- **Tech Lead**: tech-lead@wion.vn

---

**Last Updated**: 2026-07-14  
**Maintained By**: Architecture Team  
**Version**: 1.0