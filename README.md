# WION Billing Platform

**Overview**: The WION Billing Platform is a unified financial platform serving the entire WION ecosystem, handling payments, credit consumption, wallet operations, and invoice generation.

## 📚 Knowledge Base

**Complete documentation**: **[`.claude/knowledge/`](.claude/knowledge/)**

This repository uses a **multi-service Knowledge Base architecture** that supports all microservices in the WION ecosystem.

---

## 🚀 Quick Start

### For AI Agents (Claude Code)
The Knowledge Base is optimized for AI consumption:
- ✅ **Service-Agnostic**: Automatically discovers target services
- ✅ **Single Source of Truth**: Each concept exists in exactly one place
- ✅ **DDD-Compliant**: Domain-Driven Design principles
- ✅ **AI Workflows**: Comprehensive playbooks for common tasks

**Start here**: **[`.claude/CLAUDE.md`](.claude/CLAUDE.md)** - Engineering principles and service discovery

---

## 🏗️ Platform Scope

Billing Platform serves these WION products:
- **WION POS** - Point of Sale
- **WION FnB** - Food & Beverage
- **WION SPA** - Service Platform
- **WIPIX** - Media Platform
- **AI Services** - AI Generation, OCR
- **Platform Services** - SMS, Storage, Marketplace

---

## 💡 Key Capabilities

- 💳 **Payments**: QR code topup via TPayGate
- 💰 **Wallets**: Digital wallet and balance management
- ⚡ **Credits**: Real-time credit consumption (10,000 TPS)
- 📄 **Invoices**: Electronic invoice generation
- 📊 **Ledger**: Double-entry accounting audit trail

---

## 🎯 Key Principles

1. **Balance Never Negative**: BR-001 enforced at database level
2. **Double-Entry Accounting**: Every transaction creates ledger entries
3. **Idempotency**: All mutations support idempotency keys
4. **Invoice on Success**: Invoices only created for successful payments

---

## 📖 Knowledge Base Structure

The Knowledge Base follows a **multi-service architecture**:

```
.claude/
├── CLAUDE.md                      # Engineering principles + service discovery
├── playbooks/                     # Global AI workflows (service-agnostic)
├── templates/                     # Global documentation templates
├── prompts/                       # AI prompt templates
└── knowledge/                     # Service documentation
    ├── shared/                    # Shared conventions & standards
    │   ├── ddd-conventions.md
    │   ├── event-conventions.md
    │   ├── api-conventions.md
    │   └── documentation-standards.md
    ├── billing-service/           # Billing Service documentation
    │   ├── README.md
    │   ├── architecture/
    │   ├── domains/
    │   ├── application/
    │   ├── infrastructure/
    │   ├── api/
    │   ├── policies/
    │   └── reference/
    └── [other services]/          # Future services
```

---

## 🤖 For AI Agents

### Service Discovery
AI agents **automatically discover** which service(s) are affected by a task:

1. **Detect** service from task description, file paths, or context
2. **Locate** corresponding Knowledge Base in `.claude/knowledge/{service}/`
3. **Load** service-specific documentation
4. **Apply** shared conventions from `.claude/knowledge/shared/`

### Multi-Service Tasks
When a task affects multiple services:
- ✅ Identify **every affected service**
- ✅ Load **every affected Knowledge Base**
- ✅ Perform **cross-service impact analysis**
- ✅ Update **every affected Knowledge Base**

---

## 📞 Support

- **Platform Team**: platform@wion.vn
- **Tech Lead**: tech-lead@wion.vn
- **Product Owner**: po@wion.vn

---

## 📄 License

Copyright © 2026 WION Platform Team. All rights reserved.

---

*Last Updated: 2026-07-05*
