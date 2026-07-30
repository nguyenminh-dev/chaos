# Wion Platform MultiTenancy - Documentation

## 🎯 Overview

The Wion Platform MultiTenancy modules provide a **centralized tenant management pattern** where the Authentication Service owns tenant data and other services resolve tenant information via HTTP/gRPC calls.

## 📚 Documentation Structure

### Quick Start
- **[Quick Start Guide](quick-start.md)** - Get started in 5 minutes
- **[Configuration Examples](configuration.md)** - Common configuration patterns

### Architecture
- **[Architecture Guide](architecture.md)** - System architecture and data flow
- **[Module Structure](modules.md)** - Component breakdown and responsibilities

### Implementation
- **[Implementation Plan](implementation-plan.md)** - Development phases and timeline
- **[API Reference](api-reference.md)** - Interfaces and classes reference

### Operations
- **[Troubleshooting Guide](troubleshooting.md)** - Common issues and solutions
- **[Performance Guide](performance.md)** - Caching and optimization

## 🚀 Quick Links

### For New Users
1. Read [Quick Start Guide](quick-start.md)
2. Configure your service using [Configuration Examples](configuration.md)

### For Developers
1. Understand [Architecture Guide](architecture.md)
2. Review [Implementation Plan](implementation-plan.md)
3. Check [API Reference](api-reference.md)

### For DevOps
1. Setup using [Configuration Examples](configuration.md)
2. Monitor using [Performance Guide](performance.md)
3. Debug using [Troubleshooting Guide](troubleshooting.md)

## 🏗️ Module Overview

### Wion.Platform.MultiTenancy
HTTP-based tenant resolution for services without tenant databases.

**Key Features:**
- ✅ Remote tenant resolution via HTTP
- ✅ Distributed caching (Redis)
- ✅ Multiple provider support (database, HTTP, gRPC)
- ✅ No tenant database required

**Used By:** Product Service, Order Service, Payment Service, etc.

### Wion.Platform.AspNetCore.MultiTenancy
ASP.NET Core middleware for advanced tenant resolution.

**Key Features:**
- ✅ Multi-strategy resolution (JWT, Header, Cookie, QueryString)
- ✅ Configurable priority
- ✅ Enhanced error handling
- ✅ Detailed logging

**Used By:** All services using Wion.Platform.MultiTenancy

## 🔧 Key Concepts

### Centralized Tenant Management
```
Authentication Service (Tenant Owner)
    ↓ HTTP/gRPC
Other Services (Tenant Consumers)
```

### Service Classification
- **Authentication Service** → Has tenant database, uses Database Provider
- **Other Services** → No tenant database, uses HTTP Provider

### Tenant Resolution Flow
```
HTTP Request → Middleware → Tenant Resolution
    → Cache Check → Provider Call → Authentication Service
    → Cache Update → Business Logic with Tenant Context
```

## 📋 Prerequisites

- ✅ Authentication Service with tenant database
- ✅ Redis cache for distributed caching
- ✅ HTTP/gRPC endpoints exposed by Authentication Service
- ✅ .NET 6.0+ and ABP Framework 5.2.2+

## 🔗 Related Resources

- [ABP Multi-Tenancy Documentation](https://docs.abp.io/en/abp/latest/Multi-Tenancy)
- [Microservices Architecture Patterns](https://microservices.io/patterns/microservices.html)
- [TPos Development Guidelines](../../../.claude/CLAUDE.md)

## 📝 Changelog

### v1.0.0 (Current)
- Initial release of Wion.Platform.MultiTenancy
- HTTP-based tenant resolution
- Distributed caching support
- Multiple provider implementations
- ASP.NET Core middleware integration

---

**Last Updated:** 2026-06-28
**Version:** 1.0.0
**Maintained By:** TPos Development Team
